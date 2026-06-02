<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set a budget and get prioritized restocking recommendations based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Section 1: Budget Slider -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-section">
          <div class="budget-display">
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
            <span class="budget-max">of {{ formatCurrency(totalMaxCost) }} max</span>
          </div>
          <input
            type="range"
            class="budget-slider"
            :min="0"
            :max="totalMaxCost"
            :step="100"
            v-model.number="budget"
            :disabled="totalMaxCost === 0"
          />
          <div class="budget-ticks">
            <span>{{ formatCurrency(0) }}</span>
            <span>{{ formatCurrency(totalMaxCost / 2) }}</span>
            <span>{{ formatCurrency(totalMaxCost) }}</span>
          </div>
          <p class="budget-summary">
            <strong>{{ includedCount }}</strong> of <strong>{{ forecasts.length }}</strong> items fit within budget
          </p>
        </div>
      </div>

      <!-- Section 2: Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="item-count-badge">{{ includedCount }} included</span>
        </div>
        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th class="col-num">Forecasted Qty</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Total Cost</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in sortedItems"
                :key="item.id"
                :class="{ 'row-over-budget': !item.included }"
              >
                <td>{{ item.item_name }}</td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-num">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-num"><strong>{{ formatCurrency(item.itemCost) }}</strong></td>
                <td>
                  <span v-if="item.included" class="badge success">Included</span>
                  <span v-else class="badge slate">Over Budget</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="subtotal-row">
          <span class="subtotal-label">Subtotal:</span>
          <span class="subtotal-value">{{ formatCurrency(subtotal) }}</span>
        </div>
      </div>

      <!-- Section 3: Place Order -->
      <div class="card order-card">
        <div v-if="orderSuccess" class="success-banner">
          Order submitted — {{ includedCount }} items, {{ formatCurrency(subtotal) }} total
        </div>
        <div v-if="orderError" class="error order-error">{{ orderError }}</div>
        <button
          class="place-order-btn"
          :disabled="includedCount === 0 || placing || orderSuccess"
          @click="placeOrder"
        >
          <span v-if="placing">Placing order...</span>
          <span v-else>Place Restocking Order</span>
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const forecasts = ref([])
    const loading = ref(true)
    const error = ref(null)

    const budget = ref(0)
    const placing = ref(false)
    const orderSuccess = ref(false)
    const orderError = ref(null)

    // Total cost if all forecasted items were ordered
    const totalMaxCost = computed(() => {
      return forecasts.value.reduce((sum, item) => {
        return sum + item.forecasted_demand * item.unit_cost
      }, 0)
    })

    // Recommended items with greedy budget allocation
    const recommendedItems = computed(() => {
      // Priority: increasing=1, stable=2, decreasing=3
      const trendPriority = { increasing: 1, stable: 2, decreasing: 3 }

      const sorted = [...forecasts.value]
        .map(item => ({
          ...item,
          itemCost: item.forecasted_demand * item.unit_cost
        }))
        .sort((a, b) => {
          const pa = trendPriority[a.trend] ?? 4
          const pb = trendPriority[b.trend] ?? 4
          if (pa !== pb) return pa - pb
          // Within same trend: highest itemCost first
          return b.itemCost - a.itemCost
        })

      let runningTotal = 0
      return sorted.map(item => {
        const fits = runningTotal + item.itemCost <= budget.value
        if (fits) runningTotal += item.itemCost
        return { ...item, included: fits }
      })
    })

    // Included items first (in priority order), then over-budget items
    const sortedItems = computed(() => {
      const included = recommendedItems.value.filter(i => i.included)
      const excluded = recommendedItems.value.filter(i => !i.included)
      return [...included, ...excluded]
    })

    const includedCount = computed(() =>
      recommendedItems.value.filter(i => i.included).length
    )

    const subtotal = computed(() =>
      recommendedItems.value
        .filter(i => i.included)
        .reduce((sum, i) => sum + i.itemCost, 0)
    )

    const formatCurrency = (value) => {
      if (currentCurrency.value === 'JPY') {
        return '¥' + Math.round(value).toLocaleString()
      }
      return '$' + value.toLocaleString('en-US', {
        minimumFractionDigits: 0,
        maximumFractionDigits: 0
      })
    }

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getDemandForecasts()
        forecasts.value = data
        // Default budget: 50% of max, rounded to nearest 100
        budget.value = Math.round(totalMaxCost.value * 0.5 / 100) * 100
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Setting default budget after totalMaxCost is available requires a watch
    // because totalMaxCost is computed from forecasts which load async
    watch(totalMaxCost, (val) => {
      if (budget.value === 0 && val > 0) {
        budget.value = Math.round(val * 0.5 / 100) * 100
      }
    })

    // Clear order state when budget changes
    watch(budget, () => {
      orderSuccess.value = false
      orderError.value = null
    })

    const placeOrder = async () => {
      placing.value = true
      orderError.value = null
      try {
        const includedItems = recommendedItems.value.filter(i => i.included)
        await api.createRestockingOrder({
          items: includedItems.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.forecasted_demand,
            unit_cost: i.unit_cost
          })),
          total_value: subtotal.value,
          budget: budget.value
        })
        orderSuccess.value = true
      } catch (err) {
        orderError.value = 'Failed to place order: ' + (err.response?.data?.detail || err.message)
        console.error(err)
      } finally {
        placing.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      forecasts,
      loading,
      error,
      budget,
      totalMaxCost,
      recommendedItems,
      sortedItems,
      includedCount,
      subtotal,
      placing,
      orderSuccess,
      orderError,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* Budget section */
.budget-section {
  padding: 0.5rem 0;
}

.budget-display {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
  margin-bottom: 1rem;
}

.budget-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-max {
  font-size: 0.938rem;
  color: #64748b;
}

.budget-slider {
  width: 100%;
  height: 6px;
  appearance: none;
  -webkit-appearance: none;
  border-radius: 4px;
  background: #e2e8f0;
  outline: none;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-slider:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
}

.budget-ticks {
  display: flex;
  justify-content: space-between;
  margin-top: 0.375rem;
  font-size: 0.75rem;
  color: #94a3b8;
}

.budget-summary {
  margin-top: 1rem;
  font-size: 0.938rem;
  color: #475569;
}

/* Table */
.restocking-table {
  table-layout: auto;
  width: 100%;
}

.col-num {
  text-align: right;
}

.row-over-budget {
  opacity: 0.45;
}

/* Slate badge for over-budget */
.badge.slate {
  background: #f1f5f9;
  color: #475569;
}

/* Item count badge in card header */
.item-count-badge {
  font-size: 0.813rem;
  font-weight: 600;
  color: #065f46;
  background: #d1fae5;
  padding: 0.25rem 0.625rem;
  border-radius: 5px;
}

/* Subtotal */
.subtotal-row {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 1rem;
  padding: 0.875rem 0.75rem 0;
  border-top: 2px solid #e2e8f0;
  margin-top: 0.5rem;
}

.subtotal-label {
  font-size: 0.938rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.subtotal-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

/* Order section */
.order-card {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
}

.order-error {
  margin: 0;
}

.place-order-btn {
  align-self: flex-start;
  padding: 0.75rem 1.75rem;
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
