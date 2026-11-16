<script setup>
import { ref, computed, onMounted, watch } from 'vue'
const intents=['query','complaint','request','greeting','escalation','technical','billing','account','feedback','order_status','logistics','refund','invoice','payment_issue','account_security','technical_login','technical_crash','human_handoff','other']
const agents=['general','technical','billing','escalation']
const tags=['negation','quotation','context','typo','colloquial','composite','entity_noise','question','handoff','out_of_scope']
const rows=ref([]), status=ref('pending'), query=ref(''), tag=ref(''), reviewer=ref(''), selected=ref(''), draft=ref(null), historyText=ref('[]'), notice=ref(''), busy=ref(false)
const sampleOnly=ref(true)
const gate=computed(()=>{const sample=rows.value.filter(r=>r.sample_required);const reviewed=sample.filter(r=>r.review_status!=='pending'&&r.reviewer&&['pass','fail'].includes(r.quality_verdict));const passed=reviewed.filter(r=>r.quality_verdict==='pass').length;const required=Math.ceil(sample.length*.90);return {total:sample.length,reviewed:reviewed.length,passed,required,allowed:sample.length>0&&reviewed.length===sample.length&&passed>=required}})
const counts=computed(()=>Object.fromEntries(['pending','confirmed','rejected'].map(s=>[s,rows.value.filter(r=>r.review_status===s).length])))
const filtered=computed(()=>rows.value.filter(r=>(!sampleOnly.value||r.sample_required)&&(!status.value||r.review_status===status.value)&&(!tag.value||r.tags.includes(tag.value))&&(!query.value||JSON.stringify([r.message,r.history,r.id]).includes(query.value))))
const list=computed(()=>filtered.value.slice(0,150))
const clone=x=>JSON.parse(JSON.stringify(x))
const controlFields=['intent','primary','supporting','tags','expected_action']
const extraFields=[...controlFields,'simulated_time','topic','active_intents','acceptable_primary','known_slots','missing_slots','clause_ids','answerable','answer_points','forbidden_claims','required_skills','required_tools']
const extraText=ref('{}')
watch(()=>controlFields.map(key=>JSON.stringify(draft.value?.[key])),()=>{
 if(!draft.value)return
 try{
  const fields=JSON.parse(extraText.value)
  if(!fields||Array.isArray(fields)||typeof fields!=='object')return
  for(const key of controlFields)if(key in draft.value)fields[key]=clone(draft.value[key])
  if(Array.isArray(fields.active_intents)&&!fields.active_intents.includes(draft.value.intent)){
   fields.active_intents=[draft.value.intent,...fields.active_intents.slice(1).filter(i=>i!==draft.value.intent)]
  }
  extraText.value=JSON.stringify(fields,null,2)
 }catch{ /* Keep invalid in-progress JSON intact so the reviewer can fix it. */ }
},{flush:'sync'})
function applyJsonToControls(){
 try{
  const fields=JSON.parse(extraText.value)
  if(!fields||Array.isArray(fields)||typeof fields!=='object')return
  for(const key of controlFields)if(key in fields)draft.value[key]=clone(fields[key])
 }catch{ /* The preview and save action report the parse error. */ }
}
function editedRow(){
 const row=clone(draft.value)
 row.history=JSON.parse(historyText.value)
 const extra=JSON.parse(extraText.value)
 if(!Array.isArray(row.history))throw Error('对话历史必须是 JSON 数组')
 if(!extra||Array.isArray(extra)||typeof extra!=='object')throw Error('其他业务字段必须是 JSON 对象')
 for(const key of extraFields){if(key in extra)row[key]=extra[key]}
 return row
}
const preview=computed(()=>{
 if(!draft.value)return {text:'',error:''}
 try{
  const {original,audit,...row}=editedRow()
  return {text:JSON.stringify(row,null,2),error:''}
 }catch(e){return {text:'',error:`当前 JSON 无法预览：${e.message}`}}
})

