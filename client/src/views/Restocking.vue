<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Allocate your restocking budget to the highest-priority items based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-slider-row">
          <input
            type="range"
            v-model.number="budget"
            min="0"
            max="500000"
            step="1000"
            class="budget-slider"
          />
          <span class="budget-value">{{ formatCurrency(budget, currentCurrency) }}</span>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendations.length }})</h3>
        </div>
        <div v-if="recommendations.length === 0" class="empty-state">
          No items to restock within this budget.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td>{{ item.name }}</td>
                <td><code>{{ item.sku }}</code></td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.restock_qty.toLocaleString() }}</td>
                <td>{{ formatCurrency(item.unit_cost, currentCurrency) }}</td>
                <td><strong>{{ formatCurrency(item.total_cost, currentCurrency) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="card">
        <div class="summary-row">
          <div class="summary-left">
            <div class="summary-stat">
              <span class="summary-label">Total Cost:</span>
              <strong>{{ formatCurrency(totalCost, currentCurrency) }}</strong>
            </div>
            <div class="summary-stat">
              <span class="summary-label">Remaining Budget:</span>
              <strong>{{ formatCurrency(budget - totalCost, currentCurrency) }}</strong>
            </div>
            <div class="progress-track">
              <div
                class="progress-fill"
                :style="{ width: budgetUsedPercent + '%' }"
              ></div>
            </div>
            <div class="progress-label">{{ budgetUsedPercent }}% of budget used</div>
          </div>
          <div class="summary-right">
            <span v-if="submitSuccess" class="badge success">Order placed successfully</span>
            <div v-if="submitError" class="submit-error">{{ submitError }}</div>
            <button
              class="place-order-btn"
              :disabled="recommendations.length === 0 || submitting"
              @click="placeOrder"
            >
              {{ submitting ? 'Submitting...' : 'Place Order' }}
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const budget = ref(50000)
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitError = ref(null)
    const submitSuccess = ref(false)
    const demandForecasts = ref([])
    const inventoryItems = ref([])

    const inventoryBySku = computed(() => {
      const map = new Map()
      for (const item of inventoryItems.value) {
        map.set(item.sku, item)
      }
      return map
    })

    const recommendations = computed(() => {
      const candidates = []

      for (const forecast of demandForecasts.value) {
        const gap = forecast.forecasted_demand - forecast.current_demand
        if (gap <= 0) continue

        const inventoryItem = inventoryBySku.value.get(forecast.item_sku)
        if (!inventoryItem) continue

        const item_total_cost = gap * inventoryItem.unit_cost

        candidates.push({
          sku: forecast.item_sku,
          name: forecast.item_name,
          trend: forecast.trend,
          restock_qty: gap,
          unit_cost: inventoryItem.unit_cost,
          total_cost: item_total_cost,
          category: inventoryItem.category
        })
      }

      // Sort: "increasing" trend first, then by gap descending
      candidates.sort((a, b) => {
        if (a.trend === 'increasing' && b.trend !== 'increasing') return -1
        if (a.trend !== 'increasing' && b.trend === 'increasing') return 1
        return b.restock_qty - a.restock_qty
      })

      // Greedy selection within budget
      let remaining = budget.value
      const selected = []
      for (const item of candidates) {
        if (item.total_cost <= remaining) {
          selected.push(item)
          remaining -= item.total_cost
        }
      }

      return selected
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.total_cost, 0)
    )

    const budgetUsedPercent = computed(() =>
      budget.value === 0 ? 0 : Math.min(100, Math.round((totalCost.value / budget.value) * 100))
    )

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      submitError.value = null
      try {
        await api.createRestockingOrder({
          items: recommendations.value.map(item => ({
            sku: item.sku,
            name: item.name,
            restock_qty: item.restock_qty,
            unit_cost: item.unit_cost,
            total_cost: item.total_cost
          })),
          total_cost: totalCost.value
        })
        submitSuccess.value = true
        setTimeout(() => { submitSuccess.value = false }, 3000)
      } catch (err) {
        submitError.value = err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      budget,
      loading,
      error,
      submitting,
      submitError,
      submitSuccess,
      recommendations,
      totalCost,
      budgetUsedPercent,
      currentCurrency,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  padding: 0.5rem 0;
}

.budget-slider {
  flex: 1;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 120px;
  text-align: right;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 2rem;
}

.summary-left {
  flex: 1;
}

.summary-stat {
  display: flex;
  gap: 0.5rem;
  align-items: baseline;
  margin-bottom: 0.5rem;
  font-size: 0.938rem;
  color: #0f172a;
}

.summary-label {
  color: #64748b;
}

.progress-track {
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
  margin-top: 0.75rem;
  margin-bottom: 0.375rem;
}

.progress-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 4px;
  transition: width 0.3s ease;
}

.progress-label {
  font-size: 0.813rem;
  color: #64748b;
}

.summary-right {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.75rem;
}

.submit-error {
  font-size: 0.875rem;
  color: #ef4444;
  max-width: 280px;
  text-align: right;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.5rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* Trend badge colours — complement global .badge base */
:deep(.badge.increasing) {
  background: #d1fae5;
  color: #059669;
}

:deep(.badge.stable) {
  background: #dbeafe;
  color: #2563eb;
}

:deep(.badge.decreasing) {
  background: #fef9c3;
  color: #b45309;
}
</style>
