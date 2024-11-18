<template>
	<div class="container mx-auto px-6 py-8">
	  <h1 class="text-4xl font-bold text-center text-blue-600 mb-10">SpaceX Launches</h1>

		<div class="flex justify-between items-center mb-8 space-x-4">
			<!-- Year Filter -->
			<div class="flex-1">
				<label for="year-filter" class="block text-sm font-medium text-gray-700 mb-2 flex items-center">
					<CalendarIcon class="mr-2 text-blue-500" size="20" />
					Filter by Year
				</label>
				<div class="relative">
					<select
						id="year-filter"
						v-model="selectedYear"
						class="w-full px-4 py-2 border border-gray-300 rounded-lg shadow-sm focus:ring-2 focus:ring-blue-500 appearance-none pr-10"
					>
						<option value="">All Years</option>
						<option v-for="year in availableYears" :key="year" :value="year">{{ year }}</option>
					</select>
					<ChevronDownIcon class="absolute right-3 top-3 text-gray-400" size="20" />
				</div>
			</div>

			<!-- Search Input -->
			<div class="flex-1 text-right">
				<label for="search-input" class="block text-sm font-medium text-gray-700 mb-2 flex items-center justify-end">
					<SearchIcon class="mr-2 text-blue-500" size="20" />
					Search Launches
				</label>
				<div class="relative">
					<input
						id="search-input"
						v-model="searchQuery"
						placeholder="Search mission, rocket, site..."
						class="w-full px-4 py-2 border border-gray-300 rounded-lg shadow-sm focus:ring-2 focus:ring-blue-500 pr-10"
					/>
					<SearchIcon class="absolute right-3 top-3 text-gray-400" size="20" />
				</div>
			</div>
		</div>

	  <!-- Loading/Error State -->
	  <div v-if="loading" class="text-center py-8 text-xl text-gray-600">Loading launches...</div>
	  <div v-else-if="error" class="text-center py-8 text-red-600">
		Error loading launches: {{ error.message }}
	  </div>

	  <!-- Launches Table -->
	  <div v-else>
		<table class="min-w-full table-auto border-collapse mb-8">
		  <thead class="bg-gray-100">
			<tr>
			  <th
				class="border-b px-6 py-3 text-left cursor-pointer text-sm font-semibold text-gray-700 hover:bg-gray-200"
				@click="toggleSort('mission_name')"
			  >
				Mission Name
				<span class="ml-2">{{ getSortIcon('mission_name') }}</span>
			  </th>
			  <th
				class="border-b px-6 py-3 text-left cursor-pointer text-sm font-semibold text-gray-700 hover:bg-gray-200"
				@click="toggleSort('launch_date_utc')"
			  >
				Launch Date
				<span class="ml-2">{{ getSortIcon('launch_date_utc') }}</span>
			  </th>
			  <th
				class="border-b px-6 py-3 text-left cursor-pointer text-sm font-semibold text-gray-700 hover:bg-gray-200"
				@click="toggleSort('launch_site_name')"
			  >
				Launch Site Name
				<span class="ml-2">{{ getSortIcon('launch_site_name') }}</span>
			  </th>
			  <th
				class="border-b px-6 py-3 text-left cursor-pointer text-sm font-semibold text-gray-700 hover:bg-gray-200"
				@click="toggleSort('rocket_name')"
			  >
				Rocket
				<span class="ml-2">{{ getSortIcon('rocket_name') }}</span>
			  </th>
			  <th class="border-b px-6 py-3 text-left text-sm font-semibold text-gray-700">Details</th>
			</tr>
		  </thead>
		  <tbody>
			<tr
			  v-for="launch in paginatedLaunches"
			  :key="launch.id"
			  class="border-b hover:bg-gray-50"
			>
			  <td class="px-6 py-4">{{ launch.mission_name }}</td>
			  <td class="px-6 py-4">{{ formatDate(launch.launch_date_utc) }}</td>
			  <td class="px-6 py-4">{{ launch.launch_site_name?.site_name }}</td>
			  <td class="px-6 py-4">{{ launch.rocket?.rocket_name }}</td>
			  <td class="px-6 py-4" v-if="launch.details">{{ launch.details }}</td>
			  <td class="px-6 py-4" v-else>No details available</td>
			</tr>
		  </tbody>
		</table>

		<!-- Pagination Controls -->
		<div class="flex justify-between items-center text-sm text-gray-700">
		  <div>
			Showing {{ startIndex + 1 }} to {{ endIndex }} of {{ sortedLaunches.length }} launches
		  </div>
		  <div class="flex space-x-3">
			<button
			  @click="prevPage"
			  :disabled="currentPage === 1"
			  class="px-4 py-2 bg-gray-200 rounded-lg disabled:opacity-50"
			>
			  Previous
			</button>
			<div class="flex">
			  <button
				v-for="page in pageNumbers"
				:key="page"
				@click="currentPage = page"
				:class="[
				  'px-4 py-2 mx-1 rounded-lg',
				  currentPage === page ? 'bg-blue-600 text-white' : 'bg-gray-200 hover:bg-gray-300'
				]"
			  >
				{{ page }}
			  </button>
			</div>
			<button
			  @click="nextPage"
			  :disabled="currentPage === totalPages"
			  class="px-4 py-2 bg-gray-200 rounded-lg disabled:opacity-50"
			>
			  Next
			</button>
		  </div>
		</div>
	  </div>
	</div>
  </template>


