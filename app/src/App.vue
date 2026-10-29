<script setup lang="ts">
import { computed, nextTick, provide, reactive, ref, useTemplateRef, watch } from 'vue';
import type { CodesJson, GlobalProps, Item, Link, SearchResult } from './common';
import { timeOrDefault, Action, inUse, inUseRange, encodeLink, decodeLink, exposeItem } from './common';
import TreeItem from './components/TreeItem.vue';
import EvolutionGraph from './components/EvolutionGraph.vue';
import codesUrl from '../../codes.json?url';

const root = ref<Item>({
  code: 0,
  name: "中华人民共和国",
  start: 1949,
  root: true,
});

const guide: Item = {
  code: 0,
  name: "凡例",
  start: 0,
  guide: true,
  action: localStorage.getItem("closeGuide") == null ? Action.Open : undefined,
  children: [
    {
      code: 1,
      name: "灰色标签为在用时间",
      start: 1980,
    },
    {
      code: 2,
      name: "灰色划线为弃用代码",
      start: 1970,
      end: 1980,
    },
    {
      code: 3,
      name: "向右的绿色箭头表明代码的后继",
      start: 0,
      succ: [{ code: 4, time: 1980, desc: 0 }]
    },
    {
      code: 4,
      name: "向左的红色箭头表明代码的前身",
      start: 0,
    },
    {
      code: 5,
      name: "点击蓝色方括号内数字查看变更描述",
      start: 0,
    },
    {
      code: 6,
      name: "支持键盘操作（加、减、退格）",
      start: 0,
    },
    {
      code: 7,
      name: "支持名称前缀、完整代码搜索",
      start: 0,
    },
    {
      code: 8,
      name: "虚线框“市辖区”为统计用分组，非实际行政区，数据未收录",
      start: 0,
    }
  ]
};

const options = reactive({
  hideSuccessors: false,
  hidePredecessors: false,
  searchText: "",
});

const view = ref<'list' | 'graph'>(localStorage.getItem('view') == 'graph' ? 'graph' : 'list');
watch(view, v => localStorage.setItem('view', v));

const theme = ref(localStorage.getItem('theme') ?? 'auto');
function applyTheme() {
  if (theme.value == 'auto') delete document.documentElement.dataset.theme;
  else document.documentElement.dataset.theme = theme.value;
}
applyTheme();
watch(theme, v => {
  if (v == 'auto') localStorage.removeItem('theme');
  else localStorage.setItem('theme', v);
  applyTheme();
});
function cycleTheme() {
  theme.value = theme.value == 'auto' ? 'dark' : theme.value == 'dark' ? 'light' : 'auto';
}
const themeLabel = computed(() => ({ auto: '跟随系统', dark: '深色', light: '浅色' } as Record<string, string>)[theme.value]);

const dataEnd = ref(2025);
const loading = ref(true);

const items = new Map<number, Item[]>();
const predecessors = new Map<number, Link[]>();
const descriptions = new Map<number, string[]>();

descriptions.set(1980, ["这是一条变更描述，可点击返回。"]);

let nameIndexMap: Map<string, Item[]> | undefined = new Map<string, Item[]>();
const nameIndex: { name: string; items: Item[]; }[] = [];

const timeIndex: Set<number>[][] = [];

