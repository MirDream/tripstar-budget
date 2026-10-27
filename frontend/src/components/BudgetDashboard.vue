<template>
  <a-card :bordered="false" class="bb-card section-shellless">
    <div class="bb-head">
      <div class="bb-head-text">
        <h2 class="bb-title">{{ t('result.budgetBoard.title') }}</h2>
        <p class="bb-sub">{{ t('result.budgetBoard.subtitle') }}</p>
      </div>
      <a-space :size="10" wrap>
        <a-button @click="memberModalOpen = true">
          {{ t('result.budgetBoard.manageMembers') }}
        </a-button>
        <a-button type="primary" :disabled="!members.length" @click="openExpenseModal">
          {{ t('result.budgetBoard.addExpense') }}
        </a-button>
      </a-space>
    </div>

    <!-- KPI -->
    <div class="bb-kpi">
      <div class="bb-kpi-item">
        <span class="bb-kpi-label">{{ t('result.budgetBoard.kpiBudget') }}</span>
        <span class="bb-kpi-value">¥{{ fmt(budgetTotal) }}</span>
      </div>
      <div class="bb-kpi-item">
        <span class="bb-kpi-label">{{ t('result.budgetBoard.kpiActual') }}</span>
        <span class="bb-kpi-value">¥{{ fmt(actualTotal) }}</span>
      </div>
      <div class="bb-kpi-item" :class="{ 'is-over': remain < 0 }">
        <span class="bb-kpi-label">
          {{ remain < 0 ? t('result.budgetBoard.kpiOver') : t('result.budgetBoard.kpiRemain') }}
        </span>
        <span class="bb-kpi-value">¥{{ fmt(Math.abs(remain)) }}</span>
      </div>
      <div class="bb-kpi-progress">
        <div class="bb-progress-head">
          <span>{{ t('result.budgetBoard.usage') }}</span>
          <span class="bb-progress-pct" :class="{ 'is-over': usageRate > 100 }">{{ usageRate.toFixed(1) }}%</span>
        </div>
        <div class="bb-progress-track">
          <i
            class="bb-progress-fill"
            :class="{ 'is-over': usageRate > 100 }"
            :style="{ width: Math.min(usageRate, 100) + '%' }"
          ></i>
        </div>
      </div>
    </div>

    <!-- 预算 vs 实际 + 分类占比 -->
    <div class="bb-grid">
      <section class="bb-panel">
        <header class="bb-panel-head">
          <h3>{{ t('result.budgetBoard.compareTitle') }}</h3>
          <span class="bb-legend">
            <i class="lg lg--budget"></i>{{ t('result.budgetBoard.legendBudget') }}
            <i class="lg lg--actual"></i>{{ t('result.budgetBoard.legendActual') }}
          </span>
        </header>
        <div class="bb-compare">
          <div v-for="row in categoryRows" :key="row.key" class="bb-compare-row">
            <span class="bb-compare-name">{{ t(`result.budget.${row.i18n}`) }}</span>
            <div class="bb-compare-bars">
              <div class="bb-bar bb-bar--budget">
                <i :style="{ width: barWidth(row.budget, maxCategoryAmount) }"></i>
                <em>¥{{ fmt(row.budget) }}</em>
              </div>
              <div class="bb-bar bb-bar--actual" :class="{ 'is-over': row.actual > row.budget && row.budget > 0 }">
                <i :style="{ width: barWidth(row.actual, maxCategoryAmount) }"></i>
                <em>¥{{ fmt(row.actual) }}</em>
              </div>
            </div>
          </div>
        </div>
      </section>

      <section class="bb-panel">
        <header class="bb-panel-head">
          <h3>{{ t('result.budgetBoard.pieTitle') }}</h3>
        </header>
        <div v-if="actualTotal > 0" ref="pieRef" class="bb-chart"></div>
        <a-empty v-else :description="t('result.budgetBoard.emptyExpense')" />
      </section>
    </div>

    <!-- 每日趋势 -->
    <section class="bb-panel">
      <header class="bb-panel-head">
        <h3>{{ t('result.budgetBoard.trendTitle') }}</h3>
        <span class="bb-panel-hint">{{ t('result.budgetBoard.trendHint') }}</span>
      </header>
      <div v-if="actualTotal > 0" ref="lineRef" class="bb-chart bb-chart--wide"></div>
      <a-empty v-else :description="t('result.budgetBoard.emptyExpense')" />
    </section>

    <!-- 成员分摊与结算 -->
    <section class="bb-panel">
      <header class="bb-panel-head">
        <h3>{{ t('result.budgetBoard.settleTitle') }}</h3>
        <span class="bb-panel-hint">{{ t('result.budgetBoard.settleHint') }}</span>
      </header>

      <div v-if="expenses.length" class="bb-settle">
        <div class="bb-table">
          <div class="bb-tr bb-tr--head">
            <span>{{ t('result.budgetBoard.member') }}</span>
            <span>{{ t('result.budgetBoard.paid') }}</span>
            <span>{{ t('result.budgetBoard.share') }}</span>
            <span>{{ t('result.budgetBoard.net') }}</span>
          </div>
          <div v-for="m in memberStats" :key="m.name" class="bb-tr">
            <span class="bb-td-strong">{{ m.name }}</span>
            <span>¥{{ fmt(m.paid) }}</span>
            <span>¥{{ fmt(m.owed) }}</span>
            <span :class="m.net >= 0 ? 'bb-net-plus' : 'bb-net-minus'">
              {{ m.net >= 0 ? '+' : '-' }}¥{{ fmt(Math.abs(m.net)) }}
            </span>
          </div>
        </div>

        <div class="bb-suggest">
          <div class="bb-suggest-title">{{ t('result.budgetBoard.suggestTitle') }}</div>
          <ul v-if="settlements.length" class="bb-suggest-list">
            <li v-for="(s, i) in settlements" :key="i">
              <b>{{ s.from }}</b>
              <span class="bb-arrow">→</span>
              <b>{{ s.to }}</b>
              <span class="bb-amount">¥{{ fmt(s.amount) }}</span>
            </li>
          </ul>
          <p v-else class="bb-suggest-done">{{ t('result.budgetBoard.settled') }}</p>
        </div>
      </div>
      <a-empty v-else :description="t('result.budgetBoard.emptyExpense')" />
    </section>

    <!-- 支出明细 -->
    <section class="bb-panel">
      <header class="bb-panel-head">
        <h3>{{ t('result.budgetBoard.listTitle') }}</h3>
        <span class="bb-panel-hint">{{ t('result.budgetBoard.listCount', { n: expenses.length }) }}</span>
      </header>
      <div v-if="expenses.length" class="bb-expenses">
        <div v-for="e in sortedExpenses" :key="e.id" class="bb-expense">
          <span class="bb-expense-cat" :style="{ '--c': categoryColor(e.category) }">
            {{ t(`result.budget.${categoryI18n(e.category)}`) }}
          </span>
          <span class="bb-expense-title">{{ e.title }}</span>
          <span class="bb-expense-day">
            {{ e.dayIndex ? t('common.dayNumber', { day: e.dayIndex }) : '--' }}
          </span>
          <span class="bb-expense-payer">{{ e.payer }}</span>
          <span class="bb-expense-amount">¥{{ fmt(e.amount) }}</span>
          <button type="button" class="bb-del" :title="t('common.delete')" @click="removeExpense(e.id)">
            ×
          </button>
        </div>
      </div>
      <a-empty v-else :description="t('result.budgetBoard.emptyExpense')" />
    </section>

    <!-- 新增支出 -->
    <a-modal
      v-model:open="expenseModalOpen"
      :title="t('result.budgetBoard.addExpense')"
      :ok-text="t('common.confirm')"
      :cancel-text="t('common.cancel')"
      @ok="submitExpense"
    >
      <div class="bb-form">
        <label class="bb-field">
          <span>{{ t('result.budgetBoard.fieldTitle') }}</span>
          <a-input v-model:value="form.title" :placeholder="t('result.budgetBoard.fieldTitlePh')" />
        </label>
        <div class="bb-field-row">
          <label class="bb-field">
            <span>{{ t('result.budgetBoard.fieldCategory') }}</span>
            <a-select v-model:value="form.category">
              <a-select-option v-for="c in categories" :key="c" :value="c">
                {{ t(`result.budget.${categoryI18n(c)}`) }}
              </a-select-option>
            </a-select>
          </label>
          <label class="bb-field">
            <span>{{ t('result.budgetBoard.fieldAmount') }}</span>
            <a-input-number v-model:value="form.amount" :min="0" :precision="2" style="width: 100%" />
          </label>
        </div>
        <div class="bb-field-row">
          <label class="bb-field">
            <span>{{ t('result.budgetBoard.fieldDay') }}</span>
            <a-select v-model:value="form.dayIndex">
              <a-select-option :value="0">{{ t('result.budgetBoard.dayUnset') }}</a-select-option>
              <a-select-option v-for="d in dayOptions" :key="d" :value="d">
                {{ t('common.dayNumber', { day: d }) }}
              </a-select-option>
            </a-select>
          </label>
          <label class="bb-field">
            <span>{{ t('result.budgetBoard.fieldPayer') }}</span>
            <a-select v-model:value="form.payer">
              <a-select-option v-for="m in members" :key="m" :value="m">{{ m }}</a-select-option>
            </a-select>
          </label>
        </div>
        <label class="bb-field">
          <span>{{ t('result.budgetBoard.fieldParticipants') }}</span>
          <a-checkbox-group v-model:value="form.participants" class="bb-participants">
            <a-checkbox v-for="m in members" :key="m" :value="m">{{ m }}</a-checkbox>
          </a-checkbox-group>
        </label>
      </div>
    </a-modal>

    <!-- 成员管理 -->
    <a-modal
      v-model:open="memberModalOpen"
      :title="t('result.budgetBoard.manageMembers')"
      :ok-text="t('common.confirm')"
      :cancel-text="t('common.cancel')"
      @ok="memberModalOpen = false"
    >
      <div class="bb-members">
        <div class="bb-member-add">
          <a-input
            v-model:value="newMember"
            :placeholder="t('result.budgetBoard.memberPh')"
            @press-enter="addMember"
          />
          <a-button type="primary" @click="addMember">{{ t('result.budgetBoard.memberAdd') }}</a-button>
        </div>
        <div class="bb-member-list">
          <span v-for="m in members" :key="m" class="bb-member-chip">
            {{ m }}
            <button type="button" :disabled="members.length <= 1" @click="removeMember(m)">×</button>
          </span>
        </div>
        <p class="bb-member-hint">{{ t('result.budgetBoard.memberHint') }}</p>
      </div>
    </a-modal>
  </a-card>