async function database(){return new Promise((resolve,reject)=>{const r=indexedDB.open('echomind-dataset-review',1);r.onupgradeneeded=()=>r.result.createObjectStore('datasets');r.onsuccess=()=>resolve(r.result);r.onerror=()=>reject(r.error)})}
async function storage(write=false){const db=await database();try{return await new Promise((resolve,reject)=>{const tx=db.transaction('datasets',write?'readwrite':'readonly');const request=write?tx.objectStore('datasets').put(clone(rows.value),'active'):tx.objectStore('datasets').get('active');let result;request.onsuccess=()=>result=request.result;tx.oncomplete=()=>resolve(result);tx.onerror=()=>reject(tx.error);tx.onabort=()=>reject(tx.error)})}finally{db.close()}}
function choose(row){if(!row){selected.value="";draft.value=null;return}selected.value=row.id;draft.value=clone(row);historyText.value=JSON.stringify(row.history,null,2);extraText.value=JSON.stringify(Object.fromEntries(extraFields.filter(k=>k in row).map(k=>[k,row[k]])),null,2)}
function validate(row){
 if(!row.id||typeof row.message!=='string'||!row.message.trim()||!intents.includes(row.intent)||!agents.includes(row.primary))throw Error('消息或意图、主角色不合法')
 if(!Array.isArray(row.supporting)||row.supporting.some(a=>!agents.includes(a)||a===row.primary)||new Set(row.supporting).size!==row.supporting.length)throw Error('辅助角色不能重复或包含主角色')
 if(!Array.isArray(row.tags)||row.tags.some(t=>!tags.includes(t)))throw Error('请选择合法场景标签')
 if(!Array.isArray(row.history)||row.history.some(m=>!['user','assistant'].includes(m.role)||typeof m.content!=='string'))throw Error('历史必须是role/content对象数组')
 if(row.evaluation_scope!=='routing_only'&&!['answer','clarify','handoff','decline','state_unknown'].includes(row.expected_action))throw Error('处理动作不合法')
}
async function load(text){const parsed=text.split(/\r?\n/).filter(s=>s.trim()).map(s=>JSON.parse(s));if(!parsed.length)throw Error('文件中没有数据');const restore=parsed.some(r=>r.audit?.length)&&confirm('文件包含人工审核记录，是否恢复审核进度？取消则全部作为待审核候选导入。');const ids=new Set(),messages=new Set();for(const r of parsed){validate(r);if(ids.has(r.id))throw Error('重复ID：'+r.id);ids.add(r.id);const key=JSON.stringify([r.message.trim(),r.history]);if(messages.has(key))throw Error('重复消息与历史');messages.add(key);if(!restore){r.review_status='pending';r.label_status='llm_silver_needs_human_review'}else if(!['pending','confirmed','rejected'].includes(r.review_status)||(r.review_status!=='pending'&&!r.reviewer))throw Error('备份审核记录不完整');r.original=clone(r.original||{message:r.message,history:r.history,intent:r.intent,primary:r.primary,supporting:r.supporting,tags:r.tags});if(!restore)r.audit=[]}if(rows.value.some(r=>r.audit?.length)&&!confirm('载入新数据会替换浏览器当前工作集，请先导出备份。继续？'))return;rows.value=parsed;await storage(true);choose(filtered.value[0]||parsed[0]);notice.value=`已载入 ${parsed.length} 条，${restore?'已恢复审核进度':'全部待人工审核'}`}
async function serverLoad(){busy.value=true;notice.value='正在载入生成数据…';try{const response=await fetch(import.meta.env.BASE_URL+'routing-candidates.jsonl',{cache:'no-store',signal:AbortSignal.timeout(15000)});if(!response.ok)throw Error('数据尚未生成完成，请稍后刷新或导入JSONL');const text=await response.text();if(text.trimStart().startsWith('<'))throw Error('未找到生成数据文件，请检查 public/routing-candidates.jsonl');await load(text)}catch(e){notice.value=e.message}finally{busy.value=false}}
async function importFile(event){try{const file=event.target.files?.[0];if(file)await load(await file.text())}catch(e){notice.value=e.message}event.target.value=''}
async function save(nextStatus){if(!draft.value)return;busy.value=true;try{const row=editedRow();validate(row);if(nextStatus!=='pending'&&!reviewer.value.trim())throw Error('确认或拒绝前请填写审核人');const changed=Object.keys(row.original||{}).some(k=>JSON.stringify(row[k])!==JSON.stringify(row.original[k]));if(nextStatus!=='pending')row.quality_verdict=nextStatus==='rejected'||Boolean(row.generation_correction)||changed?'fail':'pass';if(nextStatus==='rejected'&&!row.review_note?.trim())throw Error('拒绝时请填写原因');row.review_status=nextStatus;row.label_status=nextStatus==='confirmed'?'human_reviewed':'llm_silver_needs_human_review';row.reviewer=reviewer.value.trim();row.reviewed_at=new Date().toISOString();row.audit=[...(row.audit||[]),{reviewer:row.reviewer,status:nextStatus,quality_verdict:row.quality_verdict,time:row.reviewed_at,note:row.review_note||''}];const index=rows.value.findIndex(r=>r.id===row.id);const previous=rows.value[index];rows.value[index]=row;try{await storage(true)}catch(e){rows.value[index]=previous;throw e}notice.value='已保存到本机浏览器；请定期导出备份';const next=filtered.value.find(r=>r.id!==row.id);choose(next||row)}catch(e){notice.value=e.message}finally{busy.value=false}}
function exportRows(confirmed){const data=confirmed?rows.value.filter(r=>r.review_status==='confirmed'):rows.value;if(!data.length){notice.value='没有可导出的数据';return}const blob=new Blob([data.map(r=>JSON.stringify(r)).join('\n')+'\n'],{type:'application/x-ndjson;charset=utf-8'});const url=URL.createObjectURL(blob);const a=document.createElement('a');a.href=url;a.download=confirmed?'routing-human-reviewed.jsonl':'routing-review-backup.jsonl';a.click();setTimeout(()=>URL.revokeObjectURL(url),1000)}
onMounted(async()=>{try{const saved=await storage();if(saved?.length){rows.value=saved;choose(saved.find(r=>r.review_status==='pending')||saved[0]);notice.value=`已恢复本机 ${saved[0]?.stage_id||'未知批次'} 的审核进度；若要切换批次，请先导出备份，再点击“载入生成数据”`}else await serverLoad()}catch(e){notice.value=e.message}})
</script>
<template>
<section class="review-page" :aria-busy="busy">
 <header><h1>路由数据审核</h1><p>模型标签是候选答案。请独立判断后修改、确认；审核进度保存在当前浏览器，导出后才有文件备份。</p></header>
 <div class="review-tools"><strong>共 {{ rows.length }} 条 · 待审核 {{ counts.pending }} · 已确认 {{ counts.confirmed }} · 已拒绝 {{ counts.rejected }}</strong><button @click="serverLoad" :disabled="busy">{{ busy ? '处理中…' : '载入生成数据' }}</button><label class="file-button">导入 JSONL<input type="file" accept=".jsonl,.ndjson" @change="importFile"></label><button @click="exportRows(false)">导出全部备份</button><button @click="exportRows(true)">导出已确认数据</button></div>
  <div v-if="gate.total"><strong>{{ rows[0]?.stage_id }}：{{ rows.length }} 条，固定抽检 {{ gate.total }} 条，通过门槛90%</strong><p>已抽检 {{ gate.reviewed }}/{{ gate.total }}，原始合格 {{ gate.passed }} 条；至少需要 {{ gate.required }} 条合格。</p><p>{{ gate.allowed ? '抽检门槛通过。' : '尚未放行：完成固定抽检；未达标则分析问题、调整提示词并重新验证。' }} 本页不自动调用付费模型。</p><label><input type="checkbox" v-model="sampleOnly">仅看固定抽检样本</label></div><p role="status">{{ notice }}</p>
 <div class="review-tools"><input v-model="reviewer" placeholder="审核人（确认前必填）"><input v-model="query" placeholder="搜索消息或ID"><select v-model="status"><option value="">全部状态</option><option value="pending">待审核</option><option value="confirmed">已确认</option><option value="rejected">已拒绝</option></select><select v-model="tag"><option value="">全部场景</option><option v-for="t in tags" :key="t">{{ t }}</option></select></div>
 <div class="review-layout"><aside><p>匹配 {{ filtered.length }} 条，展示前150条</p><button v-for="row in list" :key="row.id" :class="{selected:selected===row.id}" @click="choose(row)"><small>{{ row.id }} · {{ row.topic || row.tags?.join('、') }} · {{ row.review_status }}</small><span>{{ row.message }}</span><small v-if="row.validation_warnings?.length">⚠ {{ row.validation_warnings.join('、') }}</small></button></aside>
 <form v-if="draft" @submit.prevent="save('pending')"><h2>{{ draft.id }}</h2><p v-if="draft.generation_correction" role="alert">原始生成存在问题：{{ draft.generation_correction }} 确认或拒绝时会自动记为原始不合格。</p><label>用户消息<textarea v-model="draft.message" rows="3" /></label><label>对话历史（JSON数组）<textarea v-model="historyText" rows="5" /></label><div class="review-tools"><label>意图<select v-model="draft.intent"><option v-for="i in intents" :key="i">{{ i }}</option></select></label><label>主角色<select v-model="draft.primary"><option v-for="a in agents" :key="a">{{ a }}</option></select></label><fieldset class="supporting-field"><legend>辅助角色</legend><label v-for="a in agents" :key="a"><input type="checkbox" v-model="draft.supporting" :value="a" :disabled="a===draft.primary">{{ a }}</label></fieldset><label v-if="draft.evaluation_scope !== 'routing_only'">预期动作<select v-model="draft.expected_action"><option>answer</option><option>clarify</option><option>handoff</option><option>decline</option><option>state_unknown</option></select></label></div><div class="review-tools"><button type="submit" :disabled="busy">保存待审核</button><button type="button" @click="save('confirmed')" :disabled="busy">确认并下一条</button><button type="button" @click="save('rejected')" :disabled="busy">拒绝并下一条</button></div><label>审核备注（拒绝时必填）<textarea v-model="draft.review_note" rows="2" /></label><details class="review-extra"><summary>更多标注</summary><fieldset><legend>场景标签</legend><label v-for="t in tags" :key="t"><input type="checkbox" v-model="draft.tags" :value="t">{{ t }}</label></fieldset><label>语义场景／分组依据<input v-model="draft.scenario"></label><label>分组ID（相近改写应使用相同ID）<input v-model="draft.group_id"></label><details><summary>查看模型原始标注（只读，修改后仍保留供质量审核）</summary><p>{{ draft.rationale }}</p><pre>{{ draft.original }}</pre></details></details><label>标注 JSON（与上方控件同步，可编辑；手动修改后离开文本框同步到控件）<textarea v-model="extraText" @blur="applyJsonToControls" rows="12" /></label><details class="review-preview" open><summary>修订后标注预览（自动更新，保存时使用）</summary><p v-if="preview.error" role="alert">{{ preview.error }}</p><pre v-else>{{ preview.text }}</pre></details></form><p v-else>等待生成完成，或导入候选JSONL。</p></div>