const searchResult = computed(() => {
  const text = options.searchText;
  if (text.length == 0) {
    return undefined;
  }

  const hits = new Set<Item>(), links = new Set<number>();
  const res: SearchResult = { items: hits, hits, links };

  let start = 1984;
  let end: number | undefined;
  // Whether to recursively add successors and predecessors
  let extend = true;

  if (/^(19|20)\d{2}\.(\d+)?$/.test(text)) {
    extend = false;

    const parts = text.split('.');
    res.time = parseInt(parts[0]!);

    const arr = timeIndex[res.time];
    if (arr == undefined) return res;

    const processLink = (link: number) => {
      const [src, dst, time] = decodeLink(link);
      hits.add(resolve(src, time - 1)!);
      hits.add(resolve(dst, time)!);
      links.add(link);
    };

    if (parts[1] == '') {
      arr.forEach(subArr => subArr.forEach(processLink));
    } else {
      const idx = parseInt(parts[1]!);
      if (idx <= 0) return res;

      const subArr = arr[idx];
      if (subArr != undefined) {
        subArr.forEach(processLink);
        res.desc = idx - 1;
      }
    }
  } else if (/^\d{6}(,\d{4}(-(\d{4})?)?)?$/.test(text)) {
    let parts = text.split(',');
    const code = parseInt(parts[0]!);

    if (parts.length == 1) {
      items.get(code)?.forEach(item => hits.add(item));
    } else {
      parts = parts[1]!.split('-');
      start = parseInt(parts[0]!);
      if (parts.length == 1) {
        end = start + 1;
      } else if (parts[1] != '') {
        end = parseInt(parts[1]!);
      }
      resolveRange(code, start, end).forEach(item => hits.add(item));
    }
  } else {
    let i = binarySearch(nameIndex, t => t.name.localeCompare(text));
    while (i < nameIndex.length) {
      const { name, items } = nameIndex[i]!;
      if (!name.startsWith(text)) break;
      items.forEach(item => hits.add(item));
      i += 1;
    }
  }

  res.items = new Set(hits);
  const succ = new Set<Item>(), pred = new Set<Item>();

  if (extend) {
    for (const item of res.items) {
      item.children?.forEach(child => {
        if (inUseRange(child, start, end))
          res.items.add(child);
      });
      if (!options.hideSuccessors)
        addSuccessors(item, start ?? 0, succ, links);
      if (!options.hidePredecessors)
        addPredecessors(item, end ?? Infinity, pred, links);
    }
  }

  succ.forEach(item => res.items.add(item));
  pred.forEach(item => res.items.add(item));

  for (const item of res.items) {
    if (item.parent != undefined)
      res.items.add(item.parent);
  }
  return res;
});

function resolve(code: number, time: number): Item | undefined {
  return items.get(code)?.find(item => inUse(item, time));
}

function resolveRange(code: number, start: number, end?: number): Item[] {
  return items.get(code)?.filter(item => inUseRange(item, start, end)) ?? [];
}

function addSuccessors(item: Item, after: number, items: Set<Item>, links: Set<number>) {
  item.succ?.forEach(link => {
    const time = timeOrDefault(link, item);
    if (time > after) {
      const succ = resolve(link.code, time)!;
      items.add(succ);
      links.add(encodeLink(item.code, link.code, time));
      addSuccessors(succ, time, items, links);
    }
  });
}

function addPredecessors(item: Item, before: number, items: Set<Item>, links: Set<number>) {
  predecessors.get(item.code)?.forEach(link => {
    const time = link.time!;
    if (time < before && inUse(item, time)) {
      const pred = resolve(link.code, time - 1)!;
      items.add(pred);
      links.add(encodeLink(link.code, item.code, time));
      addPredecessors(pred, time, items, links);
    }
  });
}

watch(searchResult, res => {
  if (res == undefined) {
    root.value.children?.forEach(item => {
      item.action = Action.Close;
      item.act?.();
    });
  } else {
    const action = res.hits.size > 1 ? Action.None : Action.Open;
    res.hits.forEach(item => exposeItem(item, action));
  }
});

const history = ref<{
  searchText: string;
  item: Item;
}[]>([]);

function pushHistory(item: Item) {
  history.value.push({ searchText: options.searchText, item });
}

function popHistory() {
  const hist = history.value.pop();
  if (hist == undefined) return;

  options.searchText = hist.searchText;
  nextTick(() => exposeItem(hist.item, Action.Close | Action.Focus | Action.Scroll));
}

const descArticle = useTemplateRef('desc');

function scrollToDesc() {
  descArticle.value?.scrollIntoView();
}

const props: GlobalProps = {
  options,
  items,
  predecessors,
  descriptions,
  searchResult,
  dataEnd,
  resolve,
  pushHistory,
  scrollToDesc,
};
provide('props', props);

insertItem(guide);

fetch(codesUrl)
  .then(resp => resp.json())
  .then((resp: CodesJson) => {
    resp.items.forEach(item => insertItem(item));
    for (const [time, arr] of Object.entries(resp.descriptions)) {
      descriptions.set(parseInt(time), arr);
    }

    let end = 0;
    items.forEach(arr => arr.forEach(it => {
      if (it.code >= 100000) end = Math.max(end, it.start, it.end ?? 0);
    }));
    dataEnd.value = end;

    createIndex();
    shuffleExamples();
    followHash();
    root.value.children = resp.items;
    loading.value = false;
  });