</template>

<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import * as echarts from 'echarts'
import type { TripPlan } from '../types'

const props = defineProps<{ plan: TripPlan }>()
const { t } = useI18n()

type Category = 'attraction' | 'hotel' | 'meal' | 'transport' | 'other'

interface Expense {
  id: string
  title: string
  category: Category
  amount: number
  dayIndex: number
  payer: string
  participants: string[]
  createdAt: number
}

const categories: Category[] = ['attraction', 'hotel', 'meal', 'transport', 'other']
const CATEGORY_META: Record<Category, { i18n: string; color: string; budgetKey?: keyof NonNullable<TripPlan['budget']> }> = {
  attraction: { i18n: 'attraction', color: '#4f7cff', budgetKey: 'total_attractions' },
  hotel: { i18n: 'hotel', color: '#7c5cff', budgetKey: 'total_hotels' },
  meal: { i18n: 'meal', color: '#ff9f43', budgetKey: 'total_meals' },
  transport: { i18n: 'transport', color: '#22c1a6', budgetKey: 'total_transportation' },
  other: { i18n: 'other', color: '#94a3b8' }
}

const STORAGE_PREFIX = 'tripstar.budgetBoard.v1:'

const storeKey = computed(
  () => `${STORAGE_PREFIX}${props.plan.city}|${props.plan.start_date}|${props.plan.end_date}`
)

