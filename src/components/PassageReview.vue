<script setup>
import { computed, onMounted, ref, watch } from 'vue'

const data = ref(null)
const decisions = ref({})
const reviewer = ref('')
const selectedId = ref('')
const statusFilter = ref('sample')
const sourceFilter = ref('')
const search = ref('')
const note = ref('')
const issues = ref([])
const notice = ref('')
const loading = ref(false)
const issueOptions = ['原文缺失或截断', '段落边界不合理', '切片过短或断句', '切片重叠异常', '版本或适用范围错误', '来源与正文不符', '其他']
const sourceMap = computed(() => Object.fromEntries((data.value?.sources || []).map(s => [s.source_id, s])))
const selected = computed(() => data.value?.passages.find(p => p.id === selectedId.value))
const source = computed(() => sourceMap.value[selected.value?.source_id])
const storageKey = computed(() => data.value ? `echomind-passage-review:${data.value.snapshot}` : '')

function hash(value) {
  let h = 2166136261
  for (let i = 0; i < value.length; i++) h = Math.imul(h ^ value.charCodeAt(i), 16777619)
  return h >>> 0
}
const sampleIds = computed(() => {
  const ids = new Set()
  for (const src of data.value?.sources || []) {
    const sourcePassages = data.value.passages.filter(p => p.source_id === src.source_id)
    const count = Math.min(sourcePassages.length, Math.max(3, Math.ceil(sourcePassages.length * 0.3)))
    sourcePassages.sort((a, b) => hash(`passage-review-v1:${a.id}`) - hash(`passage-review-v1:${b.id}`))
      .slice(0, count).forEach(p => ids.add(p.id))
  }
  return ids
})
const counts = computed(() => {
  const all = data.value?.passages || []
  const sampled = all.filter(p => sampleIds.value.has(p.id))
  const done = sampled.filter(p => ['pass', 'fail'].includes(decisions.value[p.id]?.verdict))
  const passed = done.filter(p => decisions.value[p.id].verdict === 'pass').length
  return { total: all.length, sample: sampled.length, done: done.length, passed,
    required: Math.ceil(sampled.length * 0.9), allowed: sampled.length > 0 && done.length === sampled.length && passed >= Math.ceil(sampled.length * 0.9) }
})
const filtered = computed(() => (data.value?.passages || []).filter(p => {
  if (sourceFilter.value && p.source_id !== sourceFilter.value) return false
  if (statusFilter.value === 'sample' && !sampleIds.value.has(p.id)) return false
  if (statusFilter.value === 'pending' && decisions.value[p.id]?.verdict) return false
  if (['pass', 'fail'].includes(statusFilter.value) && decisions.value[p.id]?.verdict !== statusFilter.value) return false
  return !search.value || `${p.id} ${p.text} ${p.chunks.map(c => c.text).join(' ')}`.toLowerCase().includes(search.value.toLowerCase())
}))
const highlightedSource = computed(() => {
  const original = source.value?.text || ''
  const target = selected.value?.text || ''
  const start = original.indexOf(target)
  return start < 0 ? { before: original, match: '', after: '' } : {
    before: original.slice(0, start), match: target, after: original.slice(start + target.length)
  }
})

