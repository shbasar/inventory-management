<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your available budget to get restocking recommendations based on demand forecasts.</p>
    </div>

    <!-- Budget Card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
      </div>
      <div class="budget-control">
        <div class="budget-display">${{ budget.toLocaleString() }}</div>
        <input
          type="range"
          v-model.number="budget"
          min="0"
          max="100000"
          step="1000"
          class="budget-slider"
        />
        <div class="budget-range-labels">
          <span>$0</span>
          <span>$100,000</span>
        </div>
      </div>
    </div>

    <!-- Recommendations Card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items ({{ recommendations.length }})</h3>
      </div>
      <p class="card-subtitle">Sorted by highest forecasted demand. Items are added until budget is reached.</p>

      <div v-if="loading" class="loading">Loading forecasts...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else>
        <div v-if="recommendations.length === 0" class="empty-state">
          No items fit within the current budget.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item SKU</th>
                <th>Item Name</th>
                <th>Forecasted Demand</th>
                <th>Unit Cost</th>
                <th>Qty to Order</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td>${{ item.unit_cost.toLocaleString() }}</td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td><strong>${{ item.total_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="total-row">
                <td colspan="5"><strong>Total</strong></td>
                <td><strong>${{ totalCost.toLocaleString() }}</strong></td>
              </tr>
            </tfoot>
          </table>
        </div>

        <!-- Success Banner -->
        <div v-if="submitted" class="success-banner">
          Order placed successfully. Expected delivery in 7 days.
          <router-link to="/orders" class="success-link">View in Orders tab</router-link>
        </div>

        <!-- Place Order Button -->
        <div class="action-row">
          <button
            class="btn-primary"
            :disabled="recommendations.length === 0 || submitting || submitted"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : submitted ? 'Order Placed' : 'Place Order' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(50000)
    const forecasts = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const submitted = ref(false)

    // Sort by forecasted_demand DESC, greedy fill up to budget
    const recommendations = computed(() => {
      const sorted = [...forecasts.value].sort(
        (a, b) => b.forecasted_demand - a.forecasted_demand
      )
      const result = []
      let spent = 0
      for (const item of sorted) {
        const cost = item.forecasted_demand * item.unit_cost
        if (spent + cost <= budget.value) {
          result.push({ ...item, total_cost: cost })
          spent += cost
        }
      }
      return result
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((s, i) => s + i.total_cost, 0)
    )

    // Reset submitted state when budget changes
    watch(budget, () => {
      submitted.value = false
    })

    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        forecasts.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendations.value.length === 0) return
      submitting.value = true
      try {
        const orderItems = recommendations.value.map(item => ({
          item_sku: item.item_sku,
          item_name: item.item_name,
          quantity: item.forecasted_demand,
          unit_cost: item.unit_cost,
          total_cost: item.total_cost
        }))
        await api.createRestockOrder({
          items: orderItems,
          total_cost: totalCost.value
        })
        submitted.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      budget,
      loading,
      error,
      recommendations,
      totalCost,
      submitting,
      submitted,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-control {
  padding: 1rem 0;
}

.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 0.5rem;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
}

.card-subtitle {
  color: #64748b;
  font-size: 0.875rem;
  margin-bottom: 1rem;
  margin-top: -0.5rem;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

tfoot .total-row td {
  border-top: 2px solid #e2e8f0;
  padding: 0.75rem;
  font-size: 0.875rem;
  background: #f8fafc;
}

.success-banner {
  margin-top: 1rem;
  padding: 0.875rem 1rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  color: #065f46;
  font-size: 0.938rem;
  display: flex;
  align-items: center;
  gap: 1rem;
}

.success-link {
  color: #065f46;
  font-weight: 600;
  text-decoration: underline;
}

.action-row {
  display: flex;
  justify-content: flex-end;
  margin-top: 1rem;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
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
</style>