const members = ref<string[]>(['我'])
const expenses = ref<Expense[]>([])
const expenseModalOpen = ref(false)
const memberModalOpen = ref(false)
const newMember = ref('')

const form = reactive<{
  title: string
  category: Category
  amount: number | null
  dayIndex: number
  payer: string
  participants: string[]
}>({
  title: '',
  category: 'meal',
  amount: null,
  dayIndex: 0,
  payer: '我',
  participants: ['我']
})

/* ---------- 持久化 ---------- */
function load() {
  members.value = ['我']
  expenses.value = []
  if (typeof localStorage === 'undefined') return
  try {
    const raw = localStorage.getItem(storeKey.value)
    if (!raw) return
    const data = JSON.parse(raw)
    if (Array.isArray(data?.members) && data.members.length) members.value = data.members
    if (Array.isArray(data?.expenses)) {
      expenses.value = data.expenses.filter((e: Expense) => e && typeof e.amount === 'number')
    }
    if (!members.value.includes(form.payer)) form.payer = members.value[0]
  } catch {
    /* 数据损坏时回落默认，不阻断页面 */
  }
}

function save() {
  if (typeof localStorage === 'undefined') return
  try {
    localStorage.setItem(storeKey.value, JSON.stringify({ members: members.value, expenses: expenses.value }))
  } catch {
    /* 配额超限时静默失败 */
  }
}

