<script setup lang="ts">
import { computed, inject, nextTick, onMounted, onUnmounted, ref, useTemplateRef } from 'vue';
import type { GlobalProps, Item } from '@/common';
import { Action, decodeLink, exposeItem } from '@/common';

const MAX_NODES = 400;
const ROW = 30;
const AXIS_H = 28;
const BAR_H = 10;

const gProps = inject<GlobalProps>('props')!;
const hovered = ref<Item | undefined>();

const card = useTemplateRef<HTMLElement>('card');
const cardWidth = ref(1000);
let observer: ResizeObserver | undefined;
onMounted(() => {
  observer = new ResizeObserver(entries => {
    cardWidth.value = entries[0]!.contentRect.width;
  });
  if (card.value) observer.observe(card.value);
});
onUnmounted(() => observer?.disconnect());

const compact = computed(() => cardWidth.value < 640);
const LABEL_W_VAL = computed(() => compact.value ? 108 : 240);
const YEAR_W_VAL = computed(() => {
  const g = graph.value;
  if (!g?.order) return 18;
  const years = g.maxYear! - g.minYear!;
  return Math.max(5, Math.min(28, (cardWidth.value - LABEL_W_VAL.value - 24) / years));
});

interface Edge {
  src: Item,
  dst: Item,
  time: number,
  desc?: number,
  /** The source stays in use after the change (area transfer only). */
  transfer: boolean,
  /** Inferred from where the subordinate counties went; not in the data. */
  inferred?: number,
}

function isPrefecture(item?: Item): item is Item {
  return item != undefined && item.code % 100 == 0 && item.code % 10000 != 0;
}

/**
 * 数据只给县级记录了去向，地级（如徽州地区→黄山市）常常没有直接关联。
 * 这里按下辖区县的实际去向推断地级之间的关联：某年 P 的下辖区县划入 Q，
 * 且当年 P 撤销或 Q 新设，就画一条 P→Q 的推断连线。
 */
function inferPrefectureEdges(item: Item, resolve: GlobalProps['resolve'],
  predecessors: GlobalProps['predecessors']): Edge[] {
  const counts = new Map<string, { src: Item, dst: Item, time: number, n: number }>();
  const add = (src: Item, dst: Item, time: number) => {
    if (src == dst || !(src.end == time || dst.start == time)) return;
    const key = `${src.code},${src.start}>${dst.code},${dst.start}@${time}`;
    const e = counts.get(key);
    if (e) e.n += 1;
    else counts.set(key, { src, dst, time, n: 1 });
  };
  for (const child of item.children ?? []) {
    // 下辖区县划出
    for (const link of child.succ ?? []) {
      const time = link.time ?? child.end!;
      const q = resolve(link.code, time)?.parent;
      if (isPrefecture(q)) add(item, q, time);
    }
    // 下辖区县由别处划入
    for (const link of predecessors.get(child.code) ?? []) {
      const time = link.time!;
      if (time < child.start || (child.end != undefined && time >= child.end)) continue;
      const p = resolve(link.code, time - 1)?.parent;
      if (isPrefecture(p)) add(p, item, time);
    }
  }
  return [...counts.values()].map(({ src, dst, time, n }) => ({
    src, dst, time, inferred: n, transfer: src.end == undefined || src.end > time,
  }));
}

