<!-- eslint-disable vue/no-mutating-props -->
<script setup lang="ts">
import { computed, inject, nextTick, onMounted, onUnmounted, onUpdated, provide, ref, toRaw, watch } from 'vue';
import type { GlobalProps, Item, LinkCode, LinkZip } from '@/common';
import { timeOrDefault, Action, inUse, exposeItem } from '@/common';
import LinkGroup from './LinkGroup.vue';

const props = defineProps<{ item: Item; }>();
const gProps = inject<GlobalProps>('props')!;

provide('srcItem', props.item);

const isFolder = computed(() => children.value.length > 0);
const isOpen = ref(props.item.root ?? false);
const linkZips = computed(() => zipLinks(getLinks()));
const headLink = ref<HTMLElement>();

const children = computed(() => props.item.children?.filter(item => {
  if (item.code < 100000) return true;
  const res = gProps.searchResult.value;
  return res == undefined || res.items.has(toRaw(item));
}) ?? []);

// 地级市（代码第 3-4 位为 01-20）下的市辖区：数据集不收录 xx01“市辖区”统计代码，
// 这里在前端虚拟出一个分组，仅用于展示。
function isDistrict(it: Item): boolean {
  if (it.code < 100000) return false;
  const suffix = it.code % 100;
  if (!it.name.endsWith('区') || /(特区|林区)$/.test(it.name)) return false;
  return (suffix >= 1 && suffix <= 20) || (suffix >= 51 && suffix <= 80);
}

const isPrefecture = computed(() => {
  const code = props.item.code;
  if (code < 100000 || code % 100 != 0 || code % 10000 == 0) return false;
  const mid = Math.trunc(code / 100) % 100;
  return mid >= 1 && mid <= 20;
});

const districts = computed(() =>
  isPrefecture.value ? children.value.filter(isDistrict) : []);
const others = computed(() =>
  districts.value.length > 0 ? children.value.filter(it => !isDistrict(it)) : children.value);

const groupOpen = ref(true);
const group = computed(() => {
  const list = districts.value;
  if (list.length == 0) return undefined;
  const code = props.item.code + 1;
  const start = Math.min(...list.map(it => it.start));
  const end = list.some(it => it.end == undefined) ? undefined : Math.max(...list.map(it => it.end!));
  const taken = gProps.items.get(code)?.some(it => !isDistrict(it) || it.parent == toRaw(props.item));
  return { code: taken ? undefined : code, start, end };
});

const isHit = computed(() => {
  const res = gProps.searchResult.value;
  return res != undefined && res.time == undefined &&
    res.hits?.has(toRaw(props.item));
});

if (props.item.guide) {
  watch(isOpen, v => v ?
    localStorage.removeItem("closeGuide") :
    localStorage.setItem("closeGuide", "true")
  );
}

onMounted(() => {
  props.item.act = act;
  act();
});
onUpdated(() => {
  props.item.act = act;
  act();
});
onUnmounted(() => props.item.act = undefined);

function act() {
  const action = props.item.action;
  if (action != undefined) {
    if (action & Action.Open) {
      isOpen.value = isFolder.value;
    } else if (action & Action.Close) {
      isOpen.value = false;
    }

    const scroll = (action & Action.Scroll) != 0;
    if (action & Action.Focus)
      headLink.value?.focus({ preventScroll: scroll });
    if (scroll) document.fonts.ready.then(() => {
      headLink.value?.scrollIntoView();
    });
    props.item.action = undefined;
  }
}

interface LinkWithDirection {
  code: number,
  time: number,
  desc?: number,
  rev: boolean,
}

function getLinks(): LinkWithDirection[] {
  const item = props.item;
  const predecessors = gProps.predecessors.get(item.code);
  let links: LinkWithDirection[];
  if (!gProps.options.hidePredecessors && predecessors != undefined) {
    links = predecessors.filter(link => inUse(item, link.time!))
      .map(link => {
        return { code: link.code, time: link.time!, desc: link.desc, rev: true };
      });
  } else {
    links = [];
  }

  if (!gProps.options.hideSuccessors) {
    item.succ?.forEach(link => {
      links.push({
        code: link.code,
        time: timeOrDefault(link, item),
        desc: link.desc,
        rev: false,
      });
    });
  }

  links.sort((a, b) => {
    const diff = a.time - b.time;
    return diff != 0 ? diff : ((b.rev ? 1 : 0) - (a.rev ? 1 : 0));
  });
  return links;
}

function zipLinks(links: LinkWithDirection[]): LinkZip[] {
  if (links.length == 0) {
    return [];
  }
  const out: LinkZip[] = [];
  let prev = links[0]!;
  let codes: LinkCode[] = [{ code: prev.code, desc: prev.desc }];

  links.slice(1).forEach(link => {
    if (link.time == prev.time && link.rev == prev.rev) {
      if (link.desc != prev.desc && prev.desc != undefined) {
        codes[codes.length - 1]!.showDesc = true;
      }
      codes.push({ code: link.code, desc: link.desc });
    } else {
      if (prev.desc != undefined) {
        codes[codes.length - 1]!.showDesc = true;
      }
      out.push({ codes, time: prev.time, rev: prev.rev });
      codes = [{ code: link.code, desc: link.desc }];
    }
    prev = link;
  });
  if (prev.desc != undefined) {
    codes[codes.length - 1]!.showDesc = true;
  }
  out.push({ codes, time: prev.time, rev: prev.rev });
  return out;
}