watch([members, expenses], save, { deep: true })
watch(storeKey, load)
onMounted(load)

/* ---------- 计算 ---------- */
const budgetTotal = computed(() => Number(props.plan.budget?.total ?? 0))
const actualTotal = computed(() => expenses.value.reduce((s, e) => s + (Number(e.amount) || 0), 0))
const remain = computed(() => budgetTotal.value - actualTotal.value)
const usageRate = computed(() => (budgetTotal.value > 0 ? (actualTotal.value / budgetTotal.value) * 100 : 0))

const categoryRows = computed(() =>
  categories.map((c) => {
    const meta = CATEGORY_META[c]
    const budget = meta.budgetKey ? Number(props.plan.budget?.[meta.budgetKey] ?? 0) : 0
    const actual = expenses.value
      .filter((e) => e.category === c)
      .reduce((s, e) => s + (Number(e.amount) || 0), 0)
    return { key: c, i18n: meta.i18n, budget, actual }
  })
)

const maxCategoryAmount = computed(() =>
  Math.max(1, ...categoryRows.value.flatMap((r) => [r.budget, r.actual]))
)

const dayOptions = computed(() => props.plan.days?.map((d) => d.day_index) ?? [])

const sortedExpenses = computed(() =>
  [...expenses.value].sort((a, b) => b.createdAt - a.createdAt)
)

const memberStats = computed(() => {
  const names = [...members.value]
  const paid = new Map<string, number>(names.map((n) => [n, 0]))
  const owed = new Map<string, number>(names.map((n) => [n, 0]))

  for (const e of expenses.value) {
    const amount = Number(e.amount) || 0
    if (paid.has(e.payer)) paid.set(e.payer, (paid.get(e.payer) as number) + amount)
    const share = Array.isArray(e.participants) && e.participants.length ? e.participants : names
    const per = amount / share.length
    for (const p of share) {
      if (owed.has(p)) owed.set(p, (owed.get(p) as number) + per)
    }
  }

  return names.map((n) => ({
    name: n,
    paid: paid.get(n) as number,
    owed: owed.get(n) as number,
    net: (paid.get(n) as number) - (owed.get(n) as number)
  }))
})

