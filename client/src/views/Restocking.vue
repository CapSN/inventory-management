<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Slider -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div class="budget-controls">
          <div class="budget-display">
            <span class="budget-amount">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
          </div>
          <input
            v-model.number="budget"
            type="range"
            :min="0"
            :max="maxBudget"
            :step="sliderStep"
            class="budget-slider"
            :aria-label="t('restocking.budgetLabel')"
          />
          <div class="budget-range-labels">
            <span>{{ currencySymbol }}0</span>
            <span>{{ currencySymbol }}{{ maxBudget.toLocaleString() }}</span>
          </div>
        </div>
      </div>

      <!-- Summary Stats -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.selected') }}</div>
          <div class="stat-value">{{ selectedCount }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.budgetUsed') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ Math.round(totalCost).toLocaleString() }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.budgetRemaining') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ Math.round(budgetRemaining).toLocaleString() }}</div>
        </div>
      </div>

      <!-- Edge case notices -->
      <div v-if="recommendations.length === 0" class="info-notice">
        {{ t('restocking.noForecasts') }}
      </div>
      <div v-else-if="noneSelected && budget > 0" class="info-notice">
        {{ t('restocking.budgetTooLow') }}
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.totalCost') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.status') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in selectedItems"
                :key="item.id"
                :class="['recommendation-row', item.selected ? 'row-selected' : 'row-excluded']"
              >
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                <td><strong>{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</strong></td>
                <td>
                  <span :class="['badge', item.trend]">{{ t(`trends.${item.trend}`) }}</span>
                </td>
                <td>
                  <span v-if="item.selected" class="badge success">{{ t('restocking.selected') }}</span>
                  <span v-else class="badge excluded-badge">{{ t('restocking.excluded') }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order Bar -->
      <div class="place-order-bar">
        <div v-if="submitSuccess" class="success-notice">{{ t('restocking.orderPlaced') }}</div>
        <div v-if="submitError" class="error submit-error">{{ submitError }}</div>
        <button
          class="btn-primary"
          :disabled="selectedCount === 0 || submitting"
          @click="placeOrder"
        >
          <span v-if="submitting">{{ t('common.loading') }}</span>
          <span v-else>
            {{ t('restocking.placeOrder') }}
            <span v-if="selectedCount > 0"> ({{ t('restocking.itemsSelected', { count: selectedCount }) }})</span>
          </span>
        </button>
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
    const { getCurrentFilters } = useFilters()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref(0)
    const submitting = ref(false)
    const submitSuccess = ref(false)
    const submitError = ref(null)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Total cost of all items — drives the slider's max value
    const maxBudget = computed(() => {
      const total = recommendations.value.reduce((s, r) => s + r.total_cost, 0)
      return Math.ceil(total / 10000) * 10000 || 10000
    })

    // Round slider step to a clean increment relative to max
    const sliderStep = computed(() => Math.max(100, Math.round(maxBudget.value / 500) * 100))

    // Set default budget to 25% of max once data loads
    watch(maxBudget, (newMax) => {
      if (budget.value === 0 && newMax > 0) {
        budget.value = Math.round(newMax * 0.25)
      }
    }, { immediate: true })

    // Greedy algorithm: items arrive sorted by forecasted_demand desc from backend.
    // Walk in order and include each item if it fits within remaining budget.
    // Runs as a computed so it re-evaluates live as the slider moves.
    const selectedItems = computed(() => {
      let remaining = budget.value
      return recommendations.value.map(item => {
        if (item.total_cost <= remaining) {
          remaining -= item.total_cost
          return { ...item, selected: true }
        }
        return { ...item, selected: false }
      })
    })

    const selectedCount = computed(() => selectedItems.value.filter(i => i.selected).length)
    const totalCost = computed(() =>
      selectedItems.value.filter(i => i.selected).reduce((s, i) => s + i.total_cost, 0)
    )
    const budgetRemaining = computed(() => budget.value - totalCost.value)
    const noneSelected = computed(() => selectedCount.value === 0 && recommendations.value.length > 0)

    const placeOrder = async () => {
      const items = selectedItems.value.filter(i => i.selected)
      if (!items.length) return

      submitting.value = true
      submitError.value = null
      submitSuccess.value = false

      try {
        const filters = getCurrentFilters()
        await api.createRestockingOrder({
          warehouse: filters.warehouse !== 'all' ? filters.warehouse : null,
          category: filters.category !== 'all' ? filters.category : null,
          items: items.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.forecasted_demand,
            unit_price: i.unit_cost
          }))
        })
        submitSuccess.value = true
        setTimeout(() => { submitSuccess.value = false }, 4000)
      } catch (err) {
        submitError.value = t('restocking.orderFailed') + ': ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      t,
      loading,
      error,
      recommendations,
      budget,
      maxBudget,
      sliderStep,
      currencySymbol,
      selectedItems,
      selectedCount,
      totalCost,
      budgetRemaining,
      noneSelected,
      submitting,
      submitSuccess,
      submitError,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-controls {
  padding: 0.5rem 0 1rem;
}

.budget-display {
  text-align: center;
  margin-bottom: 1.25rem;
}

.budget-amount {
  font-size: 2.25rem;
  font-weight: 700;
  color: #2563eb;
}

.budget-slider {
  width: 100%;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 22px;
  height: 22px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.8rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.info-notice {
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  color: #1e40af;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

.recommendation-row.row-selected {
  background: #f0fdf4;
}

.recommendation-row.row-excluded {
  opacity: 0.45;
}

.excluded-badge {
  background: #f1f5f9;
  color: #94a3b8;
}

.place-order-bar {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.75rem;
  margin-top: 0.5rem;
  margin-bottom: 2rem;
}

.success-notice {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.75rem 1.25rem;
  border-radius: 8px;
  font-size: 0.875rem;
}

.submit-error {
  margin: 0;
  padding: 0.75rem 1.25rem;
}

.btn-primary {
  background: #2563eb;
  color: #fff;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}
</style>
