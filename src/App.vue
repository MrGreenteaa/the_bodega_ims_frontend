<script setup>
import { ref, onMounted } from 'vue'
import { computed } from 'vue'

const API_URL = 'https://eddev18.pythonanywhere.com/products'

const products = ref([])
const error = ref('')
const loading = ref(true)
const submitting = ref(false)

const newItem = ref({
	name: '',
	sku: '',
	category: '',
	quantity: 0,
	price: 0
})

const search = ref('')
const selectedCategory = ref('')
const showAddForm = ref(false)
const currentView = ref('inventory')

// Modal states
const showDeleteModal = ref(false)
const productToDelete = ref(null)
const showEditModal = ref(false)
const productToEdit = ref(null)

// Security access gate
const isAuthenticated = ref(false)
const pinInput = ref('')
const pinError = ref('')

const checkPin = () => {
	if (pinInput.value === 'mark1') {
		isAuthenticated.value = true
		pinError.value = ''
		pinInput.value = ''
	} else {
		pinError.value = 'Invalid PIN. Please try again.'
		pinInput.value = ''
	}
}

const resetForm = () => {
	newItem.value = { name: '', sku: '', category: '', quantity: 0, price: 0 }
	showAddForm.value = false
}

const validateForm = () => {
	const { name, sku, category, quantity, price } = newItem.value
	return name.trim() !== '' && 
		   sku.trim() !== '' && 
		   category.trim() !== '' && 
		   quantity > 0 && 
		   price > 0
}

const openDeleteModal = (product) => {
	productToDelete.value = product
	showDeleteModal.value = true
}

const closeDeleteModal = () => {
	showDeleteModal.value = false
	productToDelete.value = null
}

const openEditModal = (product) => {
	productToEdit.value = { ...product }
	showEditModal.value = true
}

const closeEditModal = () => {
	showEditModal.value = false
	productToEdit.value = null
}