</section>
</template>
<style scoped>
.review-page{flex:1;min-height:0;overflow-y:auto;overscroll-behavior:contain;width:100%;padding:24px;max-width:1500px;margin:0 auto;color:#172033}.review-page>header{margin-bottom:20px}.review-page p{margin:10px 0;line-height:1.65}.review-page [role=status]{position:sticky;top:0;z-index:2;background:#eff5ff;border:1px solid #ccdaef;border-radius:8px;padding:10px 14px;overflow-wrap:anywhere}.review-page [role=status]:empty{display:none}.review-page header p{color:#607086;line-height:1.7}.review-tools{display:flex;flex-wrap:wrap;gap:10px;align-items:center;margin:14px 0}.review-page button,.file-button{border:1px solid #d7dee8;border-radius:8px;padding:9px 13px;background:white;color:#172033;cursor:pointer;white-space:normal;text-align:center}.review-page button:hover:not(:disabled){background:#eef4ff;color:#172033}.review-page button:disabled{opacity:.55}.review-page input[type=checkbox]{width:auto;min-height:0;padding:0;flex:none;accent-color:#3269c8}.review-tools>input:not([type=checkbox]){width:220px}.review-tools label{margin:0}.review-layout form{min-width:0}.review-page label{line-height:1.6}.review-page select{background:white;color:#172033}.review-layout aside button{align-items:flex-start;justify-content:flex-start;overflow-wrap:anywhere}.review-page input:not([type=checkbox]),.review-page select,.review-page textarea{border:1px solid #cfd8e3;border-radius:7px;padding:9px;font:inherit;max-width:100%;box-sizing:border-box}.review-page textarea{width:100%;resize:vertical}.review-layout{display:grid;grid-template-columns:310px minmax(0,1fr);gap:24px}.review-layout aside{max-height:65vh;overflow-y:auto;position:sticky;top:65px;align-self:start;padding-right:6px}.review-layout aside button{display:flex;flex-direction:column;text-align:left;width:100%;gap:7px;margin-bottom:8px}.review-layout aside button.selected{border-color:#3269c8;background:#edf3ff}.review-layout form>label{display:block;margin:15px 0}.review-layout form>label input{display:block;width:100%}.review-layout fieldset{border:1px solid #dde4ec;border-radius:8px;display:flex;gap:12px;flex-wrap:wrap;margin:15px 0}.review-layout .supporting-field{margin:0;padding:5px 10px;align-items:center;gap:8px 12px}.review-layout .supporting-field label{white-space:nowrap}.review-layout small{color:#63748a}.review-layout pre{white-space:pre-wrap;overflow-wrap:anywhere}.review-extra{border:1px solid #dde4ec;border-radius:8px;padding:12px 16px;margin:15px 0}.review-extra>summary{cursor:pointer;font-weight:600}.review-extra>label{display:block;margin:15px 0}.review-extra>label input{display:block;width:100%}.file-button input{display:none}@media(max-width:800px){.review-layout{grid-template-columns:1fr}.review-layout aside{max-height:260px;position:static}.review-page{padding:14px}}
</style>