const settlements = computed(() => {
  const debtors = memberStats.value
    .filter((m) => m.net < -0.01)
    .map((m) => ({ name: m.name, amount: -m.net }))
    .sort((a, b) => b.amount - a.amount)
  const creditors = memberStats.value
    .filter((m) => m.net > 0.01)
    .map((m) => ({ name: m.name, amount: m.net }))
    .sort((a, b) => b.amount - a.amount)

  const out: { from: string; to: string; amount: number }[] = []
  let i = 0
  let j = 0
  let guard = 0
  while (i < debtors.length && j < creditors.length && guard++ < 200) {
    const pay = Math.min(debtors[i].amount, creditors[j].amount)
    if (pay > 0.01) {
      out.push({ from: debtors[i].name, to: creditors[j].name, amount: Math.round(pay * 100) / 100 })
    }
    debtors[i].amount -= pay
    creditors[j].amount -= pay
    if (debtors[i].amount <= 0.01) i++
    if (creditors[j].amount <= 0.01) j++
  }
  return out
})

/* ---------- 图表 ---------- */
const pieRef = ref<HTMLElement | null>(null)
const lineRef = ref<HTMLElement | null>(null)
let pie: echarts.ECharts | null = null
let line: echarts.ECharts | null = null

function renderPie() {
  if (!pieRef.value) return
  if (!pie) pie = echarts.init(pieRef.value)
  const data = categories
    .map((c) => ({
      name: t(`result.budget.${CATEGORY_META[c].i18n}`),
      value: expenses.value.filter((e) => e.category === c).reduce((s, e) => s + (Number(e.amount) || 0), 0),
      itemStyle: { color: CATEGORY_META[c].color }
    }))
    .filter((d) => d.value > 0)

  pie.setOption(
    {
      tooltip: { trigger: 'item', formatter: '{b}: ¥{c} ({d}%)' },
      legend: { bottom: 0, itemWidth: 10, itemHeight: 10, textStyle: { fontSize: 12, color: 'rgba(255,255,255,0.72)' } },
      series: [
        {
          type: 'pie',
          radius: ['46%', '70%'],
          center: ['50%', '44%'],
          avoidLabelOverlap: true,
          itemStyle: { borderColor: 'rgba(3,10,15,0.9)', borderWidth: 2 },
          label: { show: false },
          data
        }
      ]
    },
    true
  )
}

function renderLine() {
  if (!lineRef.value) return
  if (!line) line = echarts.init(lineRef.value)
  const days = dayOptions.value.length ? dayOptions.value : [1]
  const values = days.map((d) =>
    expenses.value.filter((e) => e.dayIndex === d).reduce((s, e) => s + (Number(e.amount) || 0), 0)
  )
  const unset = expenses.value
    .filter((e) => !e.dayIndex)
    .reduce((s, e) => s + (Number(e.amount) || 0), 0)

  line.setOption(
    {
      grid: { left: 8, right: 16, top: 24, bottom: 8, containLabel: true },
      tooltip: {
        trigger: 'axis',
        valueFormatter: (v: number) => `¥${Number(v).toLocaleString('zh-CN')}`
      },
      xAxis: {
        type: 'category',
        data: days.map((d) => t('common.dayNumber', { day: d })),
        axisLine: { lineStyle: { color: 'rgba(255,255,255,0.16)' } },
        axisLabel: { color: 'rgba(255,255,255,0.55)', fontSize: 12 }
      },
      yAxis: {
        type: 'value',
        splitLine: { lineStyle: { color: 'rgba(255,255,255,0.08)' } },
        axisLabel: { color: 'rgba(255,255,255,0.55)', fontSize: 12 }
      },
      series: [
        {
          type: 'line',
          smooth: true,
          symbolSize: 7,
          data: values,
          lineStyle: { width: 2.5, color: '#4f7cff' },
          itemStyle: { color: '#4f7cff' },
          areaStyle: {
            color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
              { offset: 0, color: 'rgba(79,124,255,0.26)' },
              { offset: 1, color: 'rgba(79,124,255,0.02)' }
            ])
          }
        }
      ],
      graphic: unset
        ? [
            {
              type: 'text',
              right: 12,
              top: 0,
              style: { text: `${t('result.budgetBoard.dayUnset')}: ¥${unset.toLocaleString('zh-CN')}`, fontSize: 11, fill: 'rgba(255,255,255,0.45)' }
            }
          ]
        : []
    },
    true
  )
}

