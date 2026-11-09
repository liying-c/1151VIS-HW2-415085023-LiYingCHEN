<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import * as d3 from 'd3'
import ControlBar from './components/ControlBar.vue'
import Leaderboard from './components/Leaderboard.vue'
import MethodMap from './components/MethodMap.vue'
import DominanceGraph from './components/DominanceGraph.vue'
import TrackHeatmap from './components/TrackHeatmap.vue'
import Timeline from './components/Timeline.vue'
import { loadAll, loadFrames, profile, authorSet } from './data.js'
import { median, pearson, wilcoxon, classicalMds, transitiveReduction, layerByLongestPath, isNum } from './stats.js'
import { CATEGORIES, CATEGORY, METRICS, TARGETS } from './series.js'
import { tip } from './tooltip.js'

const base = import.meta.env.BASE_URL
const ALPHA = 0.05

// ---------- 狀態 ----------
const data = ref(null)
const loading = ref(true)
const error = ref('')
const metric = ref('SDR')
const target = ref('vocals')
const cats = ref(new Set(CATEGORIES.map(c => c.key)))
const selected = ref(new Set())
const selectedTrack = ref('')
const frames = ref(null)
const dark = ref(window.matchMedia('(prefers-color-scheme: dark)').matches)
window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', e => { dark.value = e.matches })

// 網址參數：?m=SDR&s=vocals&c=supervised,blind&sel=TAK2,UHL3&t=track-id
const q = new URLSearchParams(location.search)
if (METRICS.some(m => m.key === q.get('m'))) metric.value = q.get('m')
if (TARGETS.some(t => t.key === q.get('s'))) target.value = q.get('s')
if (q.get('c')) { const c = new Set(q.get('c').split(',').filter(k => CATEGORY[k])); if (c.size) cats.value = c }
if (q.get('sel')) selected.value = new Set(q.get('sel').split(',').filter(Boolean))
if (q.get('t')) selectedTrack.value = q.get('t')

watch([metric, target, cats, selected, selectedTrack], () => {
  const p = new URLSearchParams()
  if (metric.value !== 'SDR') p.set('m', metric.value)
  if (target.value !== 'vocals') p.set('s', target.value)
  if (cats.value.size !== CATEGORIES.length) p.set('c', [...cats.value].join(','))
  if (selected.value.size) p.set('sel', [...selected.value].join(','))
  if (selectedTrack.value) p.set('t', selectedTrack.value)
  history.replaceState(null, '', p.size ? `?${p}` : location.pathname)
})

onMounted(async () => {
  try {
    data.value = await loadAll(base)
    if (selectedTrack.value && !data.value.trackInfo.has(selectedTrack.value)) selectedTrack.value = ''
  } catch (e) {
    error.value = `資料載入失敗：${e.message}\n請先執行 npm run data 產生 public/data/*.csv。`
  } finally {
    loading.value = false
  }
})

// ---------- 由狀態推導的資料（computed 會快取，只有上游變了才重算） ----------
const trackIds = computed(() => data.value ? data.value.tracks.map(t => t.track_id) : [])
const trackIndex = computed(() => new Map(trackIds.value.map((t, i) => [t, i])))
const visible = computed(() => data.value ? data.value.systems.filter(s => cats.value.has(s.category)) : [])

// 每個可見系統在目前 stem / 指標上的 50 首歌分數剖面
const profiles = computed(() => new Map(visible.value.map(s => [s.system, profile(data.value.index, s.system, target.value, metric.value, trackIds.value)])))
const medians = computed(() => new Map([...profiles.value].map(([s, p]) => [s, median(p)])))
const sortDesc = (a, b) => (isNum(b.median) ? b.median : -1e9) - (isNum(a.median) ? a.median : -1e9)
const ranking = computed(() => visible.value.map(s => ({ system: s.system, category: s.category, median: medians.value.get(s.system) })).sort(sortDesc))

// 視圖 1：排行榜，五個 stem 的中位數
const leaderboardRows = computed(() => ranking.value.map(r => ({
  ...r,
  values: Object.fromEntries(TARGETS.map(t => [t.key, median(profile(data.value.index, r.system, t.key, metric.value, trackIds.value))])),
})))

// 顏色：單色漸層（藍），低 → 高。深色模式用同一組色階反向，讓「高」是亮的。
const SEQ_LIGHT = ['#cde2fb', '#86b6ef', '#3987e5', '#1c5cab', '#0d366b']
const SEQ_DARK = ['#184f95', '#256abf', '#3987e5', '#86b6ef', '#cde2fb']
const colorDomain = computed(() => {
  const vis = new Set(visible.value.map(s => s.system))
  const vals = data.value.scores.filter(r => vis.has(r.system) && isNum(r[metric.value])).map(r => r[metric.value]).sort(d3.ascending)
  return [d3.quantile(vals, 0.05) ?? 0, d3.quantile(vals, 0.99) ?? 10]
})
const color = computed(() => {
  const sc = d3.scaleSequential(d3.interpolateRgbBasis(dark.value ? SEQ_DARK : SEQ_LIGHT)).domain(colorDomain.value).clamp(true)
  return v => sc(v)
})
const textOn = fill => (d3.hsl(fill).l > 0.62 ? '#0b0b0b' : '#ffffff')

