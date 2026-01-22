<template>
  <div class="flex flex-col md:flex-row md:gap-6 items-start w-full p-4">
      <div class="w-full md:w-1/3 p-4 bg-stone-200 rounded shadow mb-6 md:mb-0">
        <h2 class="text-xl font-semibold mb-4">Create Expense</h2>

        <form :key="formKey" @submit.prevent="submitPlain" class="space-y-3">
          <div>
            <label class="block text-sm font-medium mb-1" for="expenseName">Expense Name</label>
            <input id="expenseName" v-model="name.value" :ref="name.ref" type="text"  class="w-full border rounded px-2 py-1" />
            <p v-if="nameError && !hideErrors" class="text-sm text-red-600">{{ nameError }}</p>
          </div>

          <div>
            <label class="block text-sm font-medium mb-1" for="expenseMaker">Expense Maker</label>
            <input id="expenseMaker" v-model="maker.value" :ref="maker.ref" type="text"  class="w-full border rounded px-2 py-1" />
            <p v-if="makerError && !hideErrors" class="text-sm text-red-600">{{ makerError }}</p>
          </div>

          <div>
            <label class="block text-sm font-medium mb-1" for="expenseAmount">Amount</label>
            <input id="expenseAmount" v-model.number="amount.value" :ref="amount.ref" type="number" min="0" step="0.01" class="w-full border rounded px-2 py-1" />
            <p v-if="amountError && !hideErrors" class="text-sm text-red-600">{{ amountError }}</p>
          </div>

          <div>
            <label class="block text-sm font-medium mb-1" for="paymentMethod">Payment Method</label>
            <div class="flex items-center gap-4">
              <select id="paymentMethod" v-model="method.value" :ref="method.ref" class="w-full border rounded px-2 py-1">
                <option value="UPI">UPI</option>
                <option value="CASH">CASH</option>
                <option value="CARD">CARD</option>
                <option value="OTHERS">OTHERS</option>
              </select>

              <label class="flex items-center gap-2 text-sm">
                <input id="hasInvoice" type="checkbox" v-model="hasInvoice.value" :ref="hasInvoice.ref" class="h-4 w-4" />
                <span>Invoice</span>
              </label>
            </div>
          </div>

          <div v-if="method.value === 'OTHERS'">
            <label class="block text-sm font-medium mb-1" for="otherMethod">Specify Other Method</label>
            <input id="otherMethod" v-model="otherMethod.value" :ref="otherMethod.ref" type="text" class="w-full border rounded px-2 py-1" />
            <p v-if="otherMethod.error && !hideErrors" class="text-sm text-red-600">{{ otherMethod.error[0]?.message }}</p>
            <p v-if="otherMethodError" class="text-sm text-red-600">{{ otherMethodError }}</p>
          </div>

          <div v-if="hasInvoice.value">
            <label class="block text-sm font-medium mb-1">Invoice</label>
            <input ref="invoiceInput" type="file" @change="onInvoiceChange" accept="image/*,.pdf" />
            <div v-if="invoiceName.value" class="text-sm text-gray-600 mt-1">Selected: {{ invoiceName.value }}</div>
            <p v-if="invoiceError" class="text-sm text-red-600">{{ invoiceError }}</p>
          </div>

          <div class="pt-2">
            <button type="submit" class="bg-blue-600 text-white px-4 py-2 rounded">Add Expense</button>
            <button type="button" @click="resetForm" class="ml-2 bg-gray-200 px-3 py-2 rounded">Reset</button>
          </div>
        </form>
      </div>

      <div class="w-full md:w-2/3 p-4 bg-stone-200 rounded shadow">
        <h3 class="text-lg font-medium mb-2">Expense List</h3>
        <ExpenseList :expenses="expenses" @remove="removeExpense" />
      </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, nextTick } from 'vue'
import ExpenseList from './ExpenseList.vue'
import { useForm } from 'vue-hooks-form'

type PaymentMethod = 'UPI' | 'CASH' | 'CARD' | 'OTHERS'

interface FormValues {
  name: string
  maker: string
  amount: number
  method: PaymentMethod
  otherMethod: string
  hasInvoice: boolean
  invoice: File | null
  invoiceName: string
}

