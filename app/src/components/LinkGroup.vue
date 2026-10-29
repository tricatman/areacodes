<script setup lang="ts">
import { computed, inject } from 'vue';
import { encodeLink, type GlobalProps, type Item, type LinkZip } from '@/common';
import LinkItem from './LinkItem.vue';

const props = defineProps<{ linkZips: LinkZip[]; }>();

const gProps = inject<GlobalProps>('props')!;

const srcItem = inject<Item>('srcItem')!;

const filteredLinkZips = computed(() => {
  const out = [];
  const links = gProps.searchResult.value?.links;
  for (const linkZip of props.linkZips) {
    let filtered = true;
    const items = linkZip.codes
      .map(it => {
        let src = srcItem.code, dst = it.code;
        if (linkZip.rev) [src, dst] = [dst, src];

        const enabled = src < 100000 || links == undefined ||
          links.has(encodeLink(src, dst, linkZip.time));
        if (enabled) filtered = false;

        const item = gProps.resolve(it.code, linkZip.time - (linkZip.rev ? 1 : 0))!;
        return { item, desc: it.desc, showDesc: it.showDesc, enabled };
      });
    if (!filtered)
      out.push({ items, time: linkZip.time, rev: linkZip.rev });
  }
  return out;
});
</script>
<template>
  <ul v-if="filteredLinkZips.length" class="links">
    <li v-for="linkZip in filteredLinkZips" :class="{ rev: linkZip.rev }" :key="linkZip.time">
      <span class="arrow">{{ linkZip.rev ? "←" : "→" }}</span>
      <template v-for="(it, index) in linkZip.items" :key="it.item.code">
        <template v-if="index != 0">,<wbr /></template>
        <LinkItem :item="it.item" :enabled="it.enabled"
          :desc="it.desc != undefined ? [linkZip.time, it.desc] : undefined"
          :show-desc="it.showDesc" />
      </template>
      <span class="time">{{ linkZip.time }}</span>
    </li>
  </ul>
</template>

<style scoped>
.links {
  list-style: none;
  color: var(--succ);
  margin: 0 0 2px 26px;
  padding: 0;
  font-size: .9em;
  font-family: var(--font-mono);
}

.links li {
  padding-left: 1.4em;
  text-indent: -1.4em;
}

.links li.rev {
  color: var(--pred);
}

.arrow {
  display: inline-block;
  width: 1.4em;
  text-indent: 0;
  font-weight: 700;
}

.time {
  margin-left: 6px;
  font-size: .8em;
  padding: 0 6px;
  border-radius: 999px;
  background: var(--grid);
  color: var(--muted);
}
</style>