const examples = ref<string[]>(["杭州", "330183", "哈密", "330183,1994", "2019."]);

function pick<T>(arr: T[]): T {
  return arr[Math.floor(Math.random() * arr.length)]!;
}

// 从数据中随机生成“试试”示例：名称、完整代码、代码+年份、年份变更。
function shuffleExamples() {
  const all: Item[] = [];
  items.forEach(arr => arr.forEach(it => { if (it.code >= 100000) all.push(it); }));
  if (all.length == 0) return;
  const changed = all.filter(it => it.end != undefined && it.succ?.length);
  const named = all.filter(it => it.code % 100 == 0 && it.code % 10000 != 0);
  const out = new Set<string>();
  const add = (s: string) => { if (s) out.add(s); };
  let guard = 0;
  while (out.size < 5 && guard++ < 50) {
    switch (out.size) {
      case 0: add(pick(named).name.replace(/(市|地区|盟|自治州)$/, '')); break;
      case 1: add(String(pick(all).code)); break;
      case 2: add(pick(all).name.replace(/(市|县|区|自治县|旗)$/, '') || ''); break;
      case 3: { const it = pick(changed); add(`${it.code},${it.start}`); break; }
      default: { const it = pick(changed); add(`${it.end}.`); break; }
    }
  }
  examples.value = [...out];
}

function insertItem(item: Item, parent?: Item) {
  let arr = items.get(item.code);
  if (arr == undefined) {
    arr = [];
    items.set(item.code, arr);
  }
  arr.push(item);

  arr = nameIndexMap!.get(item.name);
  if (arr == undefined) {
    arr = [];
    nameIndexMap!.set(item.name, arr);
  }
  arr.push(item);

  item.succ?.forEach(link => {
    let links = predecessors.get(link.code);
    if (links == undefined) {
      links = [];
      predecessors.set(link.code, links);
    }
    links.push({ time: timeOrDefault(link, item), code: item.code, desc: link.desc });
  });
  item.children?.forEach(child => insertItem(child, item));
  item.parent = parent;
}

function createIndex() {
  nameIndexMap!.forEach((items, name) => {
    nameIndex.push({ name, items });

    items.forEach(item => {
      item.succ?.forEach(link => {
        const time = timeOrDefault(link, item);
        const idx = link.desc != undefined ? link.desc + 1 : 0;

        let arr = timeIndex[time];
        if (arr == undefined) {
          arr = [new Set()];
          timeIndex[time] = arr;
        }
        let subArr = arr[idx];
        if (subArr == undefined) {
          subArr = new Set();
          arr[idx] = subArr;
        }
        subArr.add(encodeLink(item.code, link.code, time));
      });
    });
  });
  nameIndexMap = undefined;
  nameIndex.sort((a, b) => a.name.localeCompare(b.name));
}

// Stolen from `slice::binary_search_by` in Rust.
// In both cases a nonnegative index is returned.
function binarySearch<T>(arr: T[], f: (t: T) => number): number {
  let size = arr.length;
  let left = 0;
  let right = size;
  while (left < right) {
    const mid = left + (size >>> 1);
    const cmp = f(arr[mid]!);
    if (cmp < 0) {
      left = mid + 1;
    } else if (cmp > 0) {
      right = mid;
    } else {
      return mid;
    }
    size = right - left;
  }
  return left;
}

window.onhashchange = followHash;

function commitHash() {
  location.hash = encodeURI(options.searchText);
}

function followHash() {
  options.searchText = decodeURIComponent(location.hash.substring(1));
}

import markdownit from 'markdown-it';
const md = markdownit({ breaks: true });

// https://github.com/markdown-it/markdown-it/blob/master/docs/architecture.md#renderer
md.renderer.rules.link_open = function (tokens, idx, options, _env, self) {
  tokens[idx]!.attrSet('target', '_blank');
  return self.renderToken(tokens, idx, options);
};