interface Expense extends FormValues { }

const expenses = ref<Expense[]>([])

const { useField, handleSubmit, set, errors, validateField, get } = useForm({ validateMode: 'submit',
  defaultValues: {
    name: '',
    maker: '',
    amount: 0,
    method: 'UPI',
    otherMethod: '',
    hasInvoice: false,
    invoice: null,
    invoiceName: ''
  }
})

const name = useField('name')
const maker = useField('maker')
const amount = useField('amount')
const method = useField('method')
const otherMethod = useField('otherMethod')
const hasInvoice = useField('hasInvoice')
const invoice = useField('invoice')
const invoiceName = useField('invoiceName')
const invoiceInput = ref<HTMLInputElement | null>(null)
const formKey = ref(0)  // used to force remount of the form to clear internal validation/UI state
const hideErrors = ref(false) // temporary flag to suppress validation messages immediately after submit/reset
const suppressWatch = ref(false) // when true, field change watchers won't re-enable errors (used during programmatic resets)

// local errors
const otherMethodError = ref<string | null>(null)
const invoiceError = ref<string | null>(null)
const nameError = ref<string | null>(null)
const makerError = ref<string | null>(null)
const amountError = ref<string | null>(null)

// clear otherMethod error when payment method changes away from OTHERS
watch(() => method.value, (val) => {
  if (val !== 'OTHERS') otherMethodError.value = null
})

// re-enable validation messages when the user edits any field after a submit/reset
watch(() => name.value, () => { if (!suppressWatch.value) hideErrors.value = false })
watch(() => maker.value, () => { if (!suppressWatch.value) hideErrors.value = false })
watch(() => amount.value, () => { if (!suppressWatch.value) hideErrors.value = false })
watch(() => otherMethod.value, () => { if (!suppressWatch.value) hideErrors.value = false })
function onInvoiceChange(e: Event) {
  const input = e.target as HTMLInputElement
  const f = input.files && input.files[0] ? input.files[0] : null
  invoice.value = f
  invoiceName.value = f ? f.name : ''
  invoiceError.value = null
}

// clear invoice when checkbox is unchecked
watch(() => hasInvoice.value, (val) => {
  if (!val) {
    invoice.value = null
    invoiceName.value = ''
    invoiceError.value = null
  }
})


// Plain submit handler that bypasses the library's validateFields (which may contain stale rules)
async function submitPlain(e: Event) {
  e.preventDefault()
  try {
    const data: FormValues = {
      name: name.value,
      maker: maker.value,
      amount: amount.value as any,
      method: method.value,
      otherMethod: otherMethod.value,
      hasInvoice: hasInvoice.value,
      invoice: invoice.value,
      invoiceName: invoiceName.value
    }
    console.log('submitPlain called with data:', JSON.parse(JSON.stringify(data)))

    // manual validation
    otherMethodError.value = null
    invoiceError.value = null
    nameError.value = null
    makerError.value = null
    amountError.value = null

    if (!data.name || String(data.name).trim() === '') nameError.value = 'Expense name is required.'
    if (!data.maker || String(data.maker).trim() === '') makerError.value = 'Expense maker is required.'
    const parsedAmt = typeof data.amount === 'string' ? parseFloat(String(data.amount)) : data.amount
    if (isNaN(Number(parsedAmt))) amountError.value = 'Amount must be a number.'
    if (nameError.value || makerError.value || amountError.value) {
      console.log('submitPlain: manual validation failed', { nameError: nameError.value, makerError: makerError.value, amountError: amountError.value })
      return
    }

    if (data.method === 'OTHERS' && (!data.otherMethod || data.otherMethod.trim() === '')) {
      otherMethodError.value = 'Other payment method is required.'
      return
    }
    if (data.hasInvoice && !invoice.value) {
      invoiceError.value = 'Invoice file is required when Invoice is checked.'
      return
    }

    const amt = typeof data.amount === 'string' ? parseFloat(String(data.amount)) : data.amount
    const methodLabel = data.method === 'OTHERS' && data.otherMethod ? data.otherMethod : data.method

    const newExpense: Expense & { methodLabel?: string } = {
      name: data.name,
      maker: data.maker,
      amount: amt,
      method: data.method,
      otherMethod: data.otherMethod,
      hasInvoice: hasInvoice.value,
      invoice: invoice.value,
      invoiceName: invoiceName.value
    }
    ;(newExpense as any).methodLabel = methodLabel

    expenses.value.push(newExpense)

    // reuse existing reset logic
    suppressWatch.value = true
    set('name', '')
    set('maker', '')
    set('amount', 0)
    set('method', 'UPI')
    set('otherMethod', '')
    set('hasInvoice', false)
    set('invoice', null)
    set('invoiceName', '')
    name.value = ''
    maker.value = ''
    amount.value = 0
    method.value = 'UPI'
    otherMethod.value = ''
    hasInvoice.value = false
    invoice.value = null
    invoiceName.value = ''
    if (invoiceInput.value) invoiceInput.value.value = ''
    otherMethodError.value = null
    invoiceError.value = null

    await nextTick()
    clearFormErrors()
    clearFieldErrors()

    suppressWatch.value = false
    hideErrors.value = true
    formKey.value += 1

    // delayed cleanup
    setTimeout(() => {
      try { console.log('form errors after submitPlain (delayed):', JSON.parse(JSON.stringify(errors))) } catch (e) { console.log('form errors after submitPlain (delayed):', errors) }
      ;['name', 'maker', 'amount', 'otherMethod'].forEach((k) => { delete (errors as any)[k] })
    }, 0)
  } catch (err) {
    console.error('submitPlain unexpected error', err)
  }
}

