<script setup>
import { ref, onMounted } from 'vue'
import { computed } from 'vue'

const API_URL = 'http://eddev18.pythonanywhere.com/products'

const products = ref([])
const error = ref('')
const loading = ref(true)
const submitting = ref(false)

const newItem = ref({
	name: '',
	sku: '',
	quantity: 0,
	price: 0
})

const search = ref('')

const resetForm = () => {
	newItem.value = { name: '', sku: '', quantity: 0, price: 0 }
}

const fetchProducts = async () => {
	try {
		const response = await fetch(API_URL)
		if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`)

		const result = await response.json()
		console.log(result)
		// result is an array, so assign directly
		products.value = result
	} catch (err) {
		error.value = err.message
		console.error('An error occurred:', err)
	} finally {
		loading.value = false
	}
}

const addProduct = async () => {
	if (submitting.value) return
	error.value = ''
	submitting.value = true
	try {
		const response = await fetch(API_URL, {
			method: 'POST',
			headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify({
				name: newItem.value.name,
				sku: newItem.value.sku,
				quantity: Number(newItem.value.quantity),
				price: Number(newItem.value.price)
			})
		})
		if (!response.ok) throw new Error(`Failed to add product. status: ${response.status}`)
		const created = await response.json()
		// Some APIs return created entity, some return message. Try to merge sensibly.
		if (created && (created.id || created.name)) {
			// Prepend newly created if full object returned; otherwise refetch
			if (created.id) {
				products.value = [created, ...products.value]
			} else {
				await fetchProducts()
			}
		} else {
			await fetchProducts()
		}
		resetForm()
	} catch (err) {
		error.value = err.message
		console.error('Add product failed:', err)
	} finally {
		submitting.value = false
	}
}

const deleteProduct = async (id) => {
	if (!id) return
	error.value = ''
	try {
		const response = await fetch(`${API_URL}/${id}`, { method: 'DELETE' })
		if (!response.ok) throw new Error(`Failed to delete product. status: ${response.status}`)
		// Update local list
		products.value = products.value.filter(p => p.id !== id)
	} catch (err) {
		error.value = err.message
		console.error('Delete product failed:', err)
	}
}

const filteredProducts = computed(() => {
	const q = search.value.trim().toLowerCase()
	if (!q) return products.value
	return products.value.filter(p => {
		return (
			(String(p.name || '').toLowerCase().includes(q)) ||
			(String(p.sku || '').toLowerCase().includes(q))
		)
	})
})

const totalUnits = computed(() => {
	return products.value.reduce((sum, p) => sum + Number(p.quantity || 0), 0)
})

const totalValue = computed(() => {
	return products.value.reduce((sum, p) => sum + Number(p.quantity || 0) * Number(p.price || 0), 0)
})

onMounted(fetchProducts)
</script>

<template>
	<div class="p-4">
		<h1 class="text-2xl font-bold mb-4">Sari-Sari Store Inventory</h1>

		<p v-if="loading">Loading…</p>
		<p v-else-if="error" class="text-red-600">An error occurred: {{ error }}</p>

		<div v-else class="space-y-4">
			<!-- Overview -->
			<div class="kpis">
				<div class="card">
					<div class="card-label">Products</div>
					<div class="card-value">{{ filteredProducts.length }}</div>
				</div>
				<div class="card">
					<div class="card-label">Total Units</div>
					<div class="card-value">{{ totalUnits }}</div>
				</div>
				<div class="card">
					<div class="card-label">Inventory Value</div>
					<div class="card-value">₱{{ totalValue.toFixed(2) }}</div>
				</div>
			</div>

			<!-- Search -->
			<div class="search-row">
				<input v-model="search" type="text" placeholder="Search by name or SKU…" class="input" />
			</div>

			<!-- Add Item Form -->
			<form @submit.prevent="addProduct" class="card form-card">
				<div class="form-grid">
					<input v-model="newItem.name" type="text" placeholder="Name" required class="input" />
					<input v-model="newItem.sku" type="text" placeholder="SKU" required class="input" />
					<input v-model.number="newItem.quantity" type="number" min="0" placeholder="Quantity" required class="input" />
					<input v-model.number="newItem.price" type="number" step="0.01" min="0" placeholder="Price" required class="input" />
					<button type="submit" :disabled="submitting" class="btn btn-primary">
						{{ submitting ? 'Adding…' : 'Add Item' }}
					</button>
				</div>
			</form>

			<!-- Products Table -->
			<div class="card">
				<div class="table-wrap">
					<table class="table">
						<thead>
							<tr>
								<th>Name</th>
								<th>SKU</th>
								<th style="text-align:right;">Qty</th>
								<th style="text-align:right;">Price</th>
								<th style="text-align:right;">Value</th>
								<th>Actions</th>
							</tr>
						</thead>
						<tbody>
							<tr v-for="item in filteredProducts" :key="item.id">
								<td><strong>{{ item.name }}</strong></td>
								<td><code>{{ item.sku }}</code></td>
								<td style="text-align:right;">{{ item.quantity }}</td>
								<td style="text-align:right;">₱{{ Number(item.price).toFixed(2) }}</td>
								<td style="text-align:right;">₱{{ (Number(item.quantity) * Number(item.price)).toFixed(2) }}</td>
								<td>
									<button @click="deleteProduct(item.id)" class="btn btn-danger">Delete</button>
								</td>
							</tr>
							<tr v-if="filteredProducts.length === 0">
								<td colspan="6" style="text-align:center; padding: 12px; color: #6b7280;">No products found.</td>
							</tr>
						</tbody>
						<tfoot v-if="filteredProducts.length">
							<tr>
								<td colspan="2" style="text-align:right; font-weight:600;">Totals:</td>
								<td style="text-align:right; font-weight:600;">{{ totalUnits }}</td>
								<td></td>
								<td style="text-align:right; font-weight:600;">₱{{ totalValue.toFixed(2) }}</td>
								<td></td>
							</tr>
						</tfoot>
					</table>
				</div>
			</div>
		</div>
	</div>
</template>

<style scoped>
/* Layout */
.space-y-4 > * + * { margin-top: 1rem; }
.text-2xl { font-size: 1.5rem; line-height: 2rem; }
.font-bold { font-weight: 700; }
.mb-4 { margin-bottom: 1rem; }
.p-4 { padding: 1rem; }

/* Cards */
.card { border: 1px solid #e5e7eb; border-radius: 8px; padding: 12px; background: #fff; }
.kpis { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 12px; }
.card-label { font-size: 0.8rem; color: #6b7280; }
.card-value { font-size: 1.25rem; font-weight: 700; margin-top: 4px; }

/* Form */
.form-card { }
.form-grid { display: grid; grid-template-columns: repeat(5, 1fr); gap: 8px; }
@media (max-width: 900px) { .form-grid { grid-template-columns: 1fr 1fr; } }
@media (max-width: 500px) { .form-grid { grid-template-columns: 1fr; } }
.input { width: 100%; border: 1px solid #d1d5db; border-radius: 6px; padding: 8px 10px; font-size: 14px; }
.input:focus { outline: none; border-color: #2563eb; box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.15); }

/* Buttons */
.btn { display: inline-flex; align-items: center; justify-content: center; height: 36px; padding: 0 12px; border-radius: 6px; border: 1px solid transparent; font-weight: 600; cursor: pointer; }
.btn:disabled { opacity: 0.6; cursor: not-allowed; }
.btn-primary { background: #2563eb; color: #fff; }
.btn-primary:hover { background: #1d4ed8; }
.btn-danger { background: #dc2626; color: #fff; }
.btn-danger:hover { background: #b91c1c; }

/* Table */
.table-wrap { width: 100%; overflow: auto; }
.table { width: 100%; border-collapse: collapse; }
.table th, .table td { padding: 10px 12px; border-bottom: 1px solid #e5e7eb; text-align: left; }
.table thead th { background: #f9fafb; font-weight: 700; font-size: 0.9rem; color: #374151; }
.table tbody tr:hover { background: #f9fafb; }
</style>