const graph = computed(() => {
  const res = gProps.searchResult.value;
  if (res == undefined) return undefined;

  const nodes = new Set<Item>();
  const edges: Edge[] = [];

  res.hits.forEach(item => { if (item.code >= 100000) nodes.add(item); });

  for (const link of res.links) {
    const [srcCode, dstCode, time] = decodeLink(link);
    const src = gProps.resolve(srcCode, time - 1);
    const dst = gProps.resolve(dstCode, time);
    if (src == undefined || dst == undefined) continue;
    nodes.add(src);
    nodes.add(dst);
    const desc = src.succ?.find(l => l.code == dstCode && (l.time ?? src.end) == time)?.desc;
    edges.push({ src, dst, time, desc, transfer: src.end == undefined || src.end > time });
  }

  // 补上数据里缺失的地级关联
  const has = new Set(edges.map(e => `${e.src.code},${e.src.start}>${e.dst.code},${e.dst.start}`));
  for (const item of [...nodes]) {
    if (!isPrefecture(item)) continue;
    for (const e of inferPrefectureEdges(item, gProps.resolve, gProps.predecessors)) {
      const key = `${e.src.code},${e.src.start}>${e.dst.code},${e.dst.start}`;
      if (has.has(key)) continue;
      has.add(key);
      nodes.add(e.src);
      nodes.add(e.dst);
      edges.push(e);
    }
  }

  if (nodes.size == 0 || nodes.size > MAX_NODES) {
    return { tooMany: nodes.size > MAX_NODES, count: nodes.size };
  }

  // Order rows by following successors depth-first, so that a lineage
  // stays together and the curves stay short.
  const out = new Map<Item, Item[]>();
  const indeg = new Map<Item, number>();
  for (const e of edges) {
    if (!out.has(e.src)) out.set(e.src, []);
    out.get(e.src)!.push(e.dst);
    indeg.set(e.dst, (indeg.get(e.dst) ?? 0) + 1);
  }
  const byCode = (a: Item, b: Item) => a.start - b.start || a.code - b.code;
  const order: Item[] = [];
  const seen = new Set<Item>();
  const visit = (item: Item) => {
    if (seen.has(item)) return;
    seen.add(item);
    order.push(item);
    (out.get(item) ?? []).slice().sort(byCode).forEach(visit);
  };
  const all = [...nodes].sort(byCode);
  all.filter(it => !indeg.has(it)).forEach(visit);
  all.forEach(visit);

  let minYear = Infinity, maxYear = 0;
  for (const it of order) {
    minYear = Math.min(minYear, it.start);
    maxYear = Math.max(maxYear, it.end ?? gProps.dataEnd.value + 1);
  }
  minYear = Math.max(Math.min(minYear, maxYear - 6), 1980);

  const row = new Map<Item, number>();
  order.forEach((it, i) => row.set(it, i));
  return { tooMany: false, count: order.length, order, edges, row, minYear, maxYear };
});

function x(year: number): number {
  return LABEL_W_VAL.value + (year - graph.value!.minYear!) * YEAR_W_VAL.value;
}

function y(item: Item): number {
  return AXIS_H + graph.value!.row!.get(item)! * ROW + ROW / 2;
}

const width = computed(() => graph.value?.order ? x(graph.value.maxYear!) + 24 : 0);
const height = computed(() => graph.value?.order ? AXIS_H + graph.value.order.length * ROW + 8 : 0);

const ticks = computed(() => {
  const g = graph.value;
  if (!g?.order) return [];
  const out = [];
  const span = (g.maxYear! - g.minYear!) * YEAR_W_VAL.value;
  const years = g.maxYear! - g.minYear!;
  const step = [1, 2, 5, 10, 20].find(s => span / (years / s) >= 44) ?? 20;
  for (let yr = Math.ceil(g.minYear! / step) * step; yr <= g.maxYear!; yr += step) out.push(yr);
  return out;
});

function edgePath(e: Edge): string {
  const xs = x(e.time), y1 = y(e.src), y2 = y(e.dst);
  const r = Math.min(10, Math.abs(y2 - y1) / 2);
  if (r == 0) return `M${xs - 8},${y1}H${xs + 8}`;
  const dir = y2 > y1 ? 1 : -1;
  return `M${xs - 14},${y1}H${xs - r}Q${xs},${y1} ${xs},${y1 + dir * r}` +
    `V${y2 - dir * r}Q${xs},${y2} ${xs + r},${y2}H${xs + 6}`;
}

function level(item: Item): string {
  if (item.code % 10000 == 0) return 'province';
  if (item.code % 100 == 0) return 'prefecture';
  return 'county';
}

function isConnected(e: Edge): boolean {
  return hovered.value == undefined || e.src == hovered.value || e.dst == hovered.value;
}

function selectItem(item: Item) {
  gProps.pushHistory(item);
  gProps.options.searchText = `${item.code},${item.start}-${item.end ?? ''}`;
  nextTick(() => exposeItem(item, Action.None));
}

function selectEdge(e: Edge) {
  if (e.desc == undefined) return;
  gProps.pushHistory(e.src);
  gProps.options.searchText = `${e.time}.${e.desc + 1}`;
  nextTick(gProps.scrollToDesc);
}
</script>

