<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Item header shown in both modes -->
            <div class="item-header">
              <div class="item-info">
                <h4 class="item-name">{{ backlogItem.item_name }}</h4>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
            </div>

            <!-- Create mode: form -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="submitForm">
              <div class="form-grid">
                <div class="form-group full-width">
                  <label class="form-label" for="supplier-name">Supplier Name</label>
                  <input
                    id="supplier-name"
                    v-model="form.supplier_name"
                    type="text"
                    class="form-input"
                    placeholder="Enter supplier name"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="quantity">Quantity</label>
                  <input
                    id="quantity"
                    v-model.number="form.quantity"
                    type="number"
                    class="form-input"
                    min="1"
                    required
                  />
                  <div class="form-hint">Shortage: {{ shortage }} units</div>
                </div>

                <div class="form-group">
                  <label class="form-label" for="unit-cost">Unit Cost</label>
                  <div class="input-prefix-wrapper">
                    <span class="input-prefix">$</span>
                    <input
                      id="unit-cost"
                      v-model.number="form.unit_cost"
                      type="number"
                      class="form-input with-prefix"
                      min="0"
                      step="0.01"
                      placeholder="0.00"
                      required
                    />
                  </div>
                </div>

                <div class="form-group">
                  <label class="form-label" for="delivery-date">Expected Delivery Date</label>
                  <input
                    id="delivery-date"
                    v-model="form.expected_delivery_date"
                    type="date"
                    class="form-input"
                    required
                  />
                </div>

                <div class="form-group total-cost-display" v-if="form.quantity && form.unit_cost">
                  <div class="total-label">Estimated Total</div>
                  <div class="total-value">${{ (form.quantity * form.unit_cost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
                </div>

                <div class="form-group full-width">
                  <label class="form-label" for="notes">Notes <span class="optional">(optional)</span></label>
                  <textarea
                    id="notes"
                    v-model="form.notes"
                    class="form-textarea"
                    rows="3"
                    placeholder="Any additional notes..."
                  ></textarea>
                </div>
              </div>

              <div v-if="submitError" class="submit-error">{{ submitError }}</div>
            </form>

            <!-- View mode: read-only display -->
            <div v-else-if="mode === 'view' && purchaseOrder" class="po-details">
              <div class="info-grid">
                <div class="info-item">
                  <div class="info-label">PO ID</div>
                  <div class="info-value mono">{{ purchaseOrder.id }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="status-badge" :class="purchaseOrder.status">{{ purchaseOrder.status }}</span>
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">{{ purchaseOrder.supplier_name }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ purchaseOrder.quantity }} units</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">${{ Number(purchaseOrder.unit_cost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Total Cost</div>
                  <div class="info-value total">
                    ${{ (purchaseOrder.quantity * purchaseOrder.unit_cost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(purchaseOrder.expected_delivery_date) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Created</div>
                  <div class="info-value">{{ formatDate(purchaseOrder.created_date) }}</div>
                </div>

                <div v-if="purchaseOrder.notes" class="info-item full-width">
                  <div class="info-label">Notes</div>
                  <div class="info-value notes">{{ purchaseOrder.notes }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close" :disabled="submitting">Close</button>
            <button
              v-if="mode === 'create'"
              type="submit"
              class="btn-primary"
              :disabled="submitting"
              @click="submitForm"
            >
              <span v-if="submitting">Creating...</span>
              <span v-else>Create Purchase Order</span>
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false
  },
  backlogItem: {
    type: Object,
    default: null
  },
  mode: {
    type: String,
    default: 'create', // 'create' | 'view'
    validator: (v) => ['create', 'view'].includes(v)
  }
})

const emit = defineEmits(['close', 'po-created'])

// Pre-computed shortage so the form hint is always accurate
const shortage = computed(() => {
  if (!props.backlogItem) return 0
  return props.backlogItem.quantity_needed - props.backlogItem.quantity_available
})

// purchase_order nested on the backlog item when mode === 'view'
const purchaseOrder = computed(() => props.backlogItem?.purchase_order ?? null)

const form = ref({
  supplier_name: '',
  quantity: 0,
  unit_cost: '',
  expected_delivery_date: '',
  notes: ''
})

const submitting = ref(false)
const submitError = ref(null)

// Reset form whenever the modal opens in create mode so stale values don't persist
watch(
  () => props.isOpen,
  (open) => {
    if (open && props.mode === 'create') {
      form.value = {
        supplier_name: '',
        // Pre-fill quantity with the shortage amount
        quantity: shortage.value > 0 ? shortage.value : 1,
        unit_cost: '',
        expected_delivery_date: '',
        notes: ''
      }
      submitError.value = null
    }
  }
)

const close = () => {
  if (!submitting.value) emit('close')
}

const submitForm = async () => {
  if (submitting.value) return

  submitError.value = null
  submitting.value = true

  try {
    const payload = {
      backlog_item_id: props.backlogItem.id,
      supplier_name: form.value.supplier_name,
      quantity: form.value.quantity,
      unit_cost: form.value.unit_cost,
      expected_delivery_date: form.value.expected_delivery_date
    }

    // Only include notes if the user actually typed something
    if (form.value.notes && form.value.notes.trim()) {
      payload.notes = form.value.notes.trim()
    }

    const poData = await api.createPurchaseOrder(payload)
    emit('po-created', poData)
  } catch (err) {
    // Surface the backend message when available, otherwise fall back to a generic one
    submitError.value =
      err?.response?.data?.detail ?? 'Failed to create purchase order. Please try again.'
    console.error('PO creation failed:', err)
  } finally {
    submitting.value = false
  }
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return dateString
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 600px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 2rem;
}

/* Item header */
.item-header {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding-bottom: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.item-name {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.25rem 0;
}

.item-sku {
  font-size: 0.813rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

/* Create mode: form */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 0;
}

.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.25rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-group.full-width {
  grid-column: 1 / -1;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.optional {
  font-weight: 400;
  text-transform: none;
  letter-spacing: 0;
  color: #94a3b8;
}

.form-input {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: #f8fafc;
  font-family: inherit;
  transition: border-color 0.15s ease, background 0.15s ease;
  width: 100%;
  box-sizing: border-box;
}

.form-input:focus {
  outline: none;
  border-color: #2563eb;
  background: white;
}

.form-input.with-prefix {
  padding-left: 2rem;
}

.input-prefix-wrapper {
  position: relative;
}

.input-prefix {
  position: absolute;
  left: 0.75rem;
  top: 50%;
  transform: translateY(-50%);
  font-size: 0.875rem;
  color: #64748b;
  pointer-events: none;
}

.form-hint {
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.125rem;
}

.form-textarea {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: #f8fafc;
  font-family: inherit;
  resize: vertical;
  transition: border-color 0.15s ease, background 0.15s ease;
  width: 100%;
  box-sizing: border-box;
}

.form-textarea:focus {
  outline: none;
  border-color: #2563eb;
  background: white;
}

.total-cost-display {
  justify-content: center;
  padding: 0.875rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 8px;
}

.total-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #16a34a;
}

.total-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #15803d;
}

.submit-error {
  margin-top: 1rem;
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #dc2626;
}

/* View mode: read-only details */
.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.5rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-item.full-width {
  grid-column: 1 / -1;
}

.info-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.mono {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

.info-value.total {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.info-value.notes {
  font-weight: 400;
  color: #475569;
  line-height: 1.5;
}

.status-badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: capitalize;
}

.status-badge.pending {
  background: #fef9c3;
  color: #854d0e;
}

.status-badge.confirmed {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.shipped {
  background: #ede9fe;
  color: #5b21b6;
}

.status-badge.delivered {
  background: #dcfce7;
  color: #166534;
}

.status-badge.cancelled {
  background: #f1f5f9;
  color: #475569;
}

/* Footer */
.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover:not(:disabled) {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  border: 1px solid #2563eb;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
  border-color: #1d4ed8;
}

.btn-primary:disabled,
.btn-secondary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Modal transition animations — mirrors BacklogDetailModal */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