function renderCharts() {
  nextTick(() => {
    renderPie()
    renderLine()
  })
}

watch([expenses, members], renderCharts, { deep: true })
onMounted(() => {
  renderCharts()
  window.addEventListener('resize', handleResize)
})

function handleResize() {
  pie?.resize()
  line?.resize()
}

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  pie?.dispose()
  line?.dispose()
  pie = null
  line = null
})

/* ---------- 操作 ---------- */
function fmt(n: number) {
  return (Number(n) || 0).toLocaleString('zh-CN', { maximumFractionDigits: 2 })
}

function barWidth(v: number, max: number) {
  return `${Math.max(v > 0 ? 2 : 0, (v / max) * 100)}%`
}

function categoryI18n(c: Category) {
  return CATEGORY_META[c].i18n
}

function categoryColor(c: Category) {
  return CATEGORY_META[c].color
}

function openExpenseModal() {
  form.title = ''
  form.category = 'meal'
  form.amount = null
  form.dayIndex = 0
  form.payer = members.value[0]
  form.participants = [...members.value]
  expenseModalOpen.value = true
}

function submitExpense() {
  const title = form.title.trim()
  const amount = Number(form.amount)
  if (!title || !Number.isFinite(amount) || amount <= 0) return
  expenses.value = [
    ...expenses.value,
    {
      id: `e_${Date.now().toString(36)}_${Math.random().toString(36).slice(2, 6)}`,
      title: title.slice(0, 40),
      category: form.category,
      amount: Math.round(amount * 100) / 100,
      dayIndex: Number(form.dayIndex) || 0,
      payer: form.payer,
      participants: form.participants.length ? [...form.participants] : [...members.value],
      createdAt: Date.now()
    }
  ]
  expenseModalOpen.value = false
}

function removeExpense(id: string) {
  expenses.value = expenses.value.filter((e) => e.id !== id)
}

function addMember() {
  const name = newMember.value.trim()
  if (!name || members.value.includes(name)) return
  members.value = [...members.value, name.slice(0, 12)]
  newMember.value = ''
}

function removeMember(name: string) {
  if (members.value.length <= 1) return
  members.value = members.value.filter((m) => m !== name)
  if (form.payer === name) form.payer = members.value[0]
  form.participants = form.participants.filter((p) => p !== name)
}
</script>

<style scoped>
.bb-card {
  background: transparent;
}

.bb-head {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
  flex-wrap: wrap;
  margin-bottom: 16px;
}
.bb-title {
  margin: 0;
  font-size: 18px;
  font-weight: 700;
  color: rgba(255, 255, 255, 0.94);
}
.bb-sub {
  margin: 4px 0 0;
  font-size: 12.5px;
  color: rgba(255, 255, 255, 0.6);
}