function onKeyDown(e: KeyboardEvent) {
  if (e.code == "Backspace") {
    const parent = props.item.parent;
    if (parent != undefined) {
      exposeItem(parent, Action.Focus);
    }
  } else if (e.code == "Minus") {
    isOpen.value = false;
  } else if (e.code == "Equal") {
    isOpen.value = true;
  }
}

function onClick() {
  const item = props.item;
  gProps.pushHistory(item);
  gProps.options.searchText =
    `${item.code},${item.start}-${item.end ?? ''}`;
  nextTick(() => exposeItem(item, Action.Scroll));
}
</script>

<template>
  <li>
    <div v-if="!item.root" class="row" :class="{ obsolete: item.end, leaf: !isFolder, hit: isHit, guide: item.guide }">
      <span v-if="isFolder" class="toggle" :class="{ open: isOpen }" @click="isOpen = !isOpen" aria-hidden="true">
        <svg viewBox="0 0 24 24" width="14" height="14"><path fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" d="M9 6l6 6-6 6"/></svg>
      </span>
      <a ref="headLink" href="javascript:"
        @click="onClick" @keydown="onKeyDown"><span v-if="item.code >= 100000" class="code">{{ item.code }}</span><span class="name">{{ item.name }}</span><span v-if="item.start" class="years">{{ `${item.start}-${item.end ?? ''}` }}</span></a>
    </div>
    <LinkGroup :link-zips="linkZips" />
    <ul v-if="isOpen">
      <li v-if="group" class="vgroup">
        <div class="row" :class="{ obsolete: group.end }" title="市辖区为统计用分组代码，并非实际行政区，数据集未收录">
          <span class="toggle" :class="{ open: groupOpen }" @click="groupOpen = !groupOpen" aria-hidden="true">
            <svg viewBox="0 0 24 24" width="14" height="14"><path fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" d="M9 6l6 6-6 6"/></svg>
          </span>
          <span class="vlabel" @click="groupOpen = !groupOpen"><span v-if="group.code" class="code">{{ group.code }}</span><span class="name">市辖区</span><span class="tag">统计用</span><span class="years">{{ `${group.start}-${group.end ?? ''}` }}</span></span>
        </div>
        <ul v-if="groupOpen">
          <TreeItem v-for="it in districts" :item="it" :key="it.code * 10000 + it.start" />
        </ul>
      </li>
      <TreeItem v-for="it in others" :item="it" :key="it.code * 10000 + it.start" />
    </ul>
  </li>
</template>

<style scoped>
.row {
  display: flex;
  align-items: baseline;
  gap: 2px;
  padding: 1px 6px 1px 0;
  border-radius: 6px;
}

.row:hover {
  background: var(--hover);
}

.row.leaf {
  padding-left: 22px;
}

.row a {
  display: inline-flex;
  flex-wrap: wrap;
  align-items: baseline;
  column-gap: 8px;
  padding: 2px 4px;
  border-radius: 6px;
}

.row a:hover {
  text-decoration: none;
}

.row a:hover .name {
  color: var(--accent);
}

.code {
  font-family: var(--font-mono);
  font-size: .85em;
  color: var(--muted);
  font-variant-numeric: tabular-nums;
}

.years {
  font-size: .75em;
  color: var(--muted);
  padding: 0 6px;
  border-radius: 999px;
  background: var(--grid);
  font-variant-numeric: tabular-nums;
}

.guide .years {
  display: none;
}

.hit .name {
  font-weight: 650;
  color: var(--accent);
}

.hit {
  background: var(--accent-soft);
}

.obsolete .name {
  color: var(--muted);
  text-decoration: line-through;
  text-decoration-color: color-mix(in srgb, var(--muted) 50%, transparent);
}

.toggle {
  flex: none;
  display: inline-grid;
  place-items: center;
  width: 22px;
  height: 22px;
  align-self: center;
  color: var(--muted);
  cursor: pointer;
  user-select: none;
  transition: transform .15s;
}

.toggle.open {
  transform: rotate(90deg);
}
.vgroup > .row {
  border: 1px dashed var(--border, color-mix(in srgb, var(--muted) 45%, transparent));
  margin: 1px 0;
}

.vlabel {
  display: inline-flex;
  flex-wrap: wrap;
  align-items: baseline;
  column-gap: 8px;
  padding: 2px 4px;
  cursor: pointer;
  user-select: none;
}

.vlabel .name {
  font-style: italic;
  color: var(--muted);
}

.vgroup > .row.obsolete .name {
  text-decoration: line-through;
}

.tag {
  font-size: .7em;
  color: var(--muted);
  border: 1px solid currentColor;
  border-radius: 4px;
  padding: 0 4px;
  opacity: .8;
}
</style>
