<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your budget and select items to restock based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>

    <div v-else-if="orderPlaced" class="success-card">
      <div class="success-icon">
        <svg viewBox="0 0 64 64" fill="none" xmlns="http://www.w3.org/2000/svg">
          <circle cx="32" cy="32" r="30" stroke="currentColor" stroke-width="3" fill="none"/>
          <path d="M18 32l10 10 18-20" stroke="currentColor" stroke-width="3" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
      </div>
      <h3 class="success-heading">Order Submitted</h3>
      <p class="success-order-number">{{ submittedOrder.order_number }}</p>
      <p class="success-delivery">Expected delivery in 14 days</p>
      <p class="success-total">
        Total: {{ formatCurrency(submittedOrder.total_cost) }} across {{ submittedOrder.items ? submittedOrder.items.length : submittedOrder.item_count }} items
      </p>
      <button class="btn-primary" @click="resetForm">Place Another Order</button>
    </div>

    <div v-else>
      <!-- Budget section -->
      <div class="card budget-card">
        <div class="budget-label">Available Budget</div>
        <div class="budget-amount">{{ formatCurrency(budget) }}</div>
        <input
          type="range"
          min="1000"
          max="100000"
          step="1000"
          v-model.number="budget"
          class="budget-slider"
        />
        <div class="budget-stats">
          <div class="budget-stat">
            <div class="budget-stat-label">Budget</div>
            <div class="budget-stat-value">{{ formatCurrency(budget) }}</div>
          </div>
          <div class="budget-stat">
            <div class="budget-stat-label">Selected</div>
            <div class="budget-stat-value" :class="{ 'over-budget': isOverBudget }">
              {{ formatCurrency(totalSelectedCost) }}
            </div>
          </div>
          <div class="budget-stat">
            <div class="budget-stat-label">Remaining</div>
            <div class="budget-stat-value" :class="{ 'over-budget': isOverBudget }">
              {{ formatCurrency(budget - totalSelectedCost) }}
            </div>
          </div>
        </div>
        <div class="progress-track">
          <div
            class="progress-fill"
            :class="{ 'progress-over': isOverBudget }"
            :style="{ width: Math.min(totalSelectedCost / budget * 100, 100) + '%' }"
          ></div>
        </div>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="count-badge">{{ selectedCount }} / {{ recommendations.length }}</span>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items match the restocking criteria.
        </div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th class="col-checkbox"></th>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Category</th>
                <th>Warehouse</th>
                <th>Stock Status</th>
                <th>Demand Trend</th>
                <th>On Hand / Reorder</th>
                <th>Rec. Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'row-selected': item.selected }"
              >
                <td class="col-checkbox">
                  <input type="checkbox" v-model="item.selected" />
                </td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.category }}</td>
                <td>{{ item.warehouse }}</td>
                <td>
                  <span
                    v-if="item.quantity_on_hand === 0"
                    class="badge danger"
                  >Out of Stock</span>
                  <span
                    v-else-if="item.quantity_on_hand <= item.reorder_point"
                    class="badge warning"
                  >Low Stock</span>
                  <span v-else class="badge success">In Stock</span>
                </td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td :class="{ 'text-danger': item.quantity_on_hand <= item.reorder_point }">
                  {{ item.quantity_on_hand }} / {{ item.reorder_point }}
                </td>
                <td>{{ item.recommended_qty }}</td>
                <td>${{ item.unit_cost.toFixed(2) }}</td>
                <td :class="{ 'text-muted': !item.selected }">
                  {{ formatCurrency(item.total_cost) }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order footer -->
      <div class="order-footer">
        <div class="order-summary">
          {{ selectedCount }} items selected &middot; Total: {{ formatCurrency(totalSelectedCost) }}
        </div>
        <div class="order-actions">
          <p v-if="isOverBudget" class="over-budget-hint">Reduce selection to fit within budget</p>
          <button
            class="btn-primary"
            :disabled="selectedCount === 0 || isOverBudget || placing"
            @click="placeOrder"
          >
            {{ placing ? 'Placing Order...' : 'Place Order' }}
          </button>
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
    const loading = ref(true)
    const error = ref(null)
    const budget = ref(50000)
    const recommendations = ref([])
    const orderPlaced = ref(false)
    const submittedOrder = ref(null)
    const placing = ref(false)

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])

        const invMap = {}
        inventory.forEach(item => { invMap[item.sku] = item })

        const items = forecasts
          .filter(f => f.trend === 'increasing' && invMap[f.item_sku])
          .map(f => {
            const inv = invMap[f.item_sku]
            const isLowStock = inv.quantity_on_hand <= inv.reorder_point
            const recommendedQty = Math.max(
              inv.reorder_point * 2 - inv.quantity_on_hand,
              f.forecasted_demand
            )
            const totalCost = recommendedQty * inv.unit_cost
            return {
              sku: inv.sku,
              name: inv.name,
              category: inv.category,
              warehouse: inv.warehouse,
              quantity_on_hand: inv.quantity_on_hand,
              reorder_point: inv.reorder_point,
              unit_cost: inv.unit_cost,
              trend: f.trend,
              forecasted_demand: f.forecasted_demand,
              recommended_qty: recommendedQty,
              total_cost: parseFloat(totalCost.toFixed(2)),
              is_low_stock: isLowStock,
              priority: isLowStock ? 1 : 2,
              selected: isLowStock
            }
          })
          .sort((a, b) => a.priority - b.priority || b.total_cost - a.total_cost)

        recommendations.value = items
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const selectedItems = computed(() => recommendations.value.filter(i => i.selected))
    const selectedCount = computed(() => selectedItems.value.length)
    const totalSelectedCost = computed(() => selectedItems.value.reduce((sum, i) => sum + i.total_cost, 0))
    const isOverBudget = computed(() => totalSelectedCost.value > budget.value)

    const formatCurrency = (amount) => {
      return '$' + Math.round(amount).toLocaleString()
    }

    const placeOrder = async () => {
      if (selectedCount.value === 0 || isOverBudget.value) return
      try {
        placing.value = true
        const orderItems = selectedItems.value.map(i => ({
          sku: i.sku,
          name: i.name,
          category: i.category,
          warehouse: i.warehouse,
          quantity: i.recommended_qty,
          unit_cost: i.unit_cost,
          total_cost: i.total_cost
        }))
        const result = await api.createRestockingOrder({ items: orderItems, budget: budget.value })
        submittedOrder.value = result
        orderPlaced.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    const resetForm = () => {
      orderPlaced.value = false
      submittedOrder.value = null
      recommendations.value.forEach(item => { item.selected = item.is_low_stock })
    }

    onMounted(loadData)

    return {
      loading,
      error,
      budget,
      recommendations,
      orderPlaced,
      submittedOrder,
      placing,
      selectedCount,
      totalSelectedCost,
      isOverBudget,
      formatCurrency,
      placeOrder,
      resetForm
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Budget card */
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.375rem;
}

.budget-amount {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  accent-color: #3b82f6;
  margin-bottom: 1.25rem;
  cursor: pointer;
}

.budget-stats {
  display: flex;
  gap: 2rem;
  margin-bottom: 1rem;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-stat-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-stat-value.over-budget {
  color: #ef4444;
}

/* Progress bar */
.progress-track {
  height: 8px;
  border-radius: 4px;
  background: #f1f5f9;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  border-radius: 4px;
  background: #3b82f6;
  transition: width 0.3s ease, background-color 0.2s ease;
}

.progress-fill.progress-over {
  background: #ef4444;
}

/* Recommendations table */
.col-checkbox {
  width: 40px;
  text-align: center;
}

.row-selected {
  background: #f0f9ff;
}

.row-selected:hover {
  background: #e0f2fe !important;
}

.text-danger {
  color: #ef4444;
  font-weight: 600;
}

.text-muted {
  color: #94a3b8;
}

/* Count badge in card header */
.count-badge {
  display: inline-flex;
  align-items: center;
  background: #eff6ff;
  color: #1d4ed8;
  border-radius: 999px;
  font-size: 0.75rem;
  font-weight: 600;
  padding: 0.15rem 0.6rem;
}

/* Empty state */
.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Place Order footer */
.order-footer {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 1.25rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  position: sticky;
  bottom: 1.5rem;
  box-shadow: 0 -2px 12px rgba(0, 0, 0, 0.06);
}

.order-summary {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
}

.order-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.over-budget-hint {
  font-size: 0.813rem;
  color: #ef4444;
  font-weight: 500;
}

.btn-primary {
  background: #3b82f6;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Success card */
.success-card {
  background: #ecfdf5;
  border: 1px solid #d1fae5;
  border-radius: 12px;
  padding: 3rem;
  text-align: center;
  max-width: 520px;
  margin: 2rem auto;
}

.success-icon {
  width: 64px;
  height: 64px;
  color: #10b981;
  margin: 0 auto 1.25rem;
}

.success-icon svg {
  width: 100%;
  height: 100%;
}

.success-heading {
  font-size: 1.5rem;
  font-weight: 700;
  color: #065f46;
  margin-bottom: 0.5rem;
  letter-spacing: -0.025em;
}

.success-order-number {
  font-size: 1rem;
  font-weight: 600;
  color: #047857;
  margin-bottom: 0.375rem;
  font-family: 'Courier New', monospace;
}

.success-delivery {
  font-size: 0.938rem;
  color: #065f46;
  margin-bottom: 0.375rem;
}

.success-total {
  font-size: 0.938rem;
  color: #065f46;
  margin-bottom: 1.5rem;
  font-weight: 600;
}
</style>
