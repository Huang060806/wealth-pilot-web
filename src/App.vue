<template>
  <div class="page">
    <header>
      <h1>WealthPilot 资产规划师</h1>
      <span class="tag">AI Agent 驱动 · 数据由确定性引擎计算</span>
    </header>
    <div class="main">
      <!-- 左侧：对话 -->
      <section class="chat">
        <div class="messages" ref="msgBox">
          <div v-for="(m, i) in messages" :key="i" :class="['msg', m.role]">
            <div class="bubble">{{ m.content }}</div>
          </div>
          <div v-if="thinking" class="msg assistant"><div class="bubble thinking">思考中…</div></div>
        </div>
        <div class="input-bar">
          <input v-model="input" placeholder="描述你的资产情况，例如：我28岁，有10万存款…" @keyup.enter="send" />
          <button :disabled="thinking || !input.trim()" @click="send">发送</button>
        </div>
      </section>

      <!-- 右侧：档案 + 图表 -->
      <section class="panel">
        <div class="card">
          <h3>资产档案</h3>
          <div class="profile-grid">
            <div v-for="(v, k) in profileView" :key="k"><span>{{ k }}</span><b>{{ v }}</b></div>
          </div>
        </div>

        <div class="card" v-if="plan">
          <h3>{{ planYears }} 年收益推演（万元）</h3>
          <div ref="chartRef" class="chart"></div>
          <div class="mc">
            蒙特卡洛 {{ plan.mcRuns }} 次模拟：中位数 <b>{{ plan.mcMedian }}</b> 万 ·
            较差（10%）{{ plan.mcP10 }} 万 · 较好（90%）{{ plan.mcP90 }} 万
          </div>
          <h3>逐资产收益分析</h3>
          <table class="asset-table">
            <thead><tr><th>资产</th><th>现在(万)</th><th>{{ planYears }}年后(万)</th><th>年化</th></tr></thead>
            <tbody>
              <tr v-for="l in plan.assetLines" :key="l.name">
                <td>{{ l.name }}<div class="src">{{ l.note }}</div></td>
                <td>{{ l.now }}</td>
                <td :style="{ color: l.terminal < 0 ? '#dc2626' : '#16a34a' }">{{ l.terminal }}</td>
                <td>{{ (l.annualReturn * 100).toFixed(1) }}%</td>
              </tr>
            </tbody>
          </table>
          <h3 style="margin-top:14px">配置对比</h3>
          <div v-for="k in ['股票','债券','现金']" :key="k" class="alloc-bar">
            <span class="alloc-name">{{ k }}</span>
            <div class="bars">
              <div class="bar current" :style="{ width: (plan.currentAllocation?.[k] || 0) * 100 + '%' }"></div>
              <div class="bar target" :style="{ width: (plan.allocation?.[k] || 0) * 100 + '%' }"></div>
            </div>
            <span class="alloc-num">{{ Math.round((plan.currentAllocation?.[k] || 0) * 100) }}% → {{ Math.round((plan.allocation?.[k] || 0) * 100) }}%</span>
          </div>
          <div class="legend"><span class="dot current"></span>当前 <span class="dot target"></span>建议</div>
          <button class="export" @click="exportReport">导出规划报告（PNG）</button>
        </div>
        <div class="card empty" v-else>完成对话后，这里会展示你的收益曲线</div>

        <div class="card" v-if="market">
          <h3>行情基准参数</h3>
          <div v-for="(v, k) in market" :key="k" class="market-row">
            <div><b>{{ k }}</b><span class="src">{{ v.source }}</span></div>
            <div>年化 {{ (v.annualReturn * 100).toFixed(1) }}% · 波动 {{ (v.volatility * 100).toFixed(1) }}%</div>
          </div>
        </div>
      </section>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, onMounted } from 'vue'
import axios from 'axios'
import * as echarts from 'echarts'

