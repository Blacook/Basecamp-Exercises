<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Success message (auto-hides after 3s) -->
      <div v-if="orderSuccess" class="success-message">
        {{ t('restocking.orderSuccess') }}
      </div>

      <!-- Budget Slider Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
          <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <div class="budget-body">
          <div class="budget-bar">
            <span class="budget-tick">{{ currencySymbol }}0</span>
            <input
              type="range"
              class="budget-slider"
              v-model.number="budget"
              :min="0"
              :max="maxBudget"
              :step="100"
            />
            <span class="budget-tick">{{ currencySymbol }}{{ maxBudget.toLocaleString() }}</span>
          </div>
        </div>
      </div>

      <!-- Recommended Items Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommendedItems') }}
            <span v-if="recommendedItems.length > 0" class="item-count">({{ recommendedItems.length }})</span>
          </h3>
          <div class="totals-summary">
            <span class="total-label">{{ t('restocking.totalSpend') }}:</span>
            <span class="total-value">{{ currencySymbol }}{{ totalSpend.toLocaleString() }}</span>
            <span class="remaining-label">{{ t('restocking.budgetRemaining') }}:</span>
            <span class="remaining-value">{{ currencySymbol }}{{ (budget - totalSpend).toLocaleString() }}</span>
          </div>
        </div>

        <div v-if="recommendedItems.length === 0" class="no-items">
          {{ t('restocking.noItems') }}
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th class="col-sku">{{ t('restocking.table.sku') }}</th>
                <th class="col-name">{{ t('restocking.table.name') }}</th>
                <th class="col-qty">{{ t('restocking.table.qty') }}</th>
                <th class="col-cost">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-total">{{ t('restocking.table.totalCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td class="col-sku"><span class="sku-tag">{{ item.sku }}</span></td>
                <td class="col-name">{{ item.name }}</td>
                <td class="col-qty">{{ item.quantity.toLocaleString() }}</td>
                <td class="col-cost">{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-total"><strong>{{ currencySymbol }}{{ item.item_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="action-row">
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('common.loading') : t('restocking.placeOrder') }}
          </button>
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
    const { t, currentCurrency } = useI18n()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const orderSuccess = ref(false)
    const budget = ref(0)
    const maxBudget = ref(0)
    const enrichedForecasts = ref([])

    // Greedy selection: include items with highest forecasted demand until budget runs out
    const recommendedItems = computed(() => {
      let running = 0
      return enrichedForecasts.value.filter(item => {
        if (running + item.item_cost <= budget.value) {
          running += item.item_cost
          return true
        }
        return false
      })
    })

    const totalSpend = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.item_cost, 0)
    })

    const loadData = async () => {
      try {
        loading.value = true
        // Fetch demand forecasts and inventory in parallel to join unit costs
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])

        // Build a SKU→unit_cost lookup from inventory
        const inventoryBySku = {}
        inventory.forEach(item => {
          inventoryBySku[item.sku] = item.unit_cost
        })

        // Enrich forecasts with unit cost and computed item cost
        const joined = forecasts.map(f => {
          const unit_cost = inventoryBySku[f.item_sku] || 0
          const quantity = f.forecasted_demand
          return {
            sku: f.item_sku,
            name: f.item_name,
            quantity,
            unit_cost,
            // Total cost to restock this item at forecasted demand
            item_cost: quantity * unit_cost
          }
        })

        // Sort by forecasted demand descending so highest-demand items fill budget first
        joined.sort((a, b) => b.quantity - a.quantity)
        enrichedForecasts.value = joined

        const total = joined.reduce((sum, item) => sum + item.item_cost, 0)
        // Round max up to nearest 100 for clean slider range
        maxBudget.value = Math.ceil(total / 100) * 100
        // Default budget to 25% of max, rounded to nearest 100
        budget.value = Math.round((maxBudget.value * 0.25) / 100) * 100
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return
      try {
        submitting.value = true
        const items = recommendedItems.value.map(({ sku, name, quantity, unit_cost }) => ({
          sku, name, quantity, unit_cost
        }))
        await api.submitRestockingOrder(items)
        orderSuccess.value = true
        // Auto-hide success message after 3 seconds
        setTimeout(() => { orderSuccess.value = false }, 3000)
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      currencySymbol,
      loading,
      error,
      submitting,
      orderSuccess,
      budget,
      maxBudget,
      recommendedItems,
      totalSpend,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.25rem;
}

.budget-body {
  padding: 0.5rem 0;
}

.budget-bar {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-tick {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
  white-space: nowrap;
  min-width: 60px;
}

.budget-tick:last-child {
  text-align: right;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
}

.item-count {
  font-size: 1rem;
  font-weight: 400;
  color: #64748b;
  margin-left: 0.25rem;
}

.totals-summary {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-size: 0.875rem;
}

.total-label,
.remaining-label {
  color: #64748b;
  font-weight: 500;
}

.total-value {
  font-weight: 700;
  color: #0f172a;
}

.remaining-value {
  font-weight: 700;
  color: #059669;
}

.no-items {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.col-sku { width: 140px; }
.col-name { width: auto; }
.col-qty { width: 140px; }
.col-cost { width: 120px; }
.col-total { width: 140px; }

.sku-tag {
  font-family: monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  color: #334155;
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
}

.action-row {
  display: flex;
  justify-content: flex-end;
  padding: 1rem 0.75rem 0.25rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 0.5rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-message {
  background: #d1fae5;
  border: 1px solid #10b981;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
