<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="card budget-card">
      <div class="budget-row">
        <label for="budget-slider" class="budget-label">{{ t('restocking.budgetSlider') }}</label>
        <span class="budget-value">{{ currencySymbol }}{{ formatMoney(budget) }}</span>
      </div>
      <input
        id="budget-slider"
        v-model.number="budget"
        type="range"
        min="0"
        max="50000"
        step="500"
        class="budget-slider"
      />
      <div class="budget-scale">
        <span>{{ currencySymbol }}0</span>
        <span>{{ currencySymbol }}50,000</span>
      </div>
    </div>

    <div v-if="feedback" :class="['feedback', feedback.type]" role="status">
      {{ feedback.text }}
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ recommendations.length }})</h3>
        <div class="header-actions">
          <button class="btn-secondary" :disabled="loading || !recommendations.length" @click="selectAll">
            {{ t('restocking.buttons.selectAll') }}
          </button>
          <button class="btn-secondary" :disabled="!selectedItems.length" @click="clearSelection">
            {{ t('restocking.buttons.clearAll') }}
          </button>
        </div>
      </div>

      <div v-if="loading" class="loading">
        <span class="spinner" aria-hidden="true"></span>
        {{ t('common.loading') }}
      </div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="!recommendations.length" class="empty">{{ t('restocking.noRecommendations') }}</div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th class="select-col"></th>
              <th>{{ t('restocking.columns.sku') }}</th>
              <th>{{ t('restocking.columns.itemName') }}</th>
              <th>{{ t('restocking.columns.category') }}</th>
              <th>{{ t('restocking.columns.currentStock') }}</th>
              <th>{{ t('restocking.columns.forecastedDemand') }}</th>
              <th>{{ t('restocking.columns.shortage') }}</th>
              <th>{{ t('restocking.columns.unitCost') }}</th>
              <th>{{ t('restocking.columns.totalCost') }}</th>
              <th>{{ t('restocking.columns.trend') }}</th>
              <th>{{ t('restocking.columns.priority') }}</th>
              <th>{{ t('restocking.columns.warehouse') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="item in recommendations"
              :key="rowKey(item)"
              :class="{ 'row-disabled': isDisabled(item), 'row-selected': isSelected(item) }"
            >
              <td class="select-col">
                <input
                  type="checkbox"
                  :checked="isSelected(item)"
                  :disabled="isDisabled(item)"
                  :title="isDisabled(item) ? t('restocking.messages.exceedsBudget') : ''"
                  :aria-label="item.item_sku"
                  @change="toggleItem(item)"
                />
              </td>
              <td><strong>{{ item.item_sku }}</strong></td>
              <td>{{ translateProductName(item.item_name) }}</td>
              <td>{{ translateCategory(item.category) }}</td>
              <td>{{ item.current_stock }}</td>
              <td>{{ item.forecasted_demand }}</td>
              <td><strong>{{ item.shortage }}</strong></td>
              <td>{{ currencySymbol }}{{ formatMoney(item.unit_cost) }}</td>
              <td><strong>{{ currencySymbol }}{{ formatMoney(item.total_cost_for_shortage) }}</strong></td>
              <td>{{ item.trend }}</td>
              <td>
                <span :class="['badge', priorityClass(item.priority)]">{{ item.priority }}</span>
              </td>
              <td>{{ translateWarehouse(item.warehouse) }}</td>
            </tr>
          </tbody>
        </table>
      </div>

      <div v-if="!loading && !error && recommendations.length" class="summary">
        <div class="summary-stats">
          <div>
            <span class="summary-label">{{ t('restocking.selections.selected') }}</span>
            <strong>{{ selectedRecords.length }}</strong>
          </div>
          <div>
            <span class="summary-label">{{ t('restocking.selections.total') }}</span>
            <strong>{{ currencySymbol }}{{ formatMoney(selectedTotal) }}</strong>
          </div>
          <div>
            <span class="summary-label">{{ t('restocking.selections.remaining') }}</span>
            <strong>{{ currencySymbol }}{{ formatMoney(remainingBudget) }}</strong>
          </div>
        </div>
        <div class="summary-action">
          <span v-if="!selectedRecords.length" class="hint">{{ t('restocking.messages.selectItems') }}</span>
          <button class="btn-primary" :disabled="!selectedRecords.length || submitting" @click="placeOrder">
            {{ t('restocking.buttons.placeOrder') }}
          </button>
        </div>
      </div>
    </div>

    <div v-if="submittedOrders.length" class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.submittedOrders') }} ({{ submittedOrders.length }})</h3>
      </div>
      <div class="table-container">
        <table>
          <thead>
            <tr>
              <th>PO</th>
              <th>{{ t('restocking.columns.sku') }}</th>
              <th>{{ t('restocking.columns.shortage') }}</th>
              <th>{{ t('restocking.columns.unitCost') }}</th>
              <th>{{ t('restocking.leadTime') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(po, idx) in submittedOrders" :key="po.id || po.po_number || `${po.backlog_item_id}-${idx}`">
              <td><strong>{{ po.id || po.po_number || '-' }}</strong></td>
              <td>{{ po.backlog_item_id ? String(po.backlog_item_id).replace('RESTOCK-', '') : '-' }}</td>
              <td>{{ po.quantity ?? '-' }}</td>
              <td>{{ po.unit_cost != null ? currencySymbol + formatMoney(po.unit_cost) : '-' }}</td>
              <td>{{ po.expected_delivery_date || '-' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const { t, currentCurrency, translateProductName, translateWarehouse } = useI18n()

const currencySymbol = computed(() => (currentCurrency.value === 'JPY' ? '¥' : '$'))

const budget = ref(10000)
const recommendations = ref([])
const selectedItems = ref([]) // row keys of selected recommendations
const loading = ref(true)
const error = ref(null)
const submitting = ref(false)
const feedback = ref(null)
const submittedOrders = ref([])

const rowKey = (item) => `${item.item_sku}|${item.warehouse}`

const formatMoney = (value) => {
  const n = Number(value)
  if (!Number.isFinite(n)) return '0.00'
  return n.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })
}

const translateCategory = (category) => {
  const map = {
    'Circuit Boards': t('categories.circuitBoards'),
    'Sensors': t('categories.sensors'),
    'Actuators': t('categories.actuators'),
    'Controllers': t('categories.controllers'),
    'Power Supplies': t('categories.powerSupplies')
  }
  return map[category] || category
}

const priorityClass = (priority) => {
  switch (String(priority).toLowerCase()) {
    case 'high': return 'success'
    case 'medium': return 'warning'
    case 'low': return 'danger'
    default: return ''
  }
}

const itemCost = (item) => Number(item.total_cost_for_shortage) || 0

const selectedRecords = computed(() =>
  recommendations.value.filter(item => selectedItems.value.includes(rowKey(item)))
)

const selectedTotal = computed(() =>
  selectedRecords.value.reduce((sum, item) => sum + itemCost(item), 0)
)

const remainingBudget = computed(() => budget.value - selectedTotal.value)

const isSelected = (item) => selectedItems.value.includes(rowKey(item))

// Hard limit: unselected items that would push the total over budget are disabled
const isDisabled = (item) => !isSelected(item) && itemCost(item) > remainingBudget.value

const toggleItem = (item) => {
  const key = rowKey(item)
  if (isSelected(item)) {
    selectedItems.value = selectedItems.value.filter(k => k !== key)
  } else if (!isDisabled(item)) {
    selectedItems.value = [...selectedItems.value, key]
  }
}

const selectAll = () => {
  let running = selectedTotal.value
  const keys = new Set(selectedItems.value)
  for (const item of recommendations.value) {
    const key = rowKey(item)
    if (keys.has(key)) continue
    const cost = itemCost(item)
    if (running + cost <= budget.value) {
      keys.add(key)
      running += cost
    }
  }
  selectedItems.value = [...keys]
}

const clearSelection = () => {
  selectedItems.value = []
}

let requestId = 0
const loadRecommendations = async () => {
  const current = ++requestId
  loading.value = true
  error.value = null
  try {
    const data = await api.getRestockingRecommendations(budget.value)
    if (current !== requestId) return
    const list = Array.isArray(data) ? data : (data?.recommendations ?? [])
    recommendations.value = list
    // Drop selections no longer valid for the new budget/result set
    const valid = new Set(list.map(rowKey))
    let running = 0
    selectedItems.value = selectedItems.value.filter(key => {
      if (!valid.has(key)) return false
      const item = list.find(i => rowKey(i) === key)
      running += itemCost(item)
      return running <= budget.value
    })
  } catch (err) {
    if (current !== requestId) return
    error.value = 'Failed to load recommendations: ' + err.message
    recommendations.value = []
  } finally {
    if (current === requestId) loading.value = false
  }
}

const placeOrder = async () => {
  if (!selectedRecords.value.length || submitting.value) return
  submitting.value = true
  feedback.value = null
  try {
    const payload = selectedRecords.value.map(item => ({
      item_sku: item.item_sku,
      item_name: item.item_name,
      quantity: item.shortage,
      unit_cost: item.unit_cost
    }))
    const result = await api.submitRestockingOrder(payload)
    submittedOrders.value = Array.isArray(result) ? result : []
    feedback.value = { type: 'success', text: t('restocking.messages.orderPlaced') }
    clearSelection()
  } catch (err) {
    feedback.value = {
      type: 'error',
      text: `${t('restocking.messages.orderFailed')}: ${err.message}`
    }
  } finally {
    submitting.value = false
  }
}

let debounceTimer = null
watch(budget, () => {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(loadRecommendations, 300)
})

onMounted(loadRecommendations)
</script>

<style scoped>
.page-header {
  margin-bottom: 1.5rem;
}

.page-header h2 {
  margin-bottom: 0.25rem;
}

.page-header p {
  color: #64748b;
  font-size: 0.875rem;
}

.budget-card {
  padding: 1.25rem 1.5rem;
  margin-bottom: 1.5rem;
}

.budget-row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 0.75rem;
}

.budget-label {
  font-weight: 600;
  color: #0f172a;
}

.budget-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  accent-color: #3b82f6;
}

.budget-scale {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 1rem;
  padding: 1.25rem 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.card-title {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
  margin: 0;
}

.header-actions {
  display: flex;
  gap: 0.5rem;
}

.btn-primary,
.btn-secondary {
  padding: 0.5rem 1rem;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-primary {
  background: #3b82f6;
  color: white;
  border: 1px solid #3b82f6;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-secondary {
  background: white;
  color: #334155;
  border: 1px solid #cbd5e1;
}

.btn-secondary:hover:not(:disabled) {
  background: #f1f5f9;
}

.btn-primary:disabled,
.btn-secondary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.table-container {
  overflow-x: auto;
}

.select-col {
  width: 40px;
  text-align: center;
}

.row-selected {
  background: #eff6ff;
}

.row-disabled {
  opacity: 0.45;
}

.row-disabled td {
  background: #f8fafc;
}

.loading,
.error,
.empty {
  padding: 2rem;
  text-align: center;
  color: #64748b;
}

.error {
  color: #ef4444;
}

.spinner {
  display: inline-block;
  width: 16px;
  height: 16px;
  margin-right: 0.5rem;
  vertical-align: middle;
  border: 2px solid #cbd5e1;
  border-top-color: #3b82f6;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.feedback {
  padding: 0.75rem 1rem;
  margin-bottom: 1rem;
  border-radius: 8px;
  font-size: 0.875rem;
}

.feedback.success {
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  color: #166534;
}

.feedback.error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #b91c1c;
  padding: 0.75rem 1rem;
  text-align: left;
}

.summary {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 1rem;
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
}

.summary-stats {
  display: flex;
  gap: 2rem;
  flex-wrap: wrap;
}

.summary-stats > div {
  display: flex;
  flex-direction: column;
}

.summary-label {
  font-size: 0.75rem;
  color: #64748b;
}

.summary-action {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.hint {
  font-size: 0.875rem;
  color: #64748b;
}
</style>
