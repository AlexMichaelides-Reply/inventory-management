<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    // --- Reactive state ---
    const budget = ref(25000)
    const allForecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(true)
    const error = ref(null)
    const orderSubmitting = ref(false)
    const orderSubmitted = ref(false)
    const submittedOrderNumber = ref(null)

    // --- Computed: lookup map for O(1) joins ---
    const inventoryBySku = computed(() => {
      return new Map(inventoryItems.value.map(i => [i.sku, i]))
    })

    // --- Computed: join forecasts with inventory to get unit_cost ---
    // Skips forecasts with no matching inventory record (no cost basis)
    const enrichedForecasts = computed(() => {
      return allForecasts.value.reduce((acc, forecast) => {
        const inv = inventoryBySku.value.get(forecast.item_sku)
        if (!inv) return acc // skip unmatched — no cost basis available

        const restock_qty = Math.max(0, forecast.forecasted_demand - forecast.current_demand)
        const unit_cost = inv.unit_cost
        const line_total = restock_qty * unit_cost

        acc.push({
          ...forecast,
          sku: forecast.item_sku,
          restock_qty,
          unit_cost,
          line_total
        })
        return acc
      }, [])
    })

    // --- Computed: items that actually need restocking, sorted by priority ---
    // Sort: increasing trend first (most urgent), then by demand gap descending
    // within the same trend bucket so highest-gap items are prioritized
    const trendScore = { increasing: 0, stable: 1, decreasing: 2 }

    const candidateItems = computed(() => {
      return enrichedForecasts.value
        .filter(item => item.restock_qty > 0)
        .slice()
        .sort((a, b) => {
          const trendDiff = (trendScore[a.trend] ?? 1) - (trendScore[b.trend] ?? 1)
          if (trendDiff !== 0) return trendDiff
          // Same trend: sort by demand gap descending (larger gap = higher priority)
          const gapA = a.forecasted_demand - a.current_demand
          const gapB = b.forecasted_demand - b.current_demand
          return gapB - gapA
        })
    })

    // --- Computed: greedy fill within budget ---
    // We walk candidates in priority order and include each item only if it fits
    // entirely within the remaining budget. Partial quantities are not supported
    // because a partial order may not satisfy the restock need (e.g., minimum
    // run sizes, shipping thresholds). Items whose full line_total exceeds the
    // total budget are skipped entirely — they can never be included.
    const recommendations = computed(() => {
      const result = []
      let running = 0

      for (const item of candidateItems.value) {
        // Skip items that can never fit regardless of what else is selected
        if (item.line_total > budget.value) continue

        if (running + item.line_total <= budget.value) {
          result.push(item)
          running += item.line_total
        }
        // If it doesn't fit now but could fit later (budget not exceeded by the
        // item alone), we still skip it — greedy in insertion order ensures the
        // highest-priority items always claim the budget first.
      }

      return result
    })

    // --- Budget summary computeds ---
    const allocatedTotal = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.line_total, 0)
    )

    const remainingBudget = computed(() => budget.value - allocatedTotal.value)

    const utilizationPercent = computed(() =>
      Math.min(100, (allocatedTotal.value / budget.value) * 100)
    )

    // --- Reset success state when budget changes (recommendations may differ) ---
    watch(budget, () => {
      orderSubmitted.value = false
      submittedOrderNumber.value = null
    })

    // --- Data loading ---
    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        // Fetch both in parallel; restocking is global — no warehouse/category filters
        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        allForecasts.value = forecastsData
        inventoryItems.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load restocking data'
        console.error('Restocking loadData error:', err)
      } finally {
        loading.value = false
      }
    }

    onMounted(loadData)

    // --- Place order ---
    const placeOrder = async () => {
      orderSubmitting.value = true
      try {
        const payload = {
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.item_name,
            quantity: r.restock_qty,
            unit_price: r.unit_cost
          })),
          total_value: allocatedTotal.value
        }
        const response = await api.createRestockingOrder(payload)
        orderSubmitted.value = true
        submittedOrderNumber.value = response.order_number
      } catch (err) {
        error.value = 'Failed to submit order. Please try again.'
        console.error('placeOrder error:', err)
      } finally {
        orderSubmitting.value = false
      }
    }

    return {
      budget,
      loading,
      error,
      orderSubmitting,
      orderSubmitted,
      submittedOrderNumber,
      candidateItems,
      recommendations,
      allocatedTotal,
      remainingBudget,
      utilizationPercent,
      placeOrder
    }
  }
}
</script>

