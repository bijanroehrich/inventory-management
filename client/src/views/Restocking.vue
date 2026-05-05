<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Allocate your budget to restock items based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error && !submittedOrder" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget slider card -->
      <div class="card budget-section">
        <div class="card-header">
          <h3 class="card-title">Budget Allocation</h3>
          <span class="budget-max-label">Max: {{ formatCurrency(budgetMax) }}</span>
        </div>
        <div class="budget-track">
          <input
            type="range"
            min="0"
            :max="budgetMax"
            step="1000"
            v-model.number="budget"
          />
        </div>
        <div class="budget-display">
          <span class="budget-selected">{{ formatCurrency(budget) }}</span>
          <span class="budget-of">of {{ formatCurrency(budgetMax) }}</span>
          <span class="budget-pct">({{ budgetPercent }}%)</span>
        </div>
        <!-- Visual fill bar showing percentage of total budget used -->
        <div class="budget-fill-track">
          <div
            class="budget-fill-bar"
            :style="{ width: budgetPercent + '%' }"
          ></div>
        </div>
      </div>

      <!-- Summary bar between slider and table -->
      <div class="summary-bar">
        <div class="summary-stat">
          <span class="summary-label">Items Selected</span>
          <span class="summary-value">{{ includedCount }}</span>
        </div>
        <div class="summary-divider"></div>
        <div class="summary-stat">
          <span class="summary-label">Total Cost</span>
          <span class="summary-value">{{ formatCurrency(selectedCost) }}</span>
        </div>
        <div class="summary-divider"></div>
        <div class="summary-stat">
          <span class="summary-label">Remaining Budget</span>
          <span class="summary-value" :class="{ 'remaining-zero': remainingBudget === 0 }">
            {{ formatCurrency(remainingBudget) }}
          </span>
        </div>
      </div>

      <!-- Place Order button -->
      <div class="order-action">
        <button
          class="place-order-btn"
          :disabled="includedCount === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing...' : 'Place Order' }}
        </button>
      </div>

      <!-- Success banner shown after a successful order submission -->
      <div v-if="submittedOrder" class="success-banner">
        Order {{ submittedOrder.order_number }} placed successfully.
        Expected delivery: {{ formatDeliveryDate(submittedOrder.expected_delivery) }}
      </div>

      <!-- Error shown only when order submission fails (loading error shown above) -->
      <div v-if="error && submittedOrder === null && !loading" class="error">{{ error }}</div>

      <!-- Recommendations table card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Restocking Recommendations</h3>
          <span class="table-hint">Items are ordered by demand trend. Dimmed rows fall outside your budget.</span>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th class="col-right">Current Stock</th>
                <th class="col-right">Forecasted Demand</th>
                <th class="col-right">Qty to Order</th>
                <th class="col-right">Unit Cost</th>
                <th class="col-right">Line Total</th>
                <th class="col-center">Status</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendedItems"
                :key="item.sku"
                :class="{ dimmed: !item.included }"
              >
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-right">{{ item.current_stock.toLocaleString() }}</td>
                <td class="col-right">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-right">{{ item.qty_to_order.toLocaleString() }}</td>
                <td class="col-right">{{ formatUnitCost(item.unit_cost) }}</td>
                <td class="col-right"><strong>{{ formatCurrency(item.line_total) }}</strong></td>
                <td class="col-center">
                  <span v-if="item.included" class="status-included">&#10003;</span>
                  <span v-else class="status-excluded">&ndash;</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    // useI18n imported per project convention; Restocking strings are hardcoded English
    const { t } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const allItems = ref([])
    const budget = ref(0)
    const submitting = ref(false)
    const submittedOrder = ref(null)

    // Sum of all line totals — the slider maximum
    const budgetMax = computed(() =>
      allItems.value.reduce((sum, item) => sum + item.line_total, 0)
    )

    // Percentage of budget used (capped at 100 for safety)
    const budgetPercent = computed(() => {
      if (budgetMax.value === 0) return 0
      return Math.min(100, Math.round((budget.value / budgetMax.value) * 100))
    })

    // Greedy fill: iterate allItems in API order (increasing → stable → decreasing)
    // Include items while they fit inside the budget, mark the rest excluded
    const recommendedItems = computed(() => {
      let running = 0
      return allItems.value.map(item => {
        const fits = running + item.line_total <= budget.value
        if (fits) {
          running += item.line_total
          return { ...item, included: true }
        }
        return { ...item, included: false }
      })
    })

    const includedCount = computed(() =>
      recommendedItems.value.filter(i => i.included).length
    )

    const selectedCost = computed(() =>
      recommendedItems.value
        .filter(i => i.included)
        .reduce((sum, i) => sum + i.line_total, 0)
    )

    const remainingBudget = computed(() => budget.value - selectedCost.value)

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations()
        allItems.value = data
        // Default budget to half of total so the slider is useful on first load
        budget.value = Math.floor(budgetMax.value / 2 / 1000) * 1000
      } catch (err) {
        error.value = 'Failed to load restocking recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      error.value = null
      try {
        // Strip the `included` flag before sending to the API
        const itemsToSend = recommendedItems.value
          .filter(i => i.included)
          .map(({ included, ...item }) => item)

        const result = await api.submitRestockingOrder(itemsToSend)
        submittedOrder.value = result
      } catch (err) {
        error.value = 'Failed to submit restocking order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    // --- Formatting helpers ---

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const formatUnitCost = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', minimumFractionDigits: 2, maximumFractionDigits: 2 })
    }

    const formatDeliveryDate = (dateStr) => {
      const date = new Date(dateStr)
      // Validate before formatting to avoid "Invalid Date" in the banner
      if (isNaN(date.getTime())) return dateStr
      return date.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    onMounted(loadRecommendations)

    return {
      loading,
      error,
      allItems,
      budget,
      budgetMax,
      budgetPercent,
      submitting,
      submittedOrder,
      recommendedItems,
      includedCount,
      selectedCost,
      remainingBudget,
      placeOrder,
      formatCurrency,
      formatUnitCost,
      formatDeliveryDate
    }
  }
}
</script>