function render(time: number, desc: number): string {
  const descStr = descriptions.get(time)![desc]!;
  return md.render(descStr
    .replace(/^#/gm, "##")
    .replace(/《国务院公报》(\d+)年第(\d+)号第(\d+)(-\d+)?页/g, (match, year, no, page) => {
      no = no.padStart(2, "0");
      return `[${match}](https://www.gov.cn/gongbao/shuju/${year}/gwyb${year}${no}.pdf#page=${page})`;
    })
  );
}
</script>

<template>
  <header class="topbar">
    <a class="brand" href="">
      <span class="logo" aria-hidden="true">区</span>
      <span>
        <span class="title">行政区划代码数据集</span>
        <span class="subtitle">1981–{{ dataEnd }} 年县级以上区划代码及新旧对应关系</span>
      </span>
    </a>
    <div class="topbar-actions">
      <button class="icon-btn" type="button" :title="`主题：${themeLabel}`" @click="cycleTheme">
        <svg v-if="theme == 'dark'" viewBox="0 0 24 24" width="18" height="18"><path fill="currentColor" d="M21 12.8A9 9 0 1 1 11.2 3a7 7 0 0 0 9.8 9.8z"/></svg>
        <svg v-else-if="theme == 'light'" viewBox="0 0 24 24" width="18" height="18"><circle cx="12" cy="12" r="4.5" fill="currentColor"/><g stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M12 2v2M12 20v2M4.9 4.9l1.4 1.4M17.7 17.7l1.4 1.4M2 12h2M20 12h2M4.9 19.1l1.4-1.4M17.7 6.3l1.4-1.4"/></g></svg>
        <svg v-else viewBox="0 0 24 24" width="18" height="18"><circle cx="12" cy="12" r="9" fill="none" stroke="currentColor" stroke-width="2"/><path fill="currentColor" d="M12 3a9 9 0 0 1 0 18z"/></svg>
      </button>
      <a class="icon-btn" title="GitHub" href="https://github.com/yescallop/areacodes">
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 16 16">
          <path fill="currentColor" fill-rule="evenodd"
            d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z" />
        </svg>
      </a>
    </div>
  </header>

  <div id="search-bar">
    <div class="search-row">
      <button v-if="history.length" class="icon-btn back" type="button" title="返回上一步" @click="popHistory">
        <svg viewBox="0 0 24 24" width="18" height="18"><path fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" d="M15 18l-6-6 6-6"/></svg>
      </button>
      <label class="search-box">
        <svg viewBox="0 0 24 24" width="18" height="18" aria-hidden="true"><circle cx="11" cy="11" r="7" fill="none" stroke="currentColor" stroke-width="2"/><path stroke="currentColor" stroke-width="2" stroke-linecap="round" d="M20 20l-3.5-3.5"/></svg>
        <input type="search" v-model="options.searchText" placeholder="名称前缀、6 位代码，或年份如 2019."
          aria-label="搜索" @keydown.enter="commitHash" />
        <a class="share" :href="'#' + encodeURI(options.searchText)" title="生成可分享链接">链接</a>
      </label>
    </div>
    <div class="tool-row">
      <div class="segmented" role="tablist">
        <button type="button" role="tab" :aria-selected="view == 'list'" :class="{ active: view == 'list' }"
          @click="view = 'list'">树状列表</button>
        <button type="button" role="tab" :aria-selected="view == 'graph'" :class="{ active: view == 'graph' }"
          @click="view = 'graph'">演变图</button>
      </div>
      <label class="chip"><input type="checkbox" v-model="options.hideSuccessors" />隐藏后继</label>
      <label class="chip"><input type="checkbox" v-model="options.hidePredecessors" />隐藏前身</label>
    </div>
  </div>
  <div id="spacer"></div>

  <main>
    <article ref="desc" class="desc-card"
      v-if="searchResult?.time && searchResult.desc != undefined"
      v-html="render(searchResult.time, searchResult.desc)" />

    <section v-if="!options.searchText" class="guide-card">
      <ul class="tree" id="guide">
        <TreeItem :item="guide" />
      </ul>
      <p class="examples">
        试试：
        <a v-for="ex in examples" :key="ex" :href="`#${ex}`">{{ ex }}</a>
        <button class="shuffle" type="button" title="换一批" @click="shuffleExamples">↻</button>
      </p>
    </section>

    <EvolutionGraph v-if="view == 'graph'" />

    <div v-if="loading" class="loading">正在加载数据…</div>
    <ul v-show="view == 'list'" class="tree tree-card" id="root">
      <TreeItem :item="root" />
    </ul>
  </main>
  <footer class="footer">
    数据以 <a href="https://creativecommons.org/publicdomain/zero/1.0/deed.zh-hans">CC0 1.0</a> 发布 ·
    <a href="https://raw.githubusercontent.com/yescallop/areacodes/master/result.csv">CSV</a> ·
    <a href="https://raw.githubusercontent.com/yescallop/areacodes/master/codes.json">JSON</a>
  </footer>
</template>

<style>
:root {
  --font-sans: system-ui, -apple-system, "Segoe UI", "PingFang SC", "Hiragino Sans GB", "Noto Sans SC", "Microsoft YaHei", sans-serif;
  --font-mono: ui-monospace, "SF Mono", "Roboto Mono", Menlo, Consolas, monospace;
  --radius: 12px;

  --bg: #f6f7f9;
  --surface: #ffffff;
  --text: #1c2330;
  --muted: #6b7380;
  --border: #e3e6eb;
  --grid: #eef0f3;
  --hover: #f1f4f9;
  --accent: #2563eb;
  --accent-soft: #e8efff;
  --succ: #15803d;
  --pred: #c2410c;
  --transfer: #d97706;
  --c-county: #3b82f6;
  --c-prefecture: #8b5cf6;
  --c-province: #0f766e;
  --shadow: 0 1px 2px rgb(16 24 40 / 6%), 0 1px 3px rgb(16 24 40 / 6%);
  color-scheme: light;
}

@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --bg: #0f1216;
    --surface: #171b21;
    --text: #e6e9ee;
    --muted: #9099a6;
    --border: #2a3039;
    --grid: #222830;
    --hover: #1f252d;
    --accent: #7aa2ff;
    --accent-soft: #1e2a44;
    --succ: #4ade80;
    --pred: #fb923c;
    --transfer: #fbbf24;
    --c-county: #60a5fa;
    --c-prefecture: #a78bfa;
    --c-province: #2dd4bf;
    --shadow: none;
    color-scheme: dark;
  }
}

