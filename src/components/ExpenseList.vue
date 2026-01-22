<template>
  <div>
    <div class="overflow-x-hidden md:overflow-x-auto">
      <div class="border rounded">
        <div class="px-3 py-2 bg-yellow-50 border-b flex justify-between items-center text-sm text-gray-700">
          <div>
            <div class="flex items-center gap-2">
              <div class="text-sm">Set Total budget:</div>
              <div class="flex items-center">
                <span class="mr-1">₹</span>
                <input v-model.number="totalBudget" type="number" min="0" step="0.01" class="w-32 border rounded px-2 py-1 text-sm" />
              </div>
            </div>
            <div class="text-xs font-bold text-gray-600">Remaining: <span :class="{ 'text-red-600 font-semibold': remainingBudget < 0 }">₹{{ remainingBudget.toFixed(2) }}</span></div>
          </div>
          <div class='font-bold'>Selected: {{ selectedCount }}</div>
          <div>Total: <span class="font-semibold">₹{{ selectedTotal.toFixed(2) }}</span></div>
        </div>
        <!-- wrapper with fixed height so table body scrolls; ~5 rows -> h-60 (~240px) -->
        <div class="max-h-60 overflow-y-auto">
          <table class="w-full divide-y divide-gray-200 table-auto md:table-fixed">
            <thead class="bg-gray-100">
              <tr class="text-left">
                <th class="px-3 py-2 text-sm font-medium text-gray-700 sticky top-0 bg-gray-100 z-10">
                  <input ref="headerCheckbox" type="checkbox" :checked="isAllSelected" @change="toggleAll($event)" class="h-4 w-4" aria-label="Select all rows" />
                </th>
                <th class="px-3 py-2 text-sm font-medium text-gray-700 sticky top-0 bg-gray-100 z-10">#</th>
                <th class="px-3 py-2 text-sm font-medium text-gray-700 sticky top-0 bg-gray-100 z-10">Expense Name</th>
                <th class="px-3 py-2 text-sm font-medium text-gray-700 sticky top-0 bg-gray-100 z-10">Amount</th>
                <th v-if="hasInvoiceColumn" class="px-3 py-2 text-sm font-medium text-gray-700 sticky top-0 bg-gray-100 z-10">Invoice</th>
                <th class="px-3 py-2 text-sm font-medium text-gray-700 sticky top-0 bg-gray-100 z-10">Method</th>
                <th class="px-3 py-2 text-sm font-medium text-gray-700 sticky top-0 bg-gray-100 z-10">Maker</th>
              </tr>
            </thead>
            <tbody class="bg-white divide-y divide-gray-200">
              <template v-for="(group) in groupedExpenses" :key="group.key">
                <tr class="bg-gray-100">
                  <td class="px-3 py-2 font-medium" :colspan="colCount">{{ group.label }} ({{ group.items.length }})</td>
                </tr>
                <tr v-for="(exp, idx) in group.items" :key="group.key + '-' + idx" :class="['h-12', isSelected(getGlobalIndex(exp)) ? 'bg-yellow-100' : (idx % 2 ? 'bg-gray-50' : 'bg-white') ]">
                  <td class="px-3 py-2">
                    <input type="checkbox" :checked="isSelected(getGlobalIndex(exp))" @change="toggle(getGlobalIndex(exp))" class="h-4 w-4" />
                  </td>
                  <td class="px-3 py-2">{{ idx + 1 }}</td>
                  <td class="px-3 py-2 first-letter:uppercase">{{ exp.name }}</td>
                  <td class="px-3 py-2">₹{{ exp.amount }}</td>
                  <td v-if="hasInvoiceColumn" class="px-3 py-2">
                    <template v-if="getInvoice(exp)">
                      <img v-if="isImage(exp) && invoiceUrlMap.has(getInvoice(exp))" :src="invoiceUrlMap.get(getInvoice(exp))" alt="invoice" class="inline-block h-8 w-8 object-cover rounded" />
                      <a href="#" @click.prevent="openInvoice(exp)" class="text-sm text-blue-600 ml-2">{{ exp.invoiceName || 'View' }}</a>
                    </template>
                    <template v-else>
                      <span class="text-sm text-gray-600">{{ exp.invoiceName || '-' }}</span>
                    </template>
                  </td>
                  <td class="px-3 py-2 first-letter:uppercase">{{ exp.methodLabel || exp.method }}</td>
                  <td class="px-3 py-2 first-letter:uppercase">{{ exp.maker }}</td>
                </tr>
              </template>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <div v-if="!expenses || expenses.length === 0" class="text-sm text-gray-500 mt-2">No expenses yet.</div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'

interface Expense {
  name: string
  maker: string
  amount: number
  method: string
  methodLabel?: string
  otherMethod?: string
  hasInvoice: boolean
  invoice: File | null
  invoiceName?: string
  date?: string | Date
}

const props = defineProps<{ expenses: Expense[] }>()
const emit = defineEmits<{ (e: 'remove', idx: number): void }>()

// track selected row indices; use a Set for efficient contains checks
const selectedIndices = ref<Set<number>>(new Set())
const totalBudget = ref<number>(20000)

// computed helpers
const selectedCount = computed(() => selectedIndices.value.size)
const selectedTotal = computed(() => {
  let total = 0
  for (const i of selectedIndices.value) {
    const e = props.expenses[i]
    if (!e) continue
    const amt = typeof e.amount === 'number' ? e.amount : Number(e.amount) || 0
    total += amt
  }
  return total
})