<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set a budget and get recommended items to restock based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- 1. Budget card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-body">
          <div class="budget-label">
            <span class="budget-amount">${{ budget.toLocaleString() }}</span>
            <span class="budget-range">$1,000 – $100,000</span>
          </div>
          <input
            type="range"
            min="1000"
            max="100000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="utilization-bar">
            <div
              class="utilization-fill"
              :class="{ warn: utilizationPercent > 90 }"
              :style="{ width: utilizationPercent + '%' }"
            ></div>
          </div>
          <div class="budget-stats">
            <div class="budget-stat">
              <span class="stat-label-sm">Allocated</span>
              <span class="stat-value-sm">${{ allocatedTotal.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}</span>
            </div>
            <div class="budget-stat">
              <span class="stat-label-sm">Remaining</span>
              <span class="stat-value-sm">${{ remainingBudget.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}</span>
            </div>
            <div class="budget-stat">
              <span class="stat-label-sm">Utilized</span>
              <span class="stat-value-sm">{{ utilizationPercent.toFixed(1) }}%</span>
            </div>
          </div>
        </div>
      </div>

      <!-- 2. Recommendations card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendations.length }})</h3>
          <p class="card-subtitle" v-if="candidateItems.length > recommendations.length">
            {{ candidateItems.length - recommendations.length }} items excluded — increase budget to include them
          </p>
        </div>
        <div v-if="recommendations.length === 0" class="empty-state">
          <p>No items qualify for restocking at this budget.</p>
          <p>Increase the budget or check the Demand Forecast tab for details.</p>
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th class="text-right">Current Demand</th>
                <th class="text-right">Forecasted</th>
                <th class="text-right">Restock Qty</th>
                <th class="text-right">Unit Cost</th>
                <th class="text-right">Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td>{{ item.item_name }}</td>
                <td class="sku">{{ item.sku }}</td>
                <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                <td class="text-right">{{ item.current_demand.toLocaleString() }}</td>
                <td class="text-right">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="text-right"><strong>{{ item.restock_qty.toLocaleString() }}</strong></td>
                <td class="text-right">${{ item.unit_cost.toFixed(2) }}</td>
                <td class="text-right"><strong>${{ item.line_total.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="total-row">
                <td colspan="7" class="text-right total-label">Total Order Value</td>
                <td class="text-right total-value">${{ allocatedTotal.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}</td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>

      <!-- 3. Place Order -->
      <div class="order-actions">
        <button
          class="btn-place-order"
          :disabled="recommendations.length === 0 || orderSubmitting || orderSubmitted"
          @click="placeOrder"
        >
          {{ orderSubmitting ? 'Submitting...' : orderSubmitted ? 'Order Placed' : 'Place Order' }}
        </button>
      </div>

      <!-- 4. Success state -->
      <div v-if="orderSubmitted" class="card success-card">
        <div class="success-content">
          <div class="success-icon">&#10003;</div>
          <div>
            <strong>{{ submittedOrderNumber }} submitted successfully.</strong>
            <p>Expected delivery in 14 days. View progress in the Orders tab.</p>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
.budget-body { padding: 1.5rem; display: flex; flex-direction: column; gap: 1rem; }
.budget-label { display: flex; justify-content: space-between; align-items: baseline; }
.budget-amount { font-size: 1.875rem; font-weight: 700; color: #0f172a; }
.budget-range { font-size: 0.813rem; color: #94a3b8; }
.budget-slider { width: 100%; accent-color: #2563eb; cursor: pointer; height: 4px; }
.utilization-bar { height: 8px; background: #e2e8f0; border-radius: 4px; overflow: hidden; }
.utilization-fill { height: 100%; background: #2563eb; border-radius: 4px; transition: width 0.3s ease; }
.utilization-fill.warn { background: #ef4444; }
.budget-stats { display: flex; gap: 2rem; }
.budget-stat { display: flex; flex-direction: column; gap: 0.25rem; }
.stat-label-sm { font-size: 0.75rem; color: #64748b; text-transform: uppercase; letter-spacing: 0.05em; font-weight: 600; }
.stat-value-sm { font-size: 0.938rem; font-weight: 600; color: #0f172a; }
.card-subtitle { font-size: 0.813rem; color: #94a3b8; margin-top: 0.25rem; }
.sku { font-family: monospace; font-size: 0.813rem; color: #64748b; }
.text-right { text-align: right; }
.total-row td { border-top: 2px solid #e2e8f0; padding-top: 0.75rem; background: #f8fafc; }
.total-label { font-weight: 600; color: #475569; font-size: 0.875rem; text-transform: uppercase; letter-spacing: 0.05em; }
.total-value { font-size: 1rem; font-weight: 700; color: #0f172a; }
.empty-state { padding: 3rem 1.5rem; text-align: center; color: #64748b; }
.empty-state p + p { margin-top: 0.5rem; font-size: 0.875rem; }
.order-actions { display: flex; justify-content: flex-end; margin: 0.5rem 0 1.25rem; }
.btn-place-order { background: #2563eb; color: white; border: none; border-radius: 8px; padding: 0.75rem 2rem; font-size: 0.938rem; font-weight: 600; cursor: pointer; transition: background 0.2s; }
.btn-place-order:hover:not(:disabled) { background: #1d4ed8; }
.btn-place-order:disabled { background: #94a3b8; cursor: not-allowed; }
.success-card { border-left: 4px solid #10b981; }
.success-content { display: flex; align-items: flex-start; gap: 1rem; padding: 1.5rem; }
.success-icon { width: 2rem; height: 2rem; border-radius: 50%; background: #10b981; color: white; display: flex; align-items: center; justify-content: center; font-weight: 700; flex-shrink: 0; }
.success-content strong { color: #065f46; }
.success-content p { font-size: 0.875rem; color: #64748b; margin-top: 0.25rem; }
</style>