:root[data-theme="dark"] {
  --bg: #0f1216;
  --surface: #171b21;
  --text: #e6e9ee;
  --muted: #9099a6;
  --border: #2a3039;
  --grid: #222830;
  --hover: #1f252d;
  --accent: #7aa2ff;
  --accent-soft: #1e2a44;
  --succ: #4ade80;
  --pred: #fb923c;
  --transfer: #fbbf24;
  --c-county: #60a5fa;
  --c-prefecture: #a78bfa;
  --c-province: #2dd4bf;
  --shadow: none;
  color-scheme: dark;
}

* {
  box-sizing: border-box;
}

html {
  scroll-padding-top: 120px;
  background: var(--bg);
}

body {
  margin: 0 auto;
  max-width: 1080px;
  padding: 0 16px 32px;
  font-family: var(--font-sans);
  color: var(--text);
  background: var(--bg);
  line-height: 1.6;
  -webkit-font-smoothing: antialiased;
}

a {
  color: inherit;
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}

a:focus-visible,
button:focus-visible,
input:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
  border-radius: 4px;
}

button {
  font: inherit;
  color: inherit;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 20px 0 12px;
}

.brand {
  display: flex;
  align-items: center;
  gap: 12px;
}

.brand:hover {
  text-decoration: none;
}

.logo {
  display: grid;
  place-items: center;
  width: 40px;
  height: 40px;
  border-radius: 10px;
  background: linear-gradient(135deg, #2563eb, #7c3aed);
  color: #fff;
  font-weight: 700;
  font-size: 20px;
  flex: none;
}

.title {
  display: block;
  font-size: 1.25rem;
  font-weight: 700;
  letter-spacing: .01em;
}

.subtitle {
  display: block;
  font-size: .85rem;
  color: var(--muted);
}

.topbar-actions {
  display: flex;
  gap: 4px;
}

.icon-btn {
  display: inline-grid;
  place-items: center;
  width: 36px;
  height: 36px;
  border-radius: 10px;
  border: 1px solid transparent;
  background: transparent;
  color: var(--muted);
  cursor: pointer;
}

.icon-btn:hover {
  background: var(--hover);
  color: var(--text);
  text-decoration: none;
}

#search-bar {
  position: sticky;
  top: 0;
  z-index: 10;
  margin: 0 -16px;
  padding: 10px 16px;
  background: color-mix(in srgb, var(--bg) 85%, transparent);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border-bottom: 1px solid var(--border);
}