// 視圖 2：MDS。距離 = sqrt(2(1 − r))，即標準化剖面之間的歐氏距離。
const mapPoints = computed(() => {
  const sys = visible.value
  if (sys.length < 3) return []
  const dist = sys.map(a => sys.map(b => {
    if (a === b) return 0
    const r = pearson(profiles.value.get(a.system), profiles.value.get(b.system))
    return isNum(r) ? Math.sqrt(Math.max(0, 2 * (1 - r))) : 2
  }))
  const xy = classicalMds(dist)
  return sys.map((s, i) => ({ system: s.system, category: s.category, x: xy[i][0], y: xy[i][1], median: medians.value.get(s.system), info: s }))
})
const authorEdges = computed(() => {
  const sets = visible.value.map(s => ({ system: s.system, authors: authorSet(s.authors) }))
  const edges = []
  for (let i = 0; i < sets.length; i++) for (let j = i + 1; j < sets.length; j++) {
    const shared = [...sets[i].authors].filter(a => sets[j].authors.has(a))
    if (shared.length) edges.push({ a: sets[i].system, b: sets[j].system, shared: shared.map(n => n.replace(/\b\w/g, c => c.toUpperCase())) })
  }
  return edges
})

// 視圖 3：兩兩比較 → 顯著優於的邊 → 遞移化簡 → 分層
const dominance = computed(() => {
  const sys = ranking.value.map(r => r.system)
  const cache = new Map()
  const win = (a, b) => {
    const key = a + '|' + b
    if (cache.has(key)) return cache.get(key)
    const pa = profiles.value.get(a), pb = profiles.value.get(b)
    let wins = 0, n = 0
    for (let i = 0; i < pa.length; i++) if (isNum(pa[i]) && isNum(pb[i])) { n++; if (pa[i] > pb[i]) wins++ }
    const res = { wins, n, rate: n ? wins / n : NaN, p: wilcoxon(pa, pb).p }
    cache.set(key, res)
    return res
  }
  const edges = []
  for (let i = 0; i < sys.length; i++) for (let j = i + 1; j < sys.length; j++) {
    const s = win(sys[i], sys[j])
    if (s.p < ALPHA && s.rate > 0.5) edges.push([sys[i], sys[j]])
  }
  const reduced = transitiveReduction(sys, edges)
  return { win, edges: reduced, allEdges: edges.length, layers: layerByLongestPath(sys, reduced) }
})

// 視圖 4：曲目難度 = 非 oracle 可見系統的中位數，由難到易
const tracksSorted = computed(() => {
  const pool = visible.value.filter(s => s.category !== 'oracle')
  const use = pool.length ? pool : visible.value
  return trackIds.value.map((t, k) => ({
    ...data.value.trackInfo.get(t),
    difficulty: median(use.map(s => profiles.value.get(s.system)[k])),
  })).sort((a, b) => (isNum(a.difficulty) ? a.difficulty : 1e9) - (isNum(b.difficulty) ? b.difficulty : 1e9))
})
const valueAt = (system, trackId) => profiles.value.get(system)?.[trackIndex.value.get(trackId)] ?? NaN

// 視圖 5：時間線要畫的系統 = 已選取者，否則排行前 5
const timelineSystems = computed(() => {
  const picked = ranking.value.filter(r => selected.value.has(r.system))
  return picked.length ? picked : ranking.value.slice(0, 5)
})
const selectedTrackInfo = computed(() => data.value?.trackInfo.get(selectedTrack.value))
watch([selectedTrack, data], async ([t]) => {
  frames.value = null
  if (!t || !data.value) return
  try { frames.value = await loadFrames(base, t) } catch { frames.value = null }
}, { immediate: true })

// ---------- 操作 ----------
function toggleSelect(system) {
  const next = new Set(selected.value)
  next.has(system) ? next.delete(system) : next.add(system)
  selected.value = next
}
function selectTrack(id) { selectedTrack.value = selectedTrack.value === id ? '' : id }
function clearSelection() { selected.value = new Set(); selectedTrack.value = '' }
const metricInfo = computed(() => METRICS.find(m => m.key === metric.value))
const targetInfo = computed(() => TARGETS.find(t => t.key === target.value))
const fmt = v => (isNum(v) ? v.toFixed(2) : '–')
</script>