const sessionId = 'web-' + Date.now()
const messages = ref([{ role: 'assistant', content: '你好，我是你的资产规划师。先聊聊你的情况吧：年龄、手里的存款/投资、每月收支、以及你的财务目标？' }])
const input = ref('')
const thinking = ref(false)
const profile = ref({})
const plan = ref(null)
const market = ref(null)
const msgBox = ref()
const chartRef = ref()
let chart = null

const profileView = computed(() => {
  const p = profile.value
  const items = {}
  if (p.age) items['年龄'] = p.age + ' 岁'
  if (p.cash) items['现金'] = p.cash + ' 万'
  if (p.deposit) items['定存'] = p.deposit + ' 万'
  if (p.funds) items['基金'] = p.funds + ' 万'
  if (p.stocks) items['股票'] = p.stocks + ' 万'
  if (p.realEstate) items['房产'] = p.realEstate + ' 万'
  if (p.debts) items['负债'] = p.debts + ' 万'
  if (p.monthlyIncome) items['月收入'] = p.monthlyIncome + ' 元'
  if (p.monthlyExpense) items['月支出'] = p.monthlyExpense + ' 元'
  if (p.goal) items['目标'] = p.goal
  if (p.years) items['规划年限'] = p.years + ' 年'
  if (p.riskLevel) items['风险偏好'] = { conservative: '保守', balanced: '稳健', aggressive: '激进' }[p.riskLevel]
  return items
})

const planYears = computed(() => plan.value ? plan.value.years.length - 1 : 0)

const send = async () => {
  const text = input.value.trim()
  if (!text) return
  messages.value.push({ role: 'user', content: text })
  input.value = ''
  thinking.value = true
  scroll()
  try {
    const { data } = await axios.post('/api/chat', { sessionId, message: text })
    messages.value.push({ role: 'assistant', content: data.reply })
    const [p, pl] = await Promise.all([
      axios.get(`/api/profile/${sessionId}`),
      axios.get(`/api/plan/${sessionId}`).catch(() => null)
    ])
    profile.value = p.data
    if (pl && pl.data) {
      plan.value = pl.data
      await nextTick()
      renderChart()
    }
  } catch (e) {
    messages.value.push({ role: 'assistant', content: '服务暂时出问题了，请稍后再试。' })
  } finally {
    thinking.value = false
    scroll()
  }
}

const renderChart = () => {
  if (!chart) chart = echarts.init(chartRef.value)
  chart.setOption({
    tooltip: { trigger: 'axis' },
    legend: { data: ['悲观', '中性', '乐观'] },
    grid: { left: 50, right: 20, top: 40, bottom: 30 },
    xAxis: { type: 'category', data: plan.value.years, name: '年' },
    yAxis: { type: 'value', name: '万元' },
    series: [
      { name: '悲观', type: 'line', data: plan.value.pessimistic, lineStyle: { type: 'dashed' }, itemStyle: { color: '#9ca3af' } },
      { name: '中性', type: 'line', data: plan.value.expected, itemStyle: { color: '#2563eb' }, areaStyle: { opacity: 0.08 } },
      { name: '乐观', type: 'line', data: plan.value.optimistic, lineStyle: { type: 'dashed' }, itemStyle: { color: '#16a34a' } }
    ]
  })
}

const exportReport = () => {
  // 报告 = 图表 PNG + 文字摘要，用浏览器下载
  const img = chart.getDataURL({ pixelRatio: 2, backgroundColor: '#fff' })
  const a = document.createElement('a')
  a.href = img
  a.download = `wealthpilot-plan-${new Date().toISOString().slice(0, 10)}.png`
  a.click()
}

const scroll = async () => {
  await nextTick()
  if (msgBox.value) msgBox.value.scrollTop = msgBox.value.scrollHeight
}

onMounted(async () => { scroll(); market.value = (await axios.get('/api/market/params')).data })
</script>