function choose(p) {
  selectedId.value = p.id
  note.value = decisions.value[p.id]?.note || ''
  issues.value = [...(decisions.value[p.id]?.issues || [])]
}
function saveLocal() {
  localStorage.setItem(storageKey.value, JSON.stringify({ snapshot: data.value.snapshot, reviewer: reviewer.value, decisions: decisions.value }))
}
function save(verdict) {
  if (!selected.value) return
  if (!reviewer.value.trim()) { notice.value = '请先填写审核人'; return }
  if (verdict === 'pass' && issues.value.length) { notice.value = '已勾选问题，请清除问题或标记不通过'; return }
  if (verdict === 'fail' && !note.value.trim() && !issues.value.length) { notice.value = '标记不通过时，请至少选择问题或填写备注'; return }
  const previous = decisions.value[selectedId.value]
  const record = { passage_id: selectedId.value, source_id: selected.value.source_id,
    verdict, issues: [...issues.value], note: note.value.trim(), reviewer: reviewer.value.trim(),
    reviewed_at: new Date().toISOString(), snapshot: data.value.snapshot,
    audit: [...(previous?.audit || []), { verdict, reviewer: reviewer.value.trim(), at: new Date().toISOString() }] }
  decisions.value = { ...decisions.value, [selectedId.value]: record }
  try { saveLocal() } catch (error) { notice.value = `浏览器保存失败：${error.message}；请立即导出备份`; return }
  notice.value = '已保存到本机浏览器；完成后请导出审核记录。'
  const position = filtered.value.findIndex(p => p.id === selectedId.value)
  const next = filtered.value.slice(position + 1).find(p => !decisions.value[p.id]?.verdict)
    || filtered.value.find(p => !decisions.value[p.id]?.verdict)
  if (next) choose(next)
}
function exportDecisions() {
  const rows = Object.values(decisions.value)
  if (!rows.length) { notice.value = '尚无审核记录'; return }
  const payload = { schema_version: 1, collection: data.value.collection, snapshot: data.value.snapshot,
    sample_ids: [...sampleIds.value].sort(), gate: counts.value, exported_at: new Date().toISOString(), decisions: rows }
  const url = URL.createObjectURL(new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json' }))
  const link = document.createElement('a')
  link.href = url
  link.download = `passage-review-${data.value.snapshot.slice(0, 12)}.json`
  link.click()
  setTimeout(() => URL.revokeObjectURL(url), 1000)
}
async function importDecisions(event) {
  try {
    const file = event.target.files?.[0]
    if (!file) return
    const payload = JSON.parse(await file.text())
    if (payload.snapshot !== data.value.snapshot) throw Error('审核文件与当前知识快照不匹配')
    if (!Array.isArray(payload.decisions)) throw Error('审核文件格式错误')
    const validIds = new Set(data.value.passages.map(p => p.id))
    const restored = {}
    for (const item of payload.decisions) {
      if (!validIds.has(item.passage_id) || !['pass', 'fail'].includes(item.verdict) || !item.reviewer || restored[item.passage_id]) {
        throw Error(`审核记录无效或重复：${item.passage_id}`)
      }
      restored[item.passage_id] = item
    }
    decisions.value = restored
    reviewer.value = payload.decisions[0]?.reviewer || reviewer.value
    saveLocal()
    if (selected.value) choose(selected.value)
    notice.value = `已恢复 ${payload.decisions.length} 条审核记录`
  } catch (error) { notice.value = error.message }
  event.target.value = ''
}
async function load() {
  loading.value = true
  try {
    const response = await fetch(`${import.meta.env.BASE_URL}passage-review.json`, { cache: 'no-store' })
    if (!response.ok) throw Error('未找到段落快照；请先运行 python -m evaluation.dataset.export_passage_review')
    const payload = await response.json()
    if (payload.schema_version !== 1 || !Array.isArray(payload.passages) || !Array.isArray(payload.sources)) throw Error('段落快照格式错误')
    data.value = payload
    const saved = JSON.parse(localStorage.getItem(storageKey.value) || 'null')
    decisions.value = saved?.snapshot === payload.snapshot ? saved.decisions || {} : {}
    reviewer.value = saved?.reviewer || ''
    choose(payload.passages.find(p => sampleIds.value.has(p.id) && !decisions.value[p.id]?.verdict) || payload.passages[0])
    notice.value = `已载入 ${payload.sources.length} 份原文、${payload.passages.length} 个段落；审核只保存在浏览器，需导出备份。`
  } catch (error) { notice.value = error.message }
  finally { loading.value = false }
}
watch(reviewer, () => { if (data.value) { try { saveLocal() } catch { /* Save button reports storage errors. */ } } })
onMounted(load)
</script>

<template>
  <section class="passage-review">
    <header>
      <div><h1>原文段落审核</h1><p>核对官方来源、段落边界和实际入库切片。审核不会修改原文或索引。</p></div>
      <button type="button" @click="exportDecisions">导出审核记录</button>
    </header>
    <p class="notice" role="status">{{ notice }}</p>
    <div v-if="data" class="summary">
      <strong>{{ data.sources.length }} 份原文 · {{ counts.total }} 个段落 · {{ data.passages.reduce((n, p) => n + p.chunks.length, 0) }} 个切片</strong>
      <span>固定分层抽检 {{ counts.done }}/{{ counts.sample }}，原始合格 {{ counts.passed }}；90% 门槛至少 {{ counts.required }} 条。</span>
      <span :class="counts.allowed ? 'good' : 'waiting'">{{ counts.allowed ? '抽检达到放行门槛' : '尚未达到放行门槛' }}</span>
      <small>抽样按来源和段落 ID 固定，每份至少 3 条；通过门槛仍需检查未抽样段落的风险。</small>
    </div>
    <div v-if="data" class="toolbar">
      <label>审核人 <input v-model="reviewer" placeholder="姓名或代号" /></label>
      <label>来源 <select v-model="sourceFilter"><option value="">全部来源</option><option v-for="item in data.sources" :key="item.source_id" :value="item.source_id">{{ item.source_id }}</option></select></label>
      <label>状态 <select v-model="statusFilter"><option value="sample">固定抽检</option><option value="pending">全部待审核</option><option value="pass">已通过</option><option value="fail">不通过</option><option value="all">全部</option></select></label>
      <label>搜索 <input v-model="search" placeholder="段落 ID 或正文" /></label>
      <label class="import-button">导入审核备份<input type="file" accept=".json" @change="importDecisions" /></label>
      <button type="button" @click="load" :disabled="loading">刷新快照</button>
    </div>
    <div v-if="data" class="review-grid">
      <aside class="passage-list"><p>匹配 {{ filtered.length }} 条</p><button v-for="p in filtered" :key="p.id" type="button" :class="{ active: selectedId === p.id }" @click="choose(p)"><small>{{ p.id }} · {{ sampleIds.has(p.id) ? '抽样' : '非抽样' }} · {{ decisions[p.id]?.verdict || '待审核' }}</small><span>{{ p.text.slice(0, 105) }}</span></button><p v-if="!filtered.length">没有匹配的段落。</p></aside>
      <div v-if="selected" class="review-detail">
        <div class="source-head"><div><h2>{{ selected.id }}</h2><p>来源：{{ selected.source_id }} · {{ selected.chunks.length }} 个切片 · SHA-256 {{ source.sha256.slice(0, 12) }}…</p></div><a :href="source.url" target="_blank" rel="noopener noreferrer">打开来源网页 ↗</a></div>
        <p class="hint">来源网页可能更新或失效；下方原文来自当前索引快照。请核对适用年份、规则例外和原文是否完整。</p>
        <div class="evidence-grid">
          <section><h3>来源原文 <small>{{ source.characters }} 字</small></h3><pre class="original-text">{{ highlightedSource.before }}<mark v-if="highlightedSource.match">{{ highlightedSource.match }}</mark>{{ highlightedSource.after }}</pre></section>
          <section><h3>待审段落 <small>{{ selected.text.length }} 字</small></h3><pre>{{ selected.text }}</pre><h3>实际入库切片</h3><article v-for="chunk in selected.chunks" :key="chunk.id" class="chunk"><div><strong>#{{ chunk.index + 1 }}</strong><small>{{ chunk.text.length }} 字 · {{ chunk.id.slice(0, 14) }}…</small></div><pre>{{ chunk.text }}</pre></article></section>
        </div>
        <div class="decision-panel"><h3>审核结论</h3><fieldset><legend>发现的问题（可多选）</legend><label v-for="issue in issueOptions" :key="issue"><input v-model="issues" type="checkbox" :value="issue" />{{ issue }}</label></fieldset><label>备注 / 修正建议<textarea v-model="note" rows="3" placeholder="记录有问题的条款、年份、切片边界及建议"></textarea></label><div class="actions"><button type="button" class="pass" @click="save('pass')">通过并下一条</button><button type="button" class="fail" @click="save('fail')">不通过并下一条</button></div><small>重新审核同一段落会保留操作历史。浏览器数据仅为草稿；导出的 JSON 是可追溯审核记录。</small></div>
      </div>
    </div>
  </section>
</template>

<style scoped>
.passage-review{width:100%;height:100%;overflow:auto;padding:24px;color:#172033}.passage-review header,.source-head{display:flex;align-items:center;justify-content:space-between;gap:20px}.passage-review h1{font-size:25px;margin:0 0 8px}.passage-review h2{font-size:20px;margin:0 0 6px}.passage-review h3{font-size:16px;margin:0 0 10px}.passage-review p{line-height:1.5;margin:6px 0}.passage-review small,.hint{color:#607086}.passage-review button,.import-button{border:1px solid #cbd5e1;border-radius:8px;background:#fff;color:#172033;padding:9px 13px;cursor:pointer;text-align:left}.passage-review button:hover{background:#eff6ff}.passage-review button:disabled{opacity:.5}.passage-review input,.passage-review select,.passage-review textarea{border:1px solid #cbd5e1;border-radius:7px;padding:8px;color:#172033;background:white;font:inherit}.passage-review textarea{width:100%;resize:vertical;display:block}.passage-review input[type=checkbox]{accent-color:#2563eb}.notice{background:#eff6ff;border:1px solid #bfdbfe;border-radius:8px;padding:10px 14px;overflow-wrap:anywhere}.summary{display:flex;flex-wrap:wrap;align-items:center;gap:12px 18px;padding:14px;background:#f8fafc;border:1px solid #e2e8f0;border-radius:9px}.summary small{width:100%}.good{color:#167143}.waiting{color:#9a5416}.toolbar{display:flex;flex-wrap:wrap;gap:10px;align-items:end;margin:16px 0}.toolbar label:not(.import-button){display:flex;flex-direction:column;gap:4px;font-size:13px}.toolbar input{min-width:160px}.import-button input{display:none}.review-grid{display:grid;grid-template-columns:245px minmax(0,1fr);gap:18px;min-height:0}.passage-list{max-height:70vh;overflow:auto;position:sticky;top:12px;border:1px solid #e2e8f0;border-radius:9px;padding:10px}.passage-list>button{display:block;width:100%;margin:7px 0;white-space:normal}.passage-list>button.active{background:#dbeafe;border-color:#3b82f6}.passage-list span,.passage-list small{display:block;overflow-wrap:anywhere}.passage-list span{margin-top:5px}.source-head a{color:#1d4ed8;white-space:nowrap}.evidence-grid{display:grid;grid-template-columns:minmax(0,1fr) minmax(0,1fr);gap:14px;margin:15px 0}.evidence-grid section,.decision-panel{border:1px solid #dbe3ee;border-radius:9px;padding:15px;min-width:0}.evidence-grid pre{white-space:pre-wrap;overflow-wrap:anywhere;font-family:inherit;line-height:1.75;margin:0}.original-text{max-height:68vh;overflow:auto}.evidence-grid mark{background:#fde68a;padding:2px}.chunk{border-top:1px solid #e2e8f0;padding:12px 0}.chunk div{display:flex;justify-content:space-between;gap:10px}.decision-panel fieldset{display:flex;flex-wrap:wrap;gap:9px 16px;border:1px solid #e2e8f0;border-radius:8px;margin:10px 0 15px}.decision-panel label{display:block}.decision-panel fieldset label{display:inline-flex;align-items:center;gap:5px}.actions{display:flex;gap:9px;margin:14px 0}.actions .pass{background:#166534;color:white;border-color:#166534}.actions .fail{background:#991b1b;color:white;border-color:#991b1b}.actions .pass:hover{background:#14532d}.actions .fail:hover{background:#7f1d1d}@media(max-width:1000px){.review-grid{grid-template-columns:1fr}.passage-list{position:static;max-height:220px}.evidence-grid{grid-template-columns:1fr}.passage-review{padding:14px}}@media(max-width:580px){.passage-review header,.source-head{align-items:flex-start;flex-direction:column}}
</style>