.search-row {
  display: flex;
  gap: 8px;
  align-items: center;
}

.search-box {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 8px;
  height: 44px;
  padding: 0 6px 0 14px;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  color: var(--muted);
}

.search-box:focus-within {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px var(--accent-soft);
}

.search-box input {
  flex: 1;
  min-width: 0;
  border: 0;
  outline: 0;
  background: transparent;
  color: var(--text);
  font: inherit;
  font-size: 1rem;
}

.share {
  font-size: .8rem;
  padding: 4px 10px;
  border-radius: 8px;
  color: var(--accent);
  background: var(--accent-soft);
}

.share:hover {
  text-decoration: none;
  filter: brightness(.97);
}

.tool-row {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 8px;
  margin-top: 8px;
}

.segmented {
  display: inline-flex;
  padding: 3px;
  border-radius: 10px;
  background: var(--grid);
}

.segmented button {
  border: 0;
  background: transparent;
  padding: 4px 12px;
  border-radius: 8px;
  font-size: .875rem;
  color: var(--muted);
  cursor: pointer;
}

.segmented button.active {
  background: var(--surface);
  color: var(--text);
  box-shadow: var(--shadow);
}

.chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  border: 1px solid var(--border);
  border-radius: 999px;
  font-size: .85rem;
  color: var(--muted);
  cursor: pointer;
  user-select: none;
  background: var(--surface);
}

.chip:has(input:checked) {
  color: var(--accent);
  border-color: var(--accent);
  background: var(--accent-soft);
}

.chip input {
  margin: 0;
  accent-color: var(--accent);
}

#spacer {
  height: calc(100dvh - 120px);
  margin-bottom: calc(120px - 100dvh);
  position: sticky;
  top: 120px;
  z-index: -1;
}

.guide-card,
.tree-card,
.desc-card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  box-shadow: var(--shadow);
  margin: 12px 0;
}

.guide-card {
  padding: 8px 12px;
}

.examples {
  margin: 4px 0 4px 4px;
  font-size: .875rem;
  color: var(--muted);
}

.examples a {
  display: inline-block;
  margin: 2px 4px;
  padding: 1px 10px;
  border-radius: 999px;
  background: var(--grid);
  color: var(--text);
  font-family: var(--font-mono);
}

.examples .shuffle {
  border: none;
  background: none;
  color: var(--muted);
  font: inherit;
  cursor: pointer;
  padding: 0 4px;
}

.examples .shuffle:hover {
  color: var(--accent);
}

.examples a:hover {
  text-decoration: none;
  background: var(--accent-soft);
  color: var(--accent);
}

.desc-card {
  padding: 4px 20px 8px;
  line-height: 1.75;
}

.desc-card h2 {
  font-size: 1.05rem;
}

.desc-card a {
  color: var(--accent);
}

.tree {
  list-style-type: none;
  padding-left: 0;
  margin: 0;
}

.tree ul {
  list-style-type: none;
  padding-left: 1.25rem;
  margin: 0;
  border-left: 1px solid var(--grid);
  margin-left: 9px;
}

#root > li > ul {
  border-left: 0;
  margin-left: 0;
  padding-left: 0;
}

.tree-card {
  padding: 8px 8px 8px 4px;
  overflow-x: auto;
}

#guide rt {
  display: none;
}

.loading {
  margin: 24px 0;
  text-align: center;
  color: var(--muted);
}

.footer {
  margin-top: 24px;
  text-align: center;
  font-size: .8rem;
  color: var(--muted);
}

.footer a {
  color: var(--accent);
}

@media (max-width: 600px) {
  body {
    padding: 0 10px 24px;
  }

  #search-bar {
    margin: 0 -10px;
    padding: 8px 10px;
  }

  .subtitle {
    display: none;
  }

  .title {
    font-size: 1.1rem;
  }
}
</style>
