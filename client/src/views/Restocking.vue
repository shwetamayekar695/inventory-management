<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="budget-card-header">
          <span class="budget-label">{{ t('restocking.budget') }}</span>
          <span class="budget-value">{{ formatCurrency(budget) }}</span>
        </div>
        <input type="range" v-model.number="budget" min="0" max="500000" step="5000" class="budget-slider" />
        <div class="slider-labels">
          <span>$0</span>
          <span>$500,000</span>
        </div>
        <div class="budget-bar-track">
          <div class="budget-bar-fill" :style="{ width: budgetFillPercent + '%' }"></div>
        </div>
        <div class="budget-summary">
          <div class="budget-summary-item">
            <span class="budget-summary-label">{{ t('restocking.budgetUsed') }}</span>
            <span class="budget-summary-value used">{{ formatCurrency(totalSelectedCost) }}</span>
          </div>
          <div class="budget-summary-item">
            <span class="budget-summary-label">{{ t('restocking.budgetRemaining') }}</span>
            <span class="budget-summary-value remaining">{{ formatCurrency(budgetRemaining) }}</span>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedItems') }}</h3>
          <span class="item-count-badge">{{ recommendedItems.length }}</span>
        </div>
        <div v-if="recommendedItems.length === 0" class="empty-state">
          {{ t('restocking.noItems') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th class="col-right">{{ t('restocking.table.gapQuantity') }}</th>
                <th class="col-right">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-right">{{ t('restocking.table.itemTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                <td class="col-right">{{ item.gap_quantity }}</td>
                <td class="col-right">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-right"><strong>{{ formatCurrency(item.item_total) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="place-order-section">
        <button @click="placeOrder" :disabled="recommendedItems.length === 0 || submitting" class="place-order-btn">
          {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
        </button>
        <p v-if="successMessage" class="success-msg">{{ successMessage }}</p>
        <p v-if="errorMessage" class="error-msg">{{ errorMessage }}</p>
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
    const budget = ref(50000)
    const allForecasts = ref([])
    const inventoryItems = ref([])
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref('')
    const errorMessage = ref('')

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const formatCurrency = (value) => {
      return currencySymbol.value + value.toLocaleString()
    }

    const inventoryBySku = computed(() => {
      const map = new Map()
      for (const item of inventoryItems.value) map.set(item.sku, item)
      return map
    })

    const candidateItems = computed(() => {
      const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
      return allForecasts.value
        .filter(f => {
          const gap = f.forecasted_demand - f.current_demand
          return gap > 0 && inventoryBySku.value.has(f.item_sku)
        })
        .map(f => {
          const gap = f.forecasted_demand - f.current_demand
          const inv = inventoryBySku.value.get(f.item_sku)
          return {
            sku: f.item_sku,
            name: f.item_name,
            trend: f.trend,
            gap_quantity: gap,
            unit_cost: inv.unit_cost,
            item_total: Math.round(gap * inv.unit_cost * 100) / 100
          }
        })
        .sort((a, b) => {
          const td = trendOrder[a.trend] - trendOrder[b.trend]
          return td !== 0 ? td : b.gap_quantity - a.gap_quantity
        })
    })

    const recommendedItems = computed(() => {
      let remaining = budget.value
      return candidateItems.value.filter(item => {
        if (item.item_total <= remaining) {
          remaining -= item.item_total
          return true
        }
        return false
      })
    })

    const totalSelectedCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.item_total, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalSelectedCost.value)

    const budgetFillPercent = computed(() =>
      budget.value > 0 ? Math.min((totalSelectedCost.value / budget.value) * 100, 100) : 0
    )

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      successMessage.value = ''
      errorMessage.value = ''
      try {
        await api.createRestockingOrder({
          items: recommendedItems.value,
          budget_used: totalSelectedCost.value
        })
        successMessage.value = t('restocking.orderSuccess')
      } catch {
        errorMessage.value = t('restocking.orderError')
      } finally {
        submitting.value = false
      }
    }

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        allForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    onMounted(loadData)

    return {
      t, budget, loading, error, submitting,
      successMessage, errorMessage,
      currencySymbol, formatCurrency,
      recommendedItems, totalSelectedCost, budgetRemaining, budgetFillPercent,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {}

.budget-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
  padding-bottom: 0.875rem;
  border-bottom: 1px solid #e2e8f0;
}

.budget-label {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
  margin-bottom: 0.5rem;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-bottom: 1.25rem;
}

.budget-bar-track {
  width: 100%;
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.75rem;
}

.budget-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 4px;
  transition: width 0.3s ease;
}

.budget-summary {
  display: flex;
  justify-content: space-between;
  font-size: 0.875rem;
}

.budget-summary-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-summary-label {
  color: #64748b;
  font-weight: 500;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-summary-value {
  font-weight: 700;
  color: #0f172a;
  font-size: 1rem;
}

.budget-summary-value.used { color: #2563eb; }
.budget-summary-value.remaining { color: #059669; }

.item-count-badge {
  font-size: 0.75rem;
  background: #eff6ff;
  color: #2563eb;
  padding: 0.25rem 0.75rem;
  border-radius: 99px;
  font-weight: 600;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #94a3b8;
  font-size: 0.938rem;
}

.col-right {
  text-align: right;
}

.place-order-section {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 0.75rem;
  margin-top: 1rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 2rem;
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

.success-msg {
  color: #059669;
  font-weight: 500;
  font-size: 0.938rem;
}

.error-msg {
  color: #dc2626;
  font-weight: 500;
  font-size: 0.938rem;
}
</style>