async function resetForm() {
  // clear errors
  invoiceError.value = null
  otherMethodError.value = null

  // prevent watchers from re-enabling errors while we programmatically clear fields
  suppressWatch.value = true

  // reset defaults
  set('name', '')
  set('maker', '')
  set('amount', 0)
  set('method', 'UPI')
  set('otherMethod', '')
  set('hasInvoice', false)
  set('invoice', null)
  set('invoiceName', '')

  // explicitly update bound fields so UI reflects the reset immediately
  name.value = ''
  maker.value = ''
  amount.value = 0
  method.value = 'UPI'
  otherMethod.value = ''
  hasInvoice.value = false
  invoice.value = null
  invoiceName.value = ''

  // clear local errors
  nameError.value = null
  makerError.value = null
  amountError.value = null

  if (invoiceInput.value) invoiceInput.value.value = ''

  // give watchers a tick to run then remove errors and remount
  await nextTick()
  clearFormErrors()
  clearFieldErrors()

  // re-enable watchers and suppress showing errors until user edits
  suppressWatch.value = false
  hideErrors.value = true

  // remount form to ensure full reset
  formKey.value += 1

  setTimeout(() => {
    try { console.log('form errors after reset (delayed):', JSON.parse(JSON.stringify(errors))) } catch (e) { console.log('form errors after reset (delayed):', errors) }
    ;['name', 'maker', 'amount', 'otherMethod'].forEach((k) => { delete (errors as any)[k] })
  }, 0)
}

function clearFormErrors() {
  // Delete all keys from the form's error object so the library's internal errors clear
  const fe = errors as any
  Object.keys(fe).forEach((k) => { delete fe[k] })
}

function clearFieldErrors() {
  // clear errors coming from the form library as the source of truth
  clearFormErrors()

  const fields = [name, maker, amount, otherMethod]
  fields.forEach((f: any) => {
    const err = f && f.error
    if (!err) return
    // if error is a ref (has a `.value`), clear it
    if (Object.prototype.hasOwnProperty.call(err, 'value')) {
      err.value = null
      return
    }
    // otherwise try clearing message or nulling the property
    if (err && typeof err === 'object') {
      if ('message' in err) err.message = ''
      try { f.error = null } catch (e) { /* ignore */ }
    } else {
      try { f.error = null } catch (e) { /* ignore */ }
    }
  })
}

function removeExpense(idx: number) {
  expenses.value.splice(idx, 1)
}
</script>

<style scoped>
/* Small, unobtrusive styles in case Tailwind isn't set up */
</style>