<script setup>
	import { ref, computed } from 'vue'
	import { useQuery } from '@vue/apollo-composable'
	import gql from 'graphql-tag'

	// Constants
	const ITEMS_PER_PAGE = 25

	// GraphQL Query
	const LAUNCHES_QUERY = gql`
		query GetLaunches {
		launches {
			id
			mission_name
			launch_date_utc
			launch_site {
			site_name
			}
			rocket {
			rocket_name
			}
			details
		}
		}
	`

	// Query hook
	const { result, loading, error } = useQuery(LAUNCHES_QUERY)

	// Reactive state for filters and sorting
	const selectedYear = ref('')
	const searchQuery = ref('')
	const sortConfig = ref({
		key: 'launch_date_utc',
		direction: 'desc'
	})
	const currentPage = ref(1)

	// Computed property for launches
	const launches = computed(() => result.value?.launches ?? [])

	// Computed property for available years
	const availableYears = computed(() => {
		return [...new Set(
		launches.value.map(launch => new Date(launch.launch_date_utc).getFullYear())
		)]
	})

	// Filtered launches based on year and search query
	const filteredLaunches = computed(() => {
		let result = launches.value

		// Filter by year
		if (selectedYear.value) {
		result = result.filter(launch =>
			new Date(launch.launch_date_utc).getFullYear() === parseInt(selectedYear.value)
		)
		}

		// Filter by search query
		if (searchQuery.value) {
		const query = searchQuery.value.toLowerCase()
		result = result.filter(launch =>
			launch.mission_name.toLowerCase().includes(query) ||
			launch.rocket?.rocket_name.toLowerCase().includes(query) ||
			launch.launch_site_name?.site_name.toLowerCase().includes(query) ||
			(launch.details && launch.details.toLowerCase().includes(query))
		)
		}

		return result
	})

	// Sorted launches based on column and direction
	const sortedLaunches = computed(() => {
		return [...filteredLaunches.value].sort((a, b) => {
		let valueA, valueB;

		switch(sortConfig.value.key) {
			case 'mission_name':
			valueA = a.mission_name
			valueB = b.mission_name
			break
			case 'launch_site_name':
			valueA = a.launch_site_name?.site_name
			valueB = b.launch_site_name?.site_name
			break
			case 'rocket_name':
			valueA = a.rocket?.rocket_name
			valueB = b.rocket?.rocket_name
			break
			case 'launch_date_utc':
			default:
			valueA = new Date(a.launch_date_utc)
			valueB = new Date(b.launch_date_utc)
		}

		// Handle comparison for different types
		if (valueA instanceof Date && valueB instanceof Date) {
			return sortConfig.value.direction === 'desc'
			? valueB.getTime() - valueA.getTime()
			: valueA.getTime() - valueB.getTime()
		}

		// String comparison
		return sortConfig.value.direction === 'desc'
			? valueB.localeCompare(valueA)
			: valueA.localeCompare(valueB)
		})
	})

	// Pagination computed properties
	const totalPages = computed(() =>
		Math.ceil(sortedLaunches.value.length / ITEMS_PER_PAGE)
	)

	const pageNumbers = computed(() => {
		const pages = []
		for (let i = 1; i <= totalPages.value; i++) {
		pages.push(i)
		}
		return pages
	})

	const startIndex = computed(() =>
		(currentPage.value - 1) * ITEMS_PER_PAGE
	)

	const endIndex = computed(() =>
		Math.min(startIndex.value + ITEMS_PER_PAGE, sortedLaunches.value.length)
	)

	const paginatedLaunches = computed(() =>
		sortedLaunches.value.slice(startIndex.value, endIndex.value)
	)

	// Pagination methods
	function nextPage() {
		if (currentPage.value < totalPages.value) {
		currentPage.value++
		}
	}

	function prevPage() {
		if (currentPage.value > 1) {
		currentPage.value--
		}
	}

	// Toggle sort for a column
	function toggleSort(key) {
		if (sortConfig.value.key === key) {
		// If same column, toggle direction
		sortConfig.value.direction =
			sortConfig.value.direction === 'asc' ? 'desc' : 'asc'
		} else {
		// If new column, set to descending
		sortConfig.value = { key, direction: 'desc' }
		}
		// Reset to first page when sorting changes
		currentPage.value = 1
	}

	// Get sort icon based on current sort configuration
	function getSortIcon(key) {
		if (sortConfig.value.key !== key) return '↕'
		return sortConfig.value.direction === 'asc' ? '▲' : '▼'
	}

	// Date formatting function
	function formatDate(dateStr) {
		return new Date(dateStr).toLocaleDateString('en-US', {
		year: 'numeric',
		month: 'long',
		day: 'numeric',
		hour: '2-digit',
		minute: '2-digit'
		})
	}
</script>

<style scoped>
	table {
	width: 100%;
	border-collapse: collapse;
	}

	th, td {
	padding: 8px;
	text-align: left;
	border: 1px solid #ddd;
	}

	th {
	background-color: #f4f4f4;
	font-weight: bold;
	}

	/* New styles for details column to enable text wrapping */
	td:last-child {
	white-space: normal;
	word-wrap: break-word;
	max-width: 300px; /* Optional: limit width to prevent extremely wide columns */
	}
</style>
