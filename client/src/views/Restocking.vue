<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <span class="card-title">{{ t('restocking.budgetLabel') }}</span>
          <span class="budget-display">${{ budget.toLocaleString() }}</span>
        </div>
        <input
          type="range"
          v-model.number="budget"
          :min="BUDGET_MIN"
          :max="BUDGET_MAX"
          :step="BUDGET_STEP"
          class="budget-slider"
        />
        <div class="budget-meta">
          <span>${{ BUDGET_MIN.toLocaleString() }}</span>
          <span>${{ BUDGET_MAX.toLocaleString() }}</span>
        </div>
        <p class="budget-hint">{{ t('restocking.budgetHint') }}</p>
      </div>

      <!-- Stats Row -->
      <div class="stats-grid stats-grid-3">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.itemsSelected', { count: selectedItems.length }) }}</div>
          <div class="stat-value">{{ selectedItems.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.estimatedTotal') }}</div>
          <div class="stat-value">${{ estimatedTotal.toLocaleString() }}</div>
        </div>
        <div class="stat-card" :class="remainingBudget >= 0 ? 'success' : 'danger'">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">${{ Math.abs(remainingBudget).toLocaleString() }}</div>
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="submitSuccess" class="success-banner">
        {{ t('restocking.orderSuccess') }}
      </div>

      <!-- Submit Error -->
      <div v-if="submitError" class="error">{{ submitError }}</div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.autoFitLabel') }}</h3>
          <button
            class="btn-primary"
            :disabled="selectedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('common.loading') : t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="enrichedForecasts.length === 0" class="loading">
          {{ t('restocking.noForecasts') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.select') }}</th>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.currentDemand') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.qtyToOrder') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in enrichedForecasts"
                :key="item.id"
                :class="{ 'row-selected': effectiveSelections.has(item.id) }"
              >
                <td>
                  <input
                    type="checkbox"
                    :checked="effectiveSelections.has(item.id)"
                    @change="toggleItem(item.id)"
                  />
                </td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.current_demand.toLocaleString() }}</td>
                <td><strong>{{ item.forecasted_demand.toLocaleString() }}</strong></td>
                <td>{{ item.qtyToOrder.toLocaleString() }}</td>
                <td>${{ item.unitCost.toLocaleString() }}</td>
                <td>${{ item.lineTotal.toLocaleString() }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const allForecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitError = ref(null)
    const submitSuccess = ref(false)

    const BUDGET_MIN = 10000
    const BUDGET_MAX = 500000
    const BUDGET_STEP = 5000
    const budget = ref(100000)

    // Map<id, boolean> tracking user checkbox overrides
    const userSelections = ref(new Map())

    // Priority order: increasing=0, stable=1, decreasing=2
    const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

    const prioritySortedForecasts = computed(() =>
      [...allForecasts.value].sort(
        (a, b) => (TREND_PRIORITY[a.trend] ?? 3) - (TREND_PRIORITY[b.trend] ?? 3)
      )
    )

    // Map from inventory SKU → item, for unit_cost lookup
    const inventoryBySku = computed(() => {
      const map = new Map()
      for (const item of inventoryItems.value) map.set(item.sku, item)
      return map
    })

    // Enrich each forecast with unitCost, qtyToOrder, lineTotal
    const enrichedForecasts = computed(() =>
      prioritySortedForecasts.value.map(f => {
        const invItem = inventoryBySku.value.get(f.item_sku)
        const unitCost = invItem ? invItem.unit_cost : 50
        const qtyToOrder = Math.max(f.forecasted_demand - f.current_demand, 50)
        return { ...f, unitCost, qtyToOrder, lineTotal: qtyToOrder * unitCost }
      })
    )

    // Greedy auto-fit: select items in priority order until budget is exceeded
    const autoFitSelections = computed(() => {
      let remaining = budget.value
      const selected = new Set()
      for (const item of enrichedForecasts.value) {
        if (item.lineTotal <= remaining) {
          selected.add(item.id)
          remaining -= item.lineTotal
        }
      }
      return selected
    })

    // Merge auto-fit defaults with explicit user overrides
    const effectiveSelections = computed(() => {
      const result = new Set(autoFitSelections.value)
      for (const [id, checked] of userSelections.value) {
        if (checked) result.add(id)
        else result.delete(id)
      }
      return result
    })

    const selectedItems = computed(() =>
      enrichedForecasts.value.filter(f => effectiveSelections.value.has(f.id))
    )

    const estimatedTotal = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.lineTotal, 0)
    )

    const remainingBudget = computed(() => budget.value - estimatedTotal.value)

    const toggleItem = (id) => {
      userSelections.value = new Map([
        ...userSelections.value,
        [id, !effectiveSelections.value.has(id)]
      ])
    }

    watch(budget, () => {
      userSelections.value = new Map()
      submitSuccess.value = false
      submitError.value = null
    })

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const filters = getCurrentFilters()
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({ warehouse: filters.warehouse, category: filters.category })
        ])
        allForecasts.value = forecastsData
        inventoryItems.value = inventoryData
        userSelections.value = new Map()
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      submitting.value = true
      submitError.value = null
      submitSuccess.value = false
      try {
        const filters = getCurrentFilters()
        const items = selectedItems.value.map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.qtyToOrder,
          unit_price: item.unitCost
        }))
        await api.createOrder({
          items,
          warehouse: filters.warehouse !== 'all' ? filters.warehouse : null,
          category: filters.category !== 'all' ? filters.category : null
        })
        submitSuccess.value = true
        userSelections.value = new Map()
      } catch (err) {
        submitError.value = t('restocking.orderError') + ': ' + err.message
      } finally {
        submitting.value = false
      }
    }

    watch([selectedLocation, selectedCategory], loadData)
    onMounted(loadData)

    return {
      t,
      loading,
      error,
      budget,
      BUDGET_MIN,
      BUDGET_MAX,
      BUDGET_STEP,
      enrichedForecasts,
      effectiveSelections,
      selectedItems,
      estimatedTotal,
      remainingBudget,
      submitting,
      submitError,
      submitSuccess,
      toggleItem,
      placeOrder
    }
  }
}
</script>

<style scoped>
.stats-grid-3 {
  grid-template-columns: repeat(3, 1fr);
}

.budget-slider {
  width: 100%;
  margin: 0.5rem 0 0.25rem;
  cursor: pointer;
  accent-color: #2563eb;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
}

.budget-meta {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
  margin-top: 0.25rem;
}

.budget-hint {
  font-size: 0.813rem;
  color: #94a3b8;
  margin-top: 0.5rem;
}

.row-selected {
  background-color: rgba(37, 99, 235, 0.08);
}

.btn-primary {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