<template>
  <div ref="card" class="graph-card">
    <p v-if="!graph" class="hint">输入名称或代码后，这里会画出它的演变过程。</p>
    <p v-else-if="graph.tooMany" class="hint">
      结果包含 {{ graph.count }} 个区划，超过 {{ MAX_NODES }} 个，图会太密。请缩小搜索范围，例如输入完整代码或加上年份。
    </p>
    <p v-else-if="!graph.order" class="hint">没有可以画出的区划。</p>
    <template v-else>
      <div class="legend">
        <span><i class="swatch county" />县级</span>
        <span><i class="swatch prefecture" />地级</span>
        <span><i class="swatch province" />省级</span>
        <span><i class="line" />代码变更</span>
        <span><i class="line transfer" />区域划转</span>
        <span><i class="line inferred" />地级关联（按下辖区县推断）</span>
      </div>
      <div class="scroller">
        <svg :width="width" :height="height" :viewBox="`0 0 ${width} ${height}`" role="img"
          aria-label="行政区划演变图">
          <g class="axis">
            <template v-for="yr in ticks" :key="yr">
              <line :x1="x(yr)" :x2="x(yr)" :y1="AXIS_H - 6" :y2="height" />
              <text :x="x(yr)" :y="AXIS_H - 12">{{ yr }}</text>
            </template>
          </g>
          <g class="edges">
            <path v-for="(e, i) in graph.edges" :key="i" :d="edgePath(e)"
              :class="{ transfer: e.transfer, inferred: e.inferred, dim: !isConnected(e), clickable: e.desc != undefined }"
              @click="selectEdge(e)">
              <title>{{ e.src.name }} → {{ e.dst.name }}（{{ e.time }}）{{ e.inferred ? `，推断：${e.inferred} 个下辖区县的去向` : '' }}{{ e.desc != undefined ? '，点击查看变更描述' : '' }}</title>
            </path>
          </g>
          <g v-for="it in graph.order" :key="it.code * 10000 + it.start" class="node"
            :class="[level(it), { obsolete: it.end != undefined, dim: hovered && hovered != it }]"
            @mouseenter="hovered = it" @mouseleave="hovered = undefined" @click="selectItem(it)">
            <rect class="row-hit" x="0" :y="y(it) - ROW / 2" :width="width" :height="ROW" />
            <text v-if="!compact" class="code" x="12" :y="y(it) + 4">{{ it.code }}</text>
            <text class="name" :x="compact ? 10 : 70" :y="y(it) + 4">{{ compact && it.name.length > 7 ? it.name.slice(0, 6) + '…' : it.name }}</text>
            <rect class="bar" :x="x(it.start)" :y="y(it) - BAR_H / 2"
              :width="Math.max(x(it.end ?? graph.maxYear!) - x(it.start) - 2, 4)" :height="BAR_H" rx="5" />
            <title>{{ it.code }} {{ it.name }}（{{ it.start }}-{{ it.end ?? '' }}）</title>
          </g>
        </svg>
      </div>
      <p class="caption">横轴为年份，每一行是一个代码的在用时段。点击行可查看该区划，点击带描述的连线可查看变更描述。</p>
    </template>
  </div>
</template>

<style scoped>
.graph-card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 12px 0;
  margin: 12px 0;
}

.hint,
.caption {
  color: var(--muted);
  margin: 4px 16px;
  font-size: 0.9rem;
}

.legend {
  display: flex;
  flex-wrap: wrap;
  gap: 4px 16px;
  margin: 0 16px 8px;
  font-size: 0.85rem;
  color: var(--muted);
}

.legend span {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.swatch {
  width: 18px;
  height: 8px;
  border-radius: 4px;
  display: inline-block;
}

.swatch.county { background: var(--c-county); }
.swatch.prefecture { background: var(--c-prefecture); }
.swatch.province { background: var(--c-province); }

.line {
  width: 18px;
  border-top: 2px solid var(--succ);
  display: inline-block;
}

.line.transfer {
  border-top: 2px dashed var(--transfer);
}

.line.inferred {
  border-top: 2px dotted var(--muted);
}

.scroller {
  overflow-x: auto;
}

svg {
  display: block;
  font-family: var(--font-sans);
}

.axis line {
  stroke: var(--grid);
}

.axis text {
  fill: var(--muted);
  font-size: 11px;
  text-anchor: middle;
  font-variant-numeric: tabular-nums;
}

.edges path {
  fill: none;
  stroke: var(--succ);
  stroke-width: 2;
  transition: opacity .15s;
}

.edges path.transfer {
  stroke: var(--transfer);
  stroke-dasharray: 4 3;
}

.edges path.inferred {
  stroke: var(--muted);
  stroke-dasharray: 1 4;
  stroke-linecap: round;
}

.edges path.clickable {
  cursor: pointer;
}

.edges path.dim {
  opacity: .15;
}

.node {
  cursor: pointer;
  transition: opacity .15s;
}

.node.dim {
  opacity: .45;
}

.row-hit {
  fill: transparent;
}

.node:hover .row-hit {
  fill: var(--hover);
}

.code {
  fill: var(--muted);
  font-family: var(--font-mono);
  font-size: 12px;
}

.name {
  fill: var(--text);
  font-size: 13px;
}

.bar {
  fill: var(--c-county);
}

.prefecture .bar { fill: var(--c-prefecture); }
.province .bar { fill: var(--c-province); }

.obsolete .bar {
  opacity: .55;
}

.obsolete .name {
  fill: var(--muted);
}
</style>