<style scoped>
.restocking {
  /* Inherits .main-content padding from App.vue */
}

/* Budget card */
.budget-section {
  margin-bottom: 1rem;
}

.budget-max-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.budget-track {
  margin-bottom: 0.75rem;
}

.budget-track input[type="range"] {
  width: 100%;
  accent-color: #3b82f6;
  cursor: pointer;
  height: 6px;
}

.budget-display {
  display: flex;
  align-items: baseline;
  gap: 0.5rem;
  margin-bottom: 0.75rem;
}

.budget-selected {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-of {
  font-size: 0.938rem;
  color: #64748b;
}

.budget-pct {
  font-size: 0.875rem;
  color: #3b82f6;
  font-weight: 600;
}

/* Progress bar beneath slider */
.budget-fill-track {
  width: 100%;
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  overflow: hidden;
}

.budget-fill-bar {
  height: 100%;
  background: #3b82f6;
  border-radius: 3px;
  transition: width 0.15s ease;
}

/* Summary bar between slider and table */
.summary-bar {
  display: flex;
  align-items: center;
  gap: 0;
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 1rem 1.5rem;
  margin-bottom: 1rem;
}

.summary-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  flex: 1;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.summary-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.remaining-zero {
  color: #64748b;
}

.summary-divider {
  width: 1px;
  height: 2.5rem;
  background: #e2e8f0;
  margin: 0 1.5rem;
  flex-shrink: 0;
}

/* Place Order button row */
.order-action {
  margin-bottom: 1rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #bbf7d0;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
  margin-bottom: 1rem;
}

/* Table column alignment helpers */
.col-right {
  text-align: right;
}

.col-center {
  text-align: center;
}

/* Dimmed row for items outside budget */
.dimmed {
  opacity: 0.4;
}

/* Status column indicators */
.status-included {
  color: #059669;
  font-weight: 700;
  font-size: 1rem;
}

.status-excluded {
  color: #94a3b8;
  font-size: 1rem;
}

/* Subtle hint text in card header */
.table-hint {
  font-size: 0.8125rem;
  color: #94a3b8;
  font-weight: 400;
}
</style>