const updateProduct = async () => {
	if (!productToEdit.value) return
	error.value = ''
	try {
		const response = await fetch(`${API_URL}/${productToEdit.value.id}`, {
			method: 'PUT',
			headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify({
				name: productToEdit.value.name,
				sku: productToEdit.value.sku,
				category: productToEdit.value.category,
				quantity: Number(productToEdit.value.quantity),
				price: Number(productToEdit.value.price)
			})
		})
		if (!response.ok) throw new Error(`Failed to update product. status: ${response.status}`)
		
		// Update local list
		const index = products.value.findIndex(p => p.id === productToEdit.value.id)
		if (index !== -1) {
			products.value[index] = { ...productToEdit.value }
		}
		
		closeEditModal()
	} catch (err) {
		error.value = err.message
		console.error('Update product failed:', err)
	}
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
	if (!validateForm()) {
		error.value = 'Please fill in all required fields with valid values'
		return
	}
	error.value = ''
	submitting.value = true
	try {
		const response = await fetch(API_URL, {
			method: 'POST',
			headers: { 'Content-Type': 'application/json' },
			body: JSON.stringify({
				name: newItem.value.name,
				sku: newItem.value.sku,
				category: newItem.value.category,
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

const confirmDelete = async () => {
	if (!productToDelete.value) return
	const id = productToDelete.value.id
	error.value = ''
	try {
		const response = await fetch(`${API_URL}/${id}`, { method: 'DELETE' })
		if (!response.ok) throw new Error(`Failed to delete product. status: ${response.status}`)
		// Update local list
		products.value = products.value.filter(p => p.id !== id)
		closeDeleteModal()
	} catch (err) {
		error.value = err.message
		console.error('Delete product failed:', err)
	}
}

const deleteProduct = (product) => {
	openDeleteModal(product)
}

const uniqueCategories = computed(() => {
	const categories = [...new Set(products.value.map(p => p.category).filter(Boolean))]
	return categories.sort()
})

const lowStockProducts = computed(() => {
	return [...products.value].sort((a, b) => Number(a.quantity) - Number(b.quantity))
})

const filteredProducts = computed(() => {
	console.log("********")
	console.log(products)
	let filtered = products.value
	
	// Apply search filter
	const q = search.value.trim().toLowerCase()
	if (q) {
		filtered = filtered.filter(p => {
			return (
				(String(p.name || '').toLowerCase().includes(q)) ||
				(String(p.sku || '').toLowerCase().includes(q)) ||
				(String(p.category || '').toLowerCase().includes(q))
			)
		})
	}
	
	// Apply category filter
	if (selectedCategory.value && selectedCategory.value !== 'all') {
		filtered = filtered.filter(p => p.category === selectedCategory.value)
	}
	
	return filtered
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
	<div class="app-container">
		<!-- PIN Gate -->
		<div v-if="!isAuthenticated" class="pin-gate">
			<div class="pin-form">
				<h1 class="text-2xl font-bold mb-4">Mark I (The Bodega) Inventory</h1>
				<h2 class="text-lg font-semibold mb-6">Access Required</h2>
				<div class="pin-input-group">
					<input 
						v-model="pinInput" 
						type="password" 
						placeholder="Enter PIN" 
						class="pin-input"
						@keyup.enter="checkPin"
					/>
					<button @click="checkPin" class="btn btn-primary">Access</button>
				</div>
				<p v-if="pinError" class="pin-error">{{ pinError }}</p>
			</div>
		</div>

		<!-- Main Content -->
		<div v-else class="p-4">
			<h1 class="text-2xl font-bold mb-4">Mark I (The Bodega) Inventory</h1>

			<!-- Navigation -->
			<div class="navigation">
				<button 
					@click="currentView = 'inventory'" 
					:class="['nav-btn', { active: currentView === 'inventory' }]"
				>
					Inventory
				</button>
				<button 
					@click="currentView = 'low-stock'" 
					:class="['nav-btn', { active: currentView === 'low-stock' }]"
				>
					Low Stock
				</button>
			</div>

			<p v-if="loading">Loading…</p>
			<p v-else-if="error" class="text-red-600">An error occurred: {{ error }}</p>

			<div v-else class="space-y-4">
				<!-- Inventory View -->
				<div v-if="currentView === 'inventory'">
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

					<!-- Search and Filters -->
					<div class="search-filters">
						<input v-model="search" type="text" placeholder="Search by name, SKU, or category…" class="input search-input" />
						<select v-model="selectedCategory" class="input category-filter">
							<option value="all">All Categories</option>
							<option v-for="category in uniqueCategories" :key="category" :value="category">
								{{ category }}
							</option>
						</select>
					</div>

					<!-- Add New Product Button -->
					<div class="add-product-section">
						<button @click="showAddForm = !showAddForm" class="btn btn-primary add-product-btn">
							{{ showAddForm ? 'Cancel' : 'Add New Product' }}
						</button>
					</div>

					<!-- Add Item Form (Collapsible) -->
					<form v-if="showAddForm" @submit.prevent="addProduct" class="card form-card">
						<div class="form-grid">
							<input v-model="newItem.name" type="text" placeholder="Name" required class="input" />
							<input v-model="newItem.sku" type="text" placeholder="SKU" required class="input" />
							<input 
								v-model="newItem.category" 
								type="text" 
								placeholder="Category" 
								required 
								class="input"
								list="categories"
							/>
							<datalist id="categories">
								<option v-for="category in uniqueCategories" :key="category" :value="category" />
							</datalist>
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
										<th>Category</th>
										<th style="text-align:right;">Qty</th>
										<th style="text-align:right;">Price</th>
										<th style="text-align:right;">Value</th>
										<th>Actions</th>
									</tr>
								</thead>
								<tbody>
									<tr v-for="(item, index) in filteredProducts" :key="item.id" :class="{ 'table-row-even': index % 2 === 0 }">
										<td><strong>{{ item.name }}</strong></td>
										<td><code>{{ item.sku }}</code></td>
										<td><span class="category-badge">{{ item.category || 'Uncategorized' }}</span></td>
										<td style="text-align:right;">{{ item.quantity }}</td>
										<td style="text-align:right;">₱{{ Number(item.price).toFixed(2) }}</td>
										<td style="text-align:right;">₱{{ (Number(item.quantity) * Number(item.price)).toFixed(2) }}</td>
										<td>
											<button @click="openEditModal(item)" class="btn btn-secondary">Edit</button>
											<button @click="deleteProduct(item)" class="btn btn-danger">Delete</button>
										</td>
									</tr>
									<tr v-if="filteredProducts.length === 0">
										<td colspan="7" style="text-align:center; padding: 12px; color: #6b7280;">No products found.</td>
									</tr>
								</tbody>
								<tfoot v-if="filteredProducts.length">
									<tr>
										<td colspan="3" style="text-align:right; font-weight:600;">Totals:</td>
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

				<!-- Low Stock View -->
				<div v-else-if="currentView === 'low-stock'">
					<div class="card">
						<h2 class="view-title">Low Stock Products</h2>
						<p class="view-subtitle">Products sorted by quantity (lowest first)</p>
						
						<div class="table-wrap">
							<table class="table">
								<thead>
									<tr>
										<th>Product Name</th>
										<th style="text-align:right;">Quantity</th>
									</tr>
								</thead>
								<tbody>
									<tr v-for="(item, index) in lowStockProducts" :key="item.id" :class="{ 'table-row-even': index % 2 === 0 }">
										<td><strong>{{ item.name }}</strong></td>
										<td style="text-align:right;">{{ item.quantity }}</td>
									</tr>
									<tr v-if="lowStockProducts.length === 0">
										<td colspan="2" style="text-align:center; padding: 12px; color: #6b7280;">No products found.</td>
									</tr>
								</tbody>
							</table>
						</div>
					</div>
				</div>
			</div>
		</div>
	</div>

	<!-- Delete Confirmation Modal -->
	<div v-if="showDeleteModal" class="modal-overlay" @click="closeDeleteModal">
		<div class="modal-content" @click.stop>
			<h3 class="modal-title">Confirm Delete</h3>
			<p class="modal-message">
				Are you sure you want to delete <strong>{{ productToDelete?.name }}</strong>?
				This action cannot be undone.
			</p>
			<div class="modal-actions">
				<button @click="closeDeleteModal" class="btn btn-secondary">Cancel</button>
				<button @click="confirmDelete" class="btn btn-danger">
					Delete Product
				</button>
			</div>
		</div>
	</div>

	<!-- Edit Product Modal -->
	<div v-if="showEditModal" class="modal-overlay" @click="closeEditModal">
		<div class="modal-content edit-modal" @click.stop>
			<h3 class="modal-title">Edit Product</h3>
			<form @submit.prevent="updateProduct" class="edit-form">
				<div class="form-row">
					<label>Name:</label>
					<input v-model="productToEdit.name" type="text" required class="input" />
				</div>
				<div class="form-row">
					<label>SKU:</label>
					<input v-model="productToEdit.sku" type="text" required class="input" />
				</div>
				<div class="form-row">
					<label>Category:</label>
					<input 
						v-model="productToEdit.category" 
						type="text" 
						required 
						class="input"
						list="edit-categories"
					/>
					<datalist id="edit-categories">
						<option v-for="category in uniqueCategories" :key="category" :value="category" />
					</datalist>
				</div>
				<div class="form-row">
					<label>Quantity:</label>
					<input v-model.number="productToEdit.quantity" type="number" min="0" required class="input" />
				</div>
				<div class="form-row">
					<label>Price:</label>
					<input v-model.number="productToEdit.price" type="number" step="0.01" min="0" required class="input" />
				</div>
				<div class="modal-actions">
					<button type="button" @click="closeEditModal" class="btn btn-secondary">Cancel</button>
					<button type="submit" class="btn btn-primary">Update Product</button>
				</div>
			</form>
		</div>
	</div>
</template>

<style scoped>
/* App Container - Max Width & Center Layout */
.app-container {
	max-width: 1200px;
	margin: 0 auto;
	min-height: 100vh;
}

/* PIN Gate Styles */
.pin-gate {
	display: flex;
	align-items: center;
	justify-content: center;
	min-height: 100vh;
	background: #000000;
	background-image: linear-gradient(rgba(0, 0, 0, 0.4), rgba(0, 0, 0, 0.5)), 
	                  url('/iron-man-mk1-armor-wallpaper-4k-3840x2160.jpg');
	background-size: contain;
	background-position: center;
	background-repeat: no-repeat;
	padding: 16px;
}

.pin-form {
	background: rgba(255, 255, 255, 0.95);
	backdrop-filter: blur(10px);
	padding: 2rem;
	border-radius: 12px;
	box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3), 
	            0 4px 16px rgba(0, 0, 0, 0.2);
	border: 1px solid rgba(255, 255, 255, 0.2);
	max-width: 400px;
	width: 100%;
}

@media (max-width: 640px) {
	.pin-gate {
		padding: 8px;
		background-attachment: scroll;
	}
	
	.pin-form {
		padding: 1.5rem;
		border-radius: 8px;
	}
}

.text-lg {
	font-size: 1.125rem;
	line-height: 1.75rem;
}

.font-semibold {
	font-weight: 600;
}

.mb-6 {
	margin-bottom: 1.5rem;
}

.pin-input-group {
	display: flex;
	gap: 8px;
}

.pin-input {
	flex: 1;
	border: 1px solid #d1d5db;
	border-radius: 8px;
	padding: 8px 12px;
	font-size: 14px;
	box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.pin-input:focus {
	outline: none;
	border-color: #06B6D4;
	box-shadow: 0 0 0 3px rgba(6, 182, 212, 0.15);
}

.pin-error {
	margin-top: 12px;
	color: #EF4444;
	font-size: 14px;
}

/* PIN Gate specific title styling */
.pin-gate .text-2xl {
	background: linear-gradient(135deg, #06B6D4, #0891B2);
	-webkit-background-clip: text;
	-webkit-text-fill-color: transparent;
	background-clip: text;
	text-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
}

.pin-gate .text-lg {
	color: #374151;
	font-weight: 500;
}

/* Layout */
.space-y-4 > * + * { margin-top: 1rem; }
.text-2xl { font-size: 1.5rem; line-height: 2rem; }
.font-bold { font-weight: 700; }
.mb-4 { margin-bottom: 1rem; }
.p-4 { padding: 1rem; }
.text-red-600 { color: #dc2626; }

@media (max-width: 640px) {
	.p-4 { padding: 0.75rem; }
	.text-2xl { font-size: 1.25rem; line-height: 1.75rem; }
	.mb-4 { margin-bottom: 0.75rem; }
	.space-y-4 > * + * { margin-top: 0.75rem; }
}

/* Navigation */
.navigation {
	display: flex;
	gap: 8px;
	margin-bottom: 1.5rem;
	border-bottom: 1px solid #e5e7eb;
	overflow-x: auto;
}

.nav-btn {
	padding: 12px 24px;
	border: none;
	background: transparent;
	color: #6b7280;
	font-weight: 600;
	cursor: pointer;
	border-bottom: 2px solid transparent;
	transition: all 0.2s ease;
	white-space: nowrap;
	flex-shrink: 0;
}

.nav-btn:hover {
	color: #06B6D4;
}

.nav-btn.active {
	color: #06B6D4;
	border-bottom-color: #06B6D4;
}

@media (max-width: 640px) {
	.nav-btn {
		padding: 10px 16px;
		font-size: 0.875rem;
	}
}

/* View Titles */
.view-title {
	font-size: 1.25rem;
	font-weight: 700;
	margin-bottom: 8px;
	color: #111827;
}

.view-subtitle {
	color: #6b7280;
	font-size: 0.875rem;
	margin-bottom: 16px;
}

/* Cards */
.card { 
	border: 1px solid #e5e7eb;
	border-radius: 8px;
	padding: 12px;
	background: #fff;
}

.kpis { 
	display: grid;
	grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
	gap: 12px;
}

@media (max-width: 640px) {
	.kpis {
		grid-template-columns: 1fr;
		gap: 8px;
	}
	
	.card {
		padding: 8px;
	}
}

.kpis .card {
	border-left: 4px solid #06B6D4;
	box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.card-label { 
	font-size: 0.875rem;
	color: #6b7280;
	font-weight: 500;
}

.card-value { 
	font-size: 1.75rem;
	font-weight: 800;
	margin-top: 6px;
	color: #111827;
}

/* Search and Filters */
.search-filters {
	display: flex;
	gap: 12px;
	align-items: center;
}

.search-input {
	flex: 1;
}

.category-filter {
	min-width: 180px;
}

@media (max-width: 640px) {
	.search-filters {
		flex-direction: column;
		gap: 8px;
		align-items: stretch;
	}
	
	.category-filter {
		min-width: auto;
	}
}

/* Add Product Section */
.add-product-section {
	margin-bottom: 1rem;
}

.add-product-btn {
	font-size: 1rem;
	padding: 12px 24px;
}

@media (max-width: 640px) {
	.add-product-btn {
		width: 100%;
		padding: 14px 24px;
		font-size: 0.9rem;
	}
}

/* Form */
.form-grid { 
	display: grid;
	grid-template-columns: repeat(6, 1fr);
	gap: 8px;
}
@media (max-width: 1200px) { .form-grid { grid-template-columns: repeat(3, 1fr); } }
@media (max-width: 900px) { .form-grid { grid-template-columns: 1fr 1fr; } }
@media (max-width: 640px) { .form-grid { grid-template-columns: 1fr; gap: 12px; } }

.input { 
	width: 100%;
	border: 1px solid #d1d5db;
	border-radius: 8px;
	padding: 8px 12px;
	font-size: 14px;
	box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.input:focus { 
	outline: none;
	border-color: #06B6D4;
	box-shadow: 0 0 0 3px rgba(6, 182, 212, 0.15);
}

/* Buttons */
.btn { 
	display: inline-flex;
	align-items: center;
	justify-content: center;
	height: 36px;
	padding: 0 12px;
	border-radius: 8px;
	border: 1px solid transparent;
	font-weight: 600;
	cursor: pointer;
	transition: all 0.2s ease;
}

@media (max-width: 640px) {
	.btn {
		height: 40px;
		padding: 0 16px;
		font-size: 0.875rem;
	}
}

.btn:disabled { 
	opacity: 0.6;
	cursor: not-allowed;
}

.btn-primary { 
	background: #06B6D4;
	color: #fff;
}

.btn-primary:hover:not(:disabled) { 
	background: #0891B2;
}

.btn-danger { 
	background: #EF4444;
	color: #fff;
}

.btn-danger:hover:not(:disabled) { 
	background: #DC2626;
}

.btn-secondary {
	background: #6b7280;
	color: #fff;
}

.btn-secondary:hover:not(:disabled) {
	background: #4b5563;
}

/* Table */
.table-wrap { 
	width: 100%;
	overflow: auto;
	-webkit-overflow-scrolling: touch;
}

.table { 
	width: 100%;
	border-collapse: collapse;
	min-width: 600px;
}

.table th, .table td { 
	padding: 10px 12px;
	border-bottom: 1px solid #e5e7eb;
	text-align: left;
}

.table thead th { 
	background: #f9fafb;
	font-weight: 700;
	font-size: 0.9rem;
	color: #374151;
}

.table tbody tr:hover { 
	background: #f9fafb;
}

.table-row-even {
	background: #F8FAFC;
}

@media (max-width: 640px) {
	.table th, .table td {
		padding: 8px 6px;
		font-size: 0.8rem;
	}
	
	.table thead th {
		font-size: 0.8rem;
	}
	
	.table-wrap {
		border-radius: 8px;
		border: 1px solid #e5e7eb;
	}
	
	/* Stack action buttons vertically on mobile */
	.table td:last-child {
		display: flex;
		flex-direction: column;
		gap: 4px;
	}
	
	.table td:last-child .btn {
		width: 100%;
		height: 32px;
		font-size: 0.75rem;
		padding: 0 8px;
	}
}

.table code {
	background: #f3f4f6;
	padding: 2px 6px;
	border-radius: 4px;
	font-size: 0.875rem;
	color: #06B6D4;
	font-family: 'Courier New', monospace;
}

.category-badge {
	background: #e0f2fe;
	color: #0369a1;
	padding: 4px 8px;
	border-radius: 12px;
	font-size: 0.75rem;
	font-weight: 500;
}

@media (max-width: 640px) {
	.category-badge {
		font-size: 0.7rem;
		padding: 3px 6px;
	}
}

/* Modals */
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
	z-index: 1000;
	padding: 16px;
}

.modal-content {
	background: #fff;
	border-radius: 12px;
	padding: 24px;
	max-width: 500px;
	width: 100%;
	max-height: 90vh;
	overflow-y: auto;
	box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
}

.edit-modal {
	max-width: 600px;
}

@media (max-width: 640px) {
	.modal-overlay {
		padding: 8px;
		align-items: flex-start;
		padding-top: 20px;
	}
	
	.modal-content {
		padding: 16px;
		border-radius: 8px;
		max-height: calc(100vh - 40px);
	}
	
	.edit-modal {
		max-width: 100%;
	}
}

.modal-title {
	font-size: 1.25rem;
	font-weight: 700;
	margin-bottom: 16px;
	color: #111827;
}

.modal-message {
	margin-bottom: 24px;
	color: #374151;
	line-height: 1.5;
}

.modal-actions {
	display: flex;
	gap: 12px;
	justify-content: flex-end;
}

@media (max-width: 640px) {
	.modal-actions {
		flex-direction: column;
		gap: 8px;
	}
	
	.modal-actions .btn {
		width: 100%;
	}
}

.edit-form {
	display: flex;
	flex-direction: column;
	gap: 16px;
}

.form-row {
	display: flex;
	flex-direction: column;
	gap: 6px;
}

.form-row label {
	font-weight: 600;
	color: #374151;
	font-size: 0.875rem;
}
</style>