// remaining budget based on initial totalBudget and selected total
const remainingBudget = computed(() => {
  return totalBudget.value - selectedTotal.value
})

// persist totalBudget to localStorage so it's remembered across reloads
onMounted(() => {
  try {
    const v = localStorage.getItem('expense_total_budget')
    if (v !== null) totalBudget.value = Number(v) || totalBudget.value
  } catch (e) { /* ignore localStorage errors */ }
})

// watch totalBudget for persistence
watch(totalBudget, (v) => {
  try {
    localStorage.setItem('expense_total_budget', String(v))
  } catch (e) { /* ignore localStorage errors */ }
})

// header checkbox ref and helpers
const headerCheckbox = ref<HTMLInputElement | null>(null)
const isAllSelected = computed(() => {
  const len = props.expenses ? props.expenses.length : 0
  return len > 0 && selectedCount.value === len
})

// debug: log whenever expenses prop changes (helps detect why table stops rendering)
watch(() => props.expenses, (v) => {
  try {
    console.log('ExpenseList: expenses prop changed, len=', v ? v.length : 0)
  } catch (e) { console.log('ExpenseList: expenses prop changed') }
}, { deep: true, immediate: true })

// whether we should render the Invoice column (if any expense has invoice or invoiceName)
const hasInvoiceColumn = computed(() => {
  return (props.expenses || []).some((e) => !!(e as any).invoice || !!e.invoiceName)
})
const colCount = computed(() => hasInvoiceColumn.value ? 7 : 6)

// cache object URLs for invoice File objects to show thumbnails and open them
const invoiceUrlMap = ref<Map<any, string>>(new Map())

function getInvoice(exp: Expense) {
  return (exp as any).invoice
}

function isImage(exp: Expense) {
  const invoice = (exp as any).invoice
  return !!invoice && typeof invoice.type === 'string' && invoice.type.startsWith('image/')
}

function openInvoice(exp: Expense) {
  try {
    const f = (exp as any).invoice as any
    if (f && invoiceUrlMap.value.has(f)) {
      const url = invoiceUrlMap.value.get(f)!
      window.open(url)
      return
    }
    if (f && typeof URL !== 'undefined') {
      const u = URL.createObjectURL(f)
      invoiceUrlMap.value.set(f, u)
      window.open(u)
    }
  } catch (e) {
    console.error('openInvoice error', e)
  }
}

// keep invoiceUrlMap in sync with current expenses; create URLs for new Files and revoke those removed
watch(() => props.expenses, (arr) => {
  const files = new Set<any>()
  for (const e of (arr || [])) {
    if (e && (e as any).invoice) files.add((e as any).invoice)
  }
  // add new
  for (const f of files) {
    if (!invoiceUrlMap.value.has(f) && typeof URL !== 'undefined') {
      try { invoiceUrlMap.value.set(f, URL.createObjectURL(f)) } catch (e) { /* ignore */ }
    }
  }
  // remove old
  for (const [k, v] of Array.from(invoiceUrlMap.value.entries())) {
    if (!files.has(k)) {
      try { URL.revokeObjectURL(v) } catch (e) { /* ignore */ }
      invoiceUrlMap.value.delete(k)
    }
  }
}, { deep: true, immediate: true })

onUnmounted(() => {
  for (const [, v] of Array.from(invoiceUrlMap.value.entries())) {
    try { URL.revokeObjectURL(v) } catch (e) { /* ignore */ }
  }
  invoiceUrlMap.value.clear()
})

// group expenses by date (YYYY-MM-DD) and sort groups desc
const groupedExpenses = computed(() => {
  const map = new Map<string, Expense[]>()
  const arr = props.expenses || []
  for (const e of arr) {
    const eDate = (e as any).date
    const key = eDate ? (new Date(eDate)).toISOString().slice(0, 10) : (new Date()).toISOString().slice(0, 10)
    if (!map.has(key)) map.set(key, [])
    map.get(key)!.push(e)
  }
  const groups = Array.from(map.entries()).sort((a, b) => b[0].localeCompare(a[0])).map(([key, items]) => ({
    key,
    label: new Date(key).toLocaleDateString(),
    items
  }))
  return groups
})

function getGlobalIndex(exp: Expense) {
  return (props.expenses || []).findIndex((e) => e === exp)
}

function toggleAll(e: Event) {
  const checked = (e.target as HTMLInputElement).checked
  if (checked) {
    const s = new Set<number>()
    for (let i = 0; i < (props.expenses ? props.expenses.length : 0); i++) s.add(i)
    selectedIndices.value = s
  } else {
    selectedIndices.value = new Set()
  }
}

// update header checkbox indeterminate state when selection changes
watch([selectedCount, () => props.expenses.length], () => {
  if (!headerCheckbox.value) return
  headerCheckbox.value.indeterminate = (selectedCount.value > 0 && !isAllSelected.value)
})

// ensure selection indices are valid when list changes
watch(() => props.expenses.length, (len) => {
  const s = new Set<number>()
  for (const i of selectedIndices.value) {
    if (i < len) s.add(i)
  }
  selectedIndices.value = s
})

function isSelected(idx: number) {
  return selectedIndices.value.has(idx)
}

function toggle(idx: number) {
  if (idx < 0) return
  const s = new Set(selectedIndices.value)
  if (s.has(idx)) s.delete(idx)
  else s.add(idx)
  // reassign to trigger reactivity
  selectedIndices.value = s
}
</script>

<style scoped>
/* Simple table styles when Tailwind isn't available */
.table { width: 100%; }
</style>
