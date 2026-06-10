<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>AI-recommended restocking based on backlog priority and demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Success banner -->
      <div v-if="submittedOrder" class="banner-success">
        Order {{ submittedOrder.order_number }} submitted! Expected delivery: {{ formatDeliveryDate(submittedOrder.expected_delivery) }}. View it in the Orders tab.
      </div>

      <!-- Error banner -->
      <div v-if="submitError" class="banner-error">{{ submitError }}</div>

      <!-- Budget section -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-body">
          <div class="budget-value">${{ budget.toLocaleString() }}</div>
          <input
            v-model.number="budget"
            type="range"
            min="0"
            max="50000"
            step="500"
            class="budget-slider"
          />
          <div class="budget-bar-row">
            <div class="budget-bar-track">
              <div
                class="budget-bar-fill"
                :class="{ 'budget-bar-danger': budgetPercent > 95 }"
                :style="{ width: Math.min(budgetPercent, 100) + '%' }"
              ></div>
            </div>
            <span class="budget-bar-label">{{ budgetPercent.toFixed(1) }}% used</span>
          </div>
          <div class="budget-summary">
            {{ recommendedItems.length }} items recommended
            &middot; ${{ budgetUsed.toLocaleString() }} total
            &middot; ${{ budgetRemaining.toLocaleString() }} remaining
          </div>
        </div>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          Increase budget to see recommendations
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Source</th>
                <th>Priority / Trend</th>
                <th style="text-align: right;">Qty</th>
                <th style="text-align: right;">Unit Cost</th>
                <th style="text-align: right;">Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td>{{ item.name }}</td>
                <td><code class="sku-code">{{ item.sku }}</code></td>
                <td>
                  <span :class="['badge', getSourceBadgeClass(item)]">
                    {{ item.source_label }}<span v-if="item.partial" class="partial-label"> (partial)</span>
                  </span>
                </td>
                <td>
                  <span v-if="item.source === 'backlog'" :class="['badge', item.priority]">
                    {{ item.priority }}
                  </span>
                  <span v-else class="badge increasing">
                    Increasing
                  </span>
                </td>
                <td style="text-align: right;">{{ item.quantity.toLocaleString() }}</td>
                <td style="text-align: right;">${{ item.unit_cost.toLocaleString() }}</td>
                <td style="text-align: right;"><strong>${{ item.total_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="totals-row">
                <td colspan="4"><strong>Total</strong></td>
                <td style="text-align: right;"><strong>{{ totalQuantity.toLocaleString() }}</strong></td>
                <td></td>
                <td style="text-align: right;"><strong>${{ budgetUsed.toLocaleString() }}</strong></td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(10000)
    const backlogData = ref([])
    const demandData = ref([])
    const inventoryData = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitError = ref(null)
    const submittedOrder = ref(null)

    const budgetUsed = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const budgetRemaining = computed(() => {
      return Math.max(0, budget.value - budgetUsed.value)
    })

    const budgetPercent = computed(() => {
      if (budget.value === 0) return 0
      return (budgetUsed.value / budget.value) * 100
    })

    const totalQuantity = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.quantity, 0)
    })

    const recommendedItems = computed(() => {
      if (budget.value === 0) return []

      const results = []
      let remaining = budget.value

      // Build inventory lookup map sku -> unit_cost
      const inventoryMap = {}
      for (const inv of inventoryData.value) {
        inventoryMap[inv.sku] = inv.unit_cost
      }

      const includedSkus = new Set()

      // Priority sort order
      const priorityOrder = { high: 0, medium: 1, low: 2 }

      // Phase 1 — Backlog items sorted by priority then days_delayed desc
      const sortedBacklog = [...backlogData.value].sort((a, b) => {
        const pa = priorityOrder[a.priority] ?? 3
        const pb = priorityOrder[b.priority] ?? 3
        if (pa !== pb) return pa - pb
        return (b.days_delayed || 0) - (a.days_delayed || 0)
      })

      for (const item of sortedBacklog) {
        if (remaining <= 0) break

        const unit_cost = inventoryMap[item.item_sku]
        if (!unit_cost || unit_cost === 0) continue

        const fullCost = item.quantity_needed * unit_cost

        let quantity = 0
        let partial = false

        if (fullCost <= remaining) {
          quantity = item.quantity_needed
        } else {
          quantity = Math.floor(remaining / unit_cost)
          if (quantity === 0) continue
          partial = true
        }

        const total_cost = quantity * unit_cost
        remaining -= total_cost

        const priority = item.priority
        const source_label =
          priority === 'high' ? 'Backlog — High Priority' :
          priority === 'medium' ? 'Backlog — Medium' :
          'Backlog — Low'

        results.push({
          sku: item.item_sku,
          name: item.item_name,
          quantity,
          unit_cost,
          total_cost,
          source: 'backlog',
          source_label,
          priority,
          days_delayed: item.days_delayed,
          partial
        })

        includedSkus.add(item.item_sku)
      }

      // Phase 2 — Demand forecast items with increasing trend
      const increasingForecasts = demandData.value
        .filter(f => f.trend === 'increasing')
        .sort((a, b) => {
          const diffA = (a.forecasted_demand || 0) - (a.current_demand || 0)
          const diffB = (b.forecasted_demand || 0) - (b.current_demand || 0)
          return diffB - diffA
        })

      for (const forecast of increasingForecasts) {
        if (remaining <= 0) break
        if (includedSkus.has(forecast.item_sku)) continue

        const unit_cost = inventoryMap[forecast.item_sku]
        if (!unit_cost || unit_cost === 0) continue

        const qty = (forecast.forecasted_demand || 0) - (forecast.current_demand || 0)
        if (qty <= 0) continue

        const fullCost = qty * unit_cost
        let quantity = 0
        let partial = false

        if (fullCost <= remaining) {
          quantity = qty
        } else {
          quantity = Math.floor(remaining / unit_cost)
          if (quantity === 0) continue
          partial = true
        }

        const total_cost = quantity * unit_cost
        remaining -= total_cost

        results.push({
          sku: forecast.item_sku,
          name: forecast.item_name,
          quantity,
          unit_cost,
          total_cost,
          source: 'demand',
          source_label: 'Demand Forecast',
          partial
        })

        includedSkus.add(forecast.item_sku)
      }

      return results
    })

    const getSourceBadgeClass = (item) => {
      if (item.source === 'demand') return 'info'
      if (item.priority === 'high') return 'danger'
      return 'warning'
    }

    const formatDeliveryDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [backlog, demand, inventory] = await Promise.all([
          api.getBacklog(),
          api.getDemandForecasts(),
          api.getInventory()
        ])
        backlogData.value = backlog
        demandData.value = demand
        inventoryData.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return

      submitting.value = true
      submitError.value = null
      submittedOrder.value = null

      try {
        const order = await api.createRestockingOrder({
          items: recommendedItems.value.map(i => ({
            sku: i.sku,
            name: i.name,
            quantity: i.quantity,
            unit_cost: i.unit_cost,
            source: i.source
          }))
        })
        submittedOrder.value = order
        // Reset to 0 (not 10000) so recommendedItems clears and the button stays
        // disabled — prevents accidental re-submission before the user re-engages the slider
        budget.value = 0
      } catch (err) {
        submitError.value = 'Failed to submit order: ' + err.message
        console.error(err)
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
      submittedOrder,
      recommendedItems,
      budgetUsed,
      budgetRemaining,
      budgetPercent,
      totalQuantity,
      getSourceBadgeClass,
      formatDeliveryDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

/* Budget section body */
.budget-body {
  padding: 0.5rem 0;
}

.budget-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 1rem;
}

/* Slider */
.budget-slider {
  width: 100%;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  margin-bottom: 1rem;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 0 0 2px #2563eb;
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 0 0 2px #2563eb;
}

/* Budget bar */
.budget-bar-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.75rem;
}

.budget-bar-track {
  flex: 1;
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}

.budget-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 4px;
  transition: width 0.3s ease, background-color 0.3s ease;
}

.budget-bar-fill.budget-bar-danger {
  background: #dc2626;
}

.budget-bar-label {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 600;
  white-space: nowrap;
}

.budget-summary {
  font-size: 0.875rem;
  color: #64748b;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 2.5rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* Place order button */
.place-order-btn {
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* Banners */
.banner-success {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.banner-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

/* SKU code */
.sku-code {
  font-family: 'Menlo', 'Monaco', 'Courier New', monospace;
  font-size: 0.813rem;
  color: #475569;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
}

/* Partial label */
.partial-label {
  font-weight: 400;
  color: #64748b;
  text-transform: none;
  letter-spacing: 0;
}

/* Totals row */
tfoot .totals-row td {
  border-top: 2px solid #e2e8f0;
  background: #f8fafc;
  color: #0f172a;
  font-size: 0.875rem;
}
</style>