<style>
* { box-sizing: border-box; margin: 0; }
body { font-family: -apple-system, "PingFang SC", sans-serif; background: #f3f4f6; }
.page { max-width: 1200px; margin: 0 auto; padding: 16px; height: 100vh; display: flex; flex-direction: column; }
header { display: flex; align-items: baseline; gap: 12px; padding: 8px 4px 16px; }
h1 { font-size: 20px; }
.tag { font-size: 12px; color: #6b7280; }
.main { display: flex; gap: 16px; flex: 1; min-height: 0; }
.chat { flex: 1; background: #fff; border-radius: 12px; display: flex; flex-direction: column; overflow: hidden; }
.messages { flex: 1; overflow-y: auto; padding: 16px; }
.msg { margin-bottom: 12px; display: flex; }
.msg.user { justify-content: flex-end; }
.bubble { max-width: 75%; padding: 10px 14px; border-radius: 12px; background: #f3f4f6; line-height: 1.6; white-space: pre-wrap; }
.msg.user .bubble { background: #2563eb; color: #fff; }
.thinking { color: #9ca3af; }
.input-bar { display: flex; gap: 8px; padding: 12px; border-top: 1px solid #eee; }
.input-bar input { flex: 1; padding: 10px 12px; border: 1px solid #ddd; border-radius: 8px; outline: none; }
.input-bar button { padding: 10px 20px; background: #2563eb; color: #fff; border: none; border-radius: 8px; cursor: pointer; }
.input-bar button:disabled { background: #93c5fd; cursor: not-allowed; }
.panel { width: 460px; display: flex; flex-direction: column; gap: 16px; overflow-y: auto; }
.card { background: #fff; border-radius: 12px; padding: 16px; }
.card h3 { font-size: 15px; margin-bottom: 12px; }
.card.empty { color: #9ca3af; text-align: center; padding: 40px 16px; }
.profile-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; }
.profile-grid div { display: flex; justify-content: space-between; font-size: 13px; padding: 6px 8px; background: #f9fafb; border-radius: 6px; }
.profile-grid span { color: #6b7280; }
.chart { width: 100%; height: 260px; }
.mc { font-size: 12px; color: #6b7280; margin-top: 8px; line-height: 1.8; }
.alloc { margin-top: 10px; }
.pill { display: inline-block; background: #eff6ff; color: #2563eb; border-radius: 20px; padding: 3px 10px; font-size: 12px; margin-right: 6px; }
.export { margin-top: 12px; width: 100%; padding: 10px; border: 1px solid #2563eb; color: #2563eb; background: #fff; border-radius: 8px; cursor: pointer; }
.market-row { display: flex; justify-content: space-between; align-items: center; font-size: 13px; padding: 8px 0; border-bottom: 1px solid #f3f4f6; }
.market-row:last-child { border-bottom: none; }
.asset-table { width: 100%; border-collapse: collapse; font-size: 13px; }
.asset-table th { text-align: left; color: #6b7280; font-weight: normal; padding: 4px 6px; border-bottom: 1px solid #eee; }
.asset-table td { padding: 8px 6px; border-bottom: 1px solid #f3f4f6; }
.alloc-bar { display: flex; align-items: center; gap: 8px; margin: 8px 0; font-size: 13px; }
.alloc-name { width: 34px; }
.bars { flex: 1; }
.bar { height: 8px; border-radius: 4px; margin: 2px 0; transition: width .5s; }
.bar.current { background: #d1d5db; }
.bar.target { background: #2563eb; }
.alloc-num { width: 84px; text-align: right; color: #6b7280; }
.legend { font-size: 11px; color: #9ca3af; margin-top: 4px; }
.dot { display: inline-block; width: 8px; height: 8px; border-radius: 4px; margin: 0 4px 0 10px; }
.dot.current { background: #d1d5db; } .dot.target { background: #2563eb; }
.src { display: block; font-size: 11px; color: #9ca3af; }
</style>