.bb-kpi {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 12px;
  margin-bottom: 14px;
}
.bb-kpi-item {
  background: rgba(3, 10, 15, 0.88);
  border: 1.2px solid rgba(255, 255, 255, 0.14);
  border-radius: 12px;
  padding: 14px 16px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.bb-kpi-label {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.6);
}
.bb-kpi-value {
  font-size: 22px;
  font-weight: 700;
  color: rgba(255, 255, 255, 0.94);
  font-variant-numeric: tabular-nums;
}
.bb-kpi-item.is-over .bb-kpi-value {
  color: #e5484d;
}
.bb-kpi-progress {
  background: rgba(3, 10, 15, 0.88);
  border: 1.2px solid rgba(255, 255, 255, 0.14);
  border-radius: 12px;
  padding: 12px 16px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 8px;
}
.bb-progress-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 12px;
  color: rgba(255, 255, 255, 0.6);
}
.bb-progress-pct {
  font-weight: 700;
  color: #4f7cff;
  font-variant-numeric: tabular-nums;
}
.bb-progress-pct.is-over {
  color: #e5484d;
}
.bb-progress-track {
  height: 6px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.1);
  overflow: hidden;
}
.bb-progress-fill {
  display: block;
  height: 100%;
  border-radius: 999px;
  background: linear-gradient(90deg, #4f7cff, #7c5cff);
  transition: width 0.35s ease;
}
.bb-progress-fill.is-over {
  background: linear-gradient(90deg, #ff7a45, #e5484d);
}

.bb-grid {
  display: grid;
  grid-template-columns: 1.15fr 1fr;
  gap: 12px;
  margin-bottom: 12px;
}

.bb-panel {
  background: rgba(3, 10, 15, 0.88);
  border: 1.2px solid rgba(255, 255, 255, 0.14);
  border-radius: 14px;
  padding: 16px 18px;
  margin-bottom: 12px;
}
.bb-panel-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 14px;
  flex-wrap: wrap;
}
.bb-panel-head h3 {
  margin: 0;
  font-size: 14px;
  font-weight: 650;
  color: rgba(255, 255, 255, 0.94);
}
.bb-panel-hint {
  font-size: 11.5px;
  color: rgba(255, 255, 255, 0.45);
}
.bb-legend {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  font-size: 11.5px;
  color: rgba(255, 255, 255, 0.6);
}
.lg {
  width: 9px;
  height: 9px;
  border-radius: 3px;
  display: inline-block;
  margin-left: 6px;
}
.lg--budget {
  background: rgba(255, 255, 255, 0.22);
}
.lg--actual {
  background: #4f7cff;
}

.bb-compare {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.bb-compare-row {
  display: flex;
  align-items: center;
  gap: 12px;
}
.bb-compare-name {
  width: 86px;
  flex: 0 0 86px;
  font-size: 12px;
  color: rgba(255, 255, 255, 0.68);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.bb-compare-bars {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.bb-bar {
  position: relative;
  height: 12px;
  border-radius: 4px;
  background: rgba(255, 255, 255, 0.08);
  overflow: visible;
}
.bb-bar i {
  display: block;
  height: 100%;
  border-radius: 4px;
  transition: width 0.35s ease;
}
.bb-bar--budget i {
  background: rgba(255, 255, 255, 0.22);
}
.bb-bar--actual i {
  background: #4f7cff;
}
.bb-bar--actual.is-over i {
  background: #e5484d;
}
.bb-bar em {
  position: absolute;
  right: 6px;
  top: -1px;
  font-size: 10.5px;
  font-style: normal;
  color: rgba(255, 255, 255, 0.45);
  line-height: 12px;
}

.bb-chart {
  width: 100%;
  height: 210px;
}
.bb-chart--wide {
  height: 230px;
}

.bb-settle {
  display: grid;
  grid-template-columns: 1.4fr 1fr;
  gap: 16px;
}
.bb-table {
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  overflow: hidden;
}
.bb-tr {
  display: grid;
  grid-template-columns: 1.1fr 1fr 1fr 1fr;
  gap: 8px;
  padding: 9px 12px;
  font-size: 12.5px;
  color: rgba(255, 255, 255, 0.68);
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  font-variant-numeric: tabular-nums;
}
.bb-tr:last-child {
  border-bottom: none;
}
.bb-tr--head {
  background: rgba(255, 255, 255, 0.04);
  color: rgba(255, 255, 255, 0.45);
  font-size: 11.5px;
}
.bb-td-strong {
  color: rgba(255, 255, 255, 0.94);
  font-weight: 600;
}
.bb-net-plus {
  color: #34d399;
  font-weight: 600;
}
.bb-net-minus {
  color: #e5484d;
  font-weight: 600;
}

.bb-suggest {
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 10px;
  padding: 12px 14px;
}
.bb-suggest-title {
  font-size: 12.5px;
  font-weight: 650;
  color: rgba(255, 255, 255, 0.94);
  margin-bottom: 8px;
}
.bb-suggest-list {
  margin: 0;
  padding: 0;
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.bb-suggest-list li {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12.5px;
  color: rgba(255, 255, 255, 0.68);
}
.bb-suggest-list b {
  color: rgba(255, 255, 255, 0.94);
  font-weight: 600;
}
.bb-arrow {
  color: rgba(255, 255, 255, 0.4);
}
.bb-amount {
  margin-left: auto;
  font-weight: 650;
  color: #4f7cff;
  font-variant-numeric: tabular-nums;
}
.bb-suggest-done {
  margin: 0;
  font-size: 12.5px;
  color: #34d399;
}

.bb-expenses {
  display: flex;
  flex-direction: column;
}
.bb-expense {
  display: grid;
  grid-template-columns: 78px 1fr 62px 70px 96px 28px;
  gap: 10px;
  align-items: center;
  padding: 9px 2px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
  font-size: 12.5px;
  color: rgba(255, 255, 255, 0.68);
}
.bb-expense:last-child {
  border-bottom: none;
}
.bb-expense-cat {
  justify-self: start;
  font-size: 11.5px;
  padding: 3px 9px;
  border-radius: 999px;
  color: var(--c);
  background: color-mix(in srgb, var(--c) 12%, transparent);
  border: 1px solid color-mix(in srgb, var(--c) 26%, transparent);
}
.bb-expense-title {
  color: rgba(255, 255, 255, 0.94);
  font-weight: 550;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.bb-expense-day,
.bb-expense-payer {
  color: rgba(255, 255, 255, 0.45);
}
.bb-expense-amount {
  text-align: right;
  font-weight: 650;
  color: rgba(255, 255, 255, 0.94);
  font-variant-numeric: tabular-nums;
}
.bb-del {
  border: none;
  background: transparent;
  color: rgba(255, 255, 255, 0.35);
  font-size: 16px;
  line-height: 1;
  cursor: pointer;
  border-radius: 6px;
  height: 22px;
}
.bb-del:hover {
  background: rgba(229, 72, 77, 0.18);
  color: #e5484d;
}

.bb-form {
  display: flex;
  flex-direction: column;
  gap: 14px;
  padding-top: 4px;
}
.bb-field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.bb-field > span {
  font-size: 12.5px;
  color: rgba(255, 255, 255, 0.68);
}
.bb-field-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}
.bb-participants {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.bb-members {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding-top: 4px;
}
.bb-member-add {
  display: flex;
  gap: 8px;
}
.bb-member-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.bb-member-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 5px 10px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.08);
  border: 1.2px solid rgba(255, 255, 255, 0.14);
  font-size: 12.5px;
  color: rgba(255, 255, 255, 0.94);
}
.bb-member-chip button {
  border: none;
  background: transparent;
  color: rgba(255, 255, 255, 0.45);
  cursor: pointer;
  font-size: 14px;
  line-height: 1;
}
.bb-member-chip button:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}
.bb-member-hint {
  margin: 0;
  font-size: 11.5px;
  color: rgba(255, 255, 255, 0.45);
}

@media (max-width: 1100px) {
  .bb-kpi {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .bb-grid,
  .bb-settle {
    grid-template-columns: 1fr;
  }
}
</style>