<template>
  <main>
    <header>
      <h1>SiSEC 2018 音樂分離評測：31 個系統、50 首歌、4 個指標的互動探索</h1>
      <p>2018 年的 Signal Separation Evaluation Campaign 讓 31 個音樂分離系統在同樣 50 首歌上把人聲、鼓、貝斯、其他樂器拆出來，並用 BSS Eval 逐秒評分。這個頁面把評測結果做成五個互相連動的視圖：誰最強、誰的行為相似、誰顯著贏誰、哪些歌對所有系統都難、分數在一首歌裡怎麼隨時間變化。</p>
      <p class="meta">1151VIS HW2 · 互動式視覺化 · Vue 3 + D3.js v7 · 資料：<a href="https://github.com/sigsep/sigsep-mus-2018">sigsep-mus-2018</a>（Stöter, Liutkus &amp; Ito, LVA/ICA 2018）</p>
    </header>

    <p v-if="loading" class="status">資料載入中…</p>
    <p v-else-if="error" class="error">{{ error }}</p>
    <template v-else>
      <ControlBar v-model:metric="metric" v-model:target="target" v-model:cats="cats"
        :selected-count="selected.size" :selected-track="selectedTrackInfo ? selectedTrackInfo.title : ''" @clear="clearSelection" />

      <div class="grid-2">
        <figure>
          <h2>圖 1　排行榜 <span class="hint">{{ metricInfo.name }} 中位數 · 點欄名切換 stem，點列選取系統</span></h2>
          <p class="desc">列是系統，欄是五個 stem，格子是該系統在 50 首歌上 {{ metricInfo.name }} 的中位數（dB），顏色越深越高。依目前 stem（{{ targetInfo.name }}）排序。</p>
          <Leaderboard :rows="leaderboardRows" :target="target" :metric="metric" :selected="selected" :color="color" :text-on="textOn"
            @select="toggleSelect" @set-target="target = $event" />
        </figure>
        <figure>
          <h2>圖 2　方法地圖 <span class="hint">MDS · 連線 = 共同作者</span></h2>
          <p class="desc">每個系統一個點，位置由 50 首歌分數剖面的相關係數經 MDS 投影而來：靠近 = 在同樣的歌上同樣好或同樣壞。連線是真實關係：兩個系統有共同作者。相似的點之間剛好有線，表示「行為相似」多半來自「同一團隊的變體」。</p>
          <MethodMap :points="mapPoints" :edges="authorEdges" :selected="selected" :metric="metric" @select="toggleSelect" />
        </figure>
      </div>

      <figure>
        <h2>圖 3　優勢圖 <span class="hint">誰顯著贏誰 · {{ targetInfo.name }} · {{ metricInfo.name }}</span></h2>
        <p class="desc">對每一對系統，用 50 首歌的配對分數做 Wilcoxon signed-rank 檢定。「A 顯著優於 B」畫成由左往右的邊，再去掉可由遞移推得的邊，剩下的就是必要的優劣關係（{{ dominance.allEdges }} 對顯著關係化簡為 {{ dominance.edges.length }} 條邊）。同一層的系統彼此分不出顯著高下。</p>
        <DominanceGraph :nodes="ranking" :edges="dominance.edges" :layers="dominance.layers" :win="dominance.win"
          :selected="selected" :metric="metric" :alpha="ALPHA" :dark="dark" @select="toggleSelect" />
      </figure>

      <figure>
        <h2>圖 4　曲目熱圖 <span class="hint">系統 × 曲目 · 由難到易 · 點曲目看逐秒</span></h2>
        <p class="desc">列依排行、欄依「難度」排序（非 oracle 系統在該曲的中位數）。整欄都淺色的歌對所有系統都難；一列整體偏深的系統全面較強。最左邊兩首 PR 的歌是純演奏曲，沒有人聲可分，vocals 的分數因此極低。</p>
        <TrackHeatmap :systems="ranking" :tracks="tracksSorted" :value="valueAt" :selected="selected" :selected-track="selectedTrack"
          :color="color" :metric="metric" @select="toggleSelect" @select-track="selectTrack" />
      </figure>

      <figure>
        <h2>圖 5　逐秒時間線
          <span class="hint" v-if="selectedTrackInfo">{{ selectedTrackInfo.artist }} – {{ selectedTrackInfo.title }} · {{ targetInfo.name }} · {{ metricInfo.name }} · 拖曳放大，雙擊還原</span>
          <span class="hint" v-else>在圖 4 點一首歌</span>
        </h2>
        <p class="desc">畫出{{ selected.size ? '已選取的系統' : '排行前 5 的系統' }}在這首歌上每一秒的分數。前奏沒有目標聲源時分數會崩到負值，副歌進來時誰恢復得快一目了然。線的顏色是系統類別，線尾標系統名。</p>
        <Timeline v-if="selectedTrack && frames" :frames="frames" :systems="timelineSystems" :target="target" :metric="metric"
          :track-label="selectedTrackInfo ? selectedTrackInfo.title : ''" />
        <p v-else-if="selectedTrack" class="status">載入逐秒資料中…</p>
        <p v-else class="placeholder-text">尚未選取曲目。</p>
      </figure>

      <section class="panel">
        <h2>可以試試看</h2>
        <ul class="tips">
          <li><strong>關掉 Oracle</strong>：排行榜和優勢圖重排，真正的參賽系統裡 Sony 的 TAK2、TAU1 在人聲領先；它們在方法地圖上也是同一群。</li>
          <li><strong>點 vocals 以外的欄</strong>：切到 bass 或 other，排名大洗牌，深度學習系統的優勢縮小。</li>
          <li><strong>看方法地圖的連線</strong>：nussl 工具箱的六個盲分離方法由同一位實作者提交、行為也最相似；中研院的 JY1 到 JY3 幾乎疊在同一點。</li>
          <li><strong>切到勝率矩陣</strong>：依排行排序後幾乎是整齊的上三角，表示優劣關係接近遞移；少數灰格就是分不出高下的對手，例如 TAK2 對 TAU1 剛好五五波。</li>
          <li><strong>切指標到 SIR 再到 SAR</strong>：有些系統 SIR 高（分得乾淨）但 SAR 低（瑕疵多），排名位置互換，這就是干擾與瑕疵的取捨。</li>
          <li><strong>點熱圖最左邊的歌</strong>：看逐秒線在整首歌都貼著底部；再點最右邊的歌比較。</li>
        </ul>
      </section>

      <section class="panel">
        <h2>背後的方法（摘要，完整說明見 README）</h2>
        <ul class="math">
          <li><strong>指標</strong>：BSS Eval 把估計出的 stem 拆成 目標 + 空間失真 + 干擾 + 瑕疵 四個正交分量，SDR、ISR、SIR、SAR 分別是目標對（全部錯誤 / 空間失真 / 干擾 / 瑕疵）的能量比，單位 dB。</li>
          <li><strong>彙整</strong>：每首歌對逐秒分數取中位數，系統排名再對 50 首歌取中位數（與 SiSEC 官方一致）；靜音秒數的 NaN 與 ±∞ 忽略。</li>
          <li><strong>相似度</strong>：兩系統剖面的 Pearson 相關 r，距離 d = √(2(1 − r))；古典 MDS 對距離矩陣做雙中心化後取前兩個特徵向量當座標。</li>
          <li><strong>顯著優於</strong>：Wilcoxon signed-rank 檢定（配對 = 同一首歌），常態近似，雙尾 p &lt; 0.05 且勝率 &gt; 0.5。遞移化簡保留無法由其他路徑推得的邊；分層 = 最長路徑長度。</li>
          <li><strong>難度</strong>：非 oracle 系統在該曲分數的中位數；熱圖的列與欄分別依排行與難度排序（seriation）。</li>
        </ul>
      </section>

      <details class="panel">
        <summary>資料表：31 個系統的屬性（作者、機構、程式）</summary>
        <table>
          <thead><tr><th>系統</th><th>類別</th><th>blind</th><th>額外資料</th><th>{{ metricInfo.name }} 中位數</th><th>作者</th><th>機構</th><th>程式</th></tr></thead>
          <tbody>
            <tr v-for="r in leaderboardRows" :key="r.system" :class="{ selected: selected.has(r.system) }" @click="toggleSelect(r.system)">
              <td><span class="swatch" :style="{ background: CATEGORY[r.category].color }"></span>{{ r.system }}</td>
              <td>{{ CATEGORY[r.category].short }}</td>
              <td>{{ data.systemInfo.get(r.system).is_blind }}</td>
              <td>{{ data.systemInfo.get(r.system).extra_data }}</td>
              <td class="num">{{ fmt(r.median) }}</td>
              <td class="small">{{ data.systemInfo.get(r.system).authors }}</td>
              <td class="small">{{ data.systemInfo.get(r.system).affiliation }}</td>
              <td class="small"><a v-if="data.systemInfo.get(r.system).code" :href="data.systemInfo.get(r.system).code" target="_blank" rel="noopener">連結</a></td>
            </tr>
          </tbody>
        </table>
      </details>
    </template>

    <footer>
      執行方式：<code>npm install</code> 之後 <code>npm run dev</code>。資料由 <code>python3 -I prepare_data.py &lt;sigsep-mus-2018 路徑&gt;</code> 彙整到 <code>public/data/</code>。
      <code>npm run check</code> 驗證統計函式。
    </footer>

    <div v-if="tip.show" class="tooltip" :style="{ left: tip.x + 'px', top: tip.y + 'px' }" v-html="tip.html"></div>
  </main>
</template>
