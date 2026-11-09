<script setup>
// 視圖 4：系統 × 曲目熱圖（seriation）。列依排行、欄依「難度」（非 oracle 系統的中位數）排序，
// 讓「所有系統都難的歌」浮現成整欄的淺色。點欄選取曲目 → 下方出現逐秒時間線。
import { ref, onMounted, watch } from 'vue'
import * as d3 from 'd3'
import { CATEGORY } from '../series.js'
import { showTip, moveTip, hideTip } from '../tooltip.js'

const props = defineProps({
  systems: { type: Array, required: true },    // [{ system, category, median }] 已排序
  tracks: { type: Array, required: true },     // [{ track_id, artist, title, difficulty }] 已排序（難 → 易）
  value: { type: Function, required: true },   // (system, track_id) → number | NaN
  selected: { type: Object, required: true },
  selectedTrack: { type: String, default: '' },
  color: { type: Function, required: true },
  metric: { type: String, required: true },
})
const emit = defineEmits(['select', 'selectTrack'])

const host = ref(null)
const W = 880
const ML = 56
const MT = 110
const RH = 13

function draw() {
  const el = host.value
  if (!el) return
  d3.select(el).selectAll('*').remove()
  if (!props.systems.length) return
  const n = props.tracks.length
  const CW = (W - ML - 4) / n
  const H = MT + props.systems.length * RH + 22
  const svg = d3.select(el).append('svg').attr('viewBox', [0, 0, W, H])
    .attr('role', 'img').attr('aria-label', `各系統在 50 首歌上的 ${props.metric}，曲目由難到易排列`)

  // 欄標題（曲名，旋轉），可點擊
  const head = svg.append('g').selectAll('g').data(props.tracks, d => d.track_id).join('g')
    .attr('transform', (d, j) => `translate(${ML + j * CW + CW / 2},${MT - 6})`)
    .style('cursor', 'pointer')
    .on('click', (e, d) => emit('selectTrack', d.track_id))
    .on('pointerenter', (e, d) => showTip(e, `<b>${d.artist} – ${d.title}</b><br>非 oracle 系統 ${props.metric} 中位數 ${Number.isFinite(d.difficulty) ? d.difficulty.toFixed(2) + ' dB' : '無'}<br><span class="tip-muted">點擊看逐秒分數</span>`))
    .on('pointermove', moveTip).on('pointerleave', hideTip)
  head.append('text').attr('class', d => d.track_id === props.selectedTrack ? 'track-label on' : 'track-label')
    .attr('transform', 'rotate(-62)').attr('x', 4).attr('y', 4)
    .text(d => (d.artist + ' – ' + d.title).length > 26 ? (d.artist + ' – ' + d.title).slice(0, 25) + '…' : d.artist + ' – ' + d.title)

  // 選取曲目的欄位高亮
  if (props.selectedTrack) {
    const j = props.tracks.findIndex(t => t.track_id === props.selectedTrack)
    if (j >= 0) svg.append('rect').attr('class', 'col-hl').attr('x', ML + j * CW - 1).attr('y', MT - 2)
      .attr('width', CW + 2).attr('height', props.systems.length * RH + 4).attr('rx', 3)
  }

  const row = svg.append('g').selectAll('g').data(props.systems, d => d.system).join('g')
    .attr('transform', (d, i) => `translate(0,${MT + i * RH})`)
  row.append('rect').attr('x', 2).attr('y', 0).attr('width', ML - 6).attr('height', RH).attr('rx', 2)
    .attr('class', d => props.selected.has(d.system) ? 'row-bg selected' : 'row-bg')
    .style('cursor', 'pointer').on('click', (e, d) => emit('select', d.system))
  row.append('rect').attr('x', 6).attr('y', RH / 2 - 4).attr('width', 8).attr('height', 8).attr('rx', 2)
    .style('fill', d => CATEGORY[d.category].color).style('pointer-events', 'none')
  row.append('text').attr('class', 'sys-label small').attr('x', 18).attr('y', RH / 2 + 3.5).text(d => d.system)
    .style('pointer-events', 'none')

  row.selectAll('rect.hm').data(d => props.tracks.map((t, j) => ({ system: d.system, track: t, j, v: props.value(d.system, t.track_id) }))).join('rect')
    .attr('class', 'hm')
    .attr('x', d => ML + d.j * CW + 0.5).attr('y', 0.5).attr('width', CW - 1).attr('height', RH - 1)
    .style('fill', d => Number.isFinite(d.v) ? props.color(d.v) : 'var(--surface-2)')
    .style('cursor', 'pointer')
    .on('click', (e, d) => emit('selectTrack', d.track.track_id))
    .on('pointerenter', (e, d) => showTip(e, `<b>${d.system}</b> · ${d.track.artist} – ${d.track.title}<br>${props.metric} ${Number.isFinite(d.v) ? d.v.toFixed(2) + ' dB' : '無'}`))
    .on('pointermove', moveTip).on('pointerleave', hideTip)

  const yb = MT + props.systems.length * RH + 16
  svg.append('text').attr('class', 'axis-label').attr('x', ML).attr('y', yb).text('← 最難（非 oracle 系統中位數最低）')
  svg.append('text').attr('class', 'axis-label').attr('x', W - 4).attr('y', yb).attr('text-anchor', 'end').text('最易 →')
}

onMounted(draw)
watch(() => [props.systems, props.tracks, props.selected, props.selectedTrack, props.metric], draw, { flush: 'post' })
</script>

<template>
  <div ref="host" class="chart"></div>
</template>
