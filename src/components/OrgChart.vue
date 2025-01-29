<template>
  <div class="org-chart-container">
    <div v-if="loading" class="flex justify-center items-center min-h-screen">
      <div class="animate-pulse flex space-x-4">
        <div class="text-xl text-gray-600">Loading organization data...</div>
        <svg class="animate-spin h-5 w-5 text-gray-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
          <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
          <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
        </svg>
      </div>
    </div>
    <div v-else-if="error" class="flex justify-center items-center min-h-screen">
      <div class="text-xl text-red-600 bg-red-50 p-4 rounded-lg shadow">{{ error }}</div>
    </div>
    <div v-else class="org-chart-wrapper">
      <div class="org-chart">
        <div class="chart-content">
          <EmployeeNode 
            v-if="rootEmployee"
            :employee="rootEmployee"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'
import Papa from 'papaparse'
import EmployeeNode from './EmployeeNode.vue'

export default {
  name: 'OrgChart',
  components: {
    EmployeeNode
  },
  setup() {
    const rootEmployee = ref(null)
    const loading = ref(true)
    const error = ref(null)
    const employeesMap = new Map()
    
    // Memoization caches
    const descendantCountCache = new Map()
    const metricsCache = new Map()

    const formatCost = (amount) => {
      if (amount >= 1000000000) {
        return `$${(amount / 1000000000).toFixed(1)}B / year`
      } else if (amount >= 1000000) {
        return `$${(amount / 1000000).toFixed(1)}M / year`
      } else if (amount > 0) {
        return `$${(amount / 1000).toFixed(1)}K / year`
      } else {
        return '$0'
      }
    }

    // Get memoized descendant count
    const getDescendantCount = (employeeId) => {
      if (descendantCountCache.has(employeeId)) {
        return descendantCountCache.get(employeeId)
      }

      const employee = employeesMap.get(employeeId)
      if (!employee) return 0

      let count = employee.children.length
      employee.children.forEach(child => {
        count += getDescendantCount(child.id)
      })

      descendantCountCache.set(employeeId, count)
      return count
    }

    // Get memoized metrics
    const getMetrics = (employeeId, layer) => {
      const cacheKey = `${employeeId}-${layer}`
      if (metricsCache.has(cacheKey)) {
        return metricsCache.get(cacheKey)
      }

      const employee = employeesMap.get(employeeId)
      if (!employee) return null

      // Initialize metrics
      let metrics = {
        totalManagerCost: 0,
        totalICCost: 0,
        totalDescendants: 0,
        maxLayer: layer,
        managerCount: 0,
        icCount: 0
      }

      // Calculate child metrics first
      employee.children.forEach(child => {
        const childMetrics = getMetrics(child.id, layer + 1)
        if (childMetrics) {
          metrics.totalManagerCost += childMetrics.totalManagerCost
          metrics.totalICCost += childMetrics.totalICCost
          metrics.totalDescendants += childMetrics.totalDescendants + 1
          metrics.maxLayer = Math.max(metrics.maxLayer, childMetrics.maxLayer)
          metrics.managerCount += childMetrics.managerCount
          metrics.icCount += childMetrics.icCount
        }
      })

      // Add current employee's contribution
      const employeeTotalComp = employee.salary + employee.bonus
      if (employee.children.length > 0) {
        metrics.totalManagerCost += employeeTotalComp
        metrics.managerCount += 1
      } else {
        metrics.totalICCost += employeeTotalComp
        metrics.icCount += 1
      }

      metricsCache.set(cacheKey, metrics)
      return metrics
    }

    const processEmployeeData = (data) => {
      try {
        // Clear caches
        descendantCountCache.clear()
        metricsCache.clear()
        
        // Skip header row and empty rows
        const rows = data.slice(1).filter(row => row.length > 1)
        
        // First pass: Create employee objects
        rows.forEach(row => {
          const id = row[0]?.trim()
          if (!id) return

          const employee = {
            id,
            name: row[1]?.trim(),
            title: row[2]?.trim(),
            email: row[3]?.trim(),
            managerId: row[4]?.trim(),
            status: row[5]?.trim(),
            hireDate: row[6]?.trim(),
            department: row[7]?.trim(),
            location: row[8]?.trim(),
            salary: parseFloat(row[9]) || 0,
            bonus: parseFloat(row[10]) || 0,
            children: []
          }

          if (employee.status === 'Active') {
            employeesMap.set(employee.id, employee)
          }
        })

        // Second pass: Build the tree structure
        employeesMap.forEach(employee => {
          if (employee.managerId && employeesMap.has(employee.managerId)) {
            const manager = employeesMap.get(employee.managerId)
            manager.children.push(employee)
          }
        })

        // Find the root employee
        rootEmployee.value = Array.from(employeesMap.values())
          .find(emp => !emp.managerId || !employeesMap.has(emp.managerId))

        if (!rootEmployee.value) {
          throw new Error('No root employee found in the data')
        }

        // Calculate and cache all metrics
        const calculateEmployeeMetrics = (employee, layer = 1) => {
          const metrics = getMetrics(employee.id, layer)
          
          // Update employee with calculated metrics
          employee.layer = layer
          employee.reportingLayers = metrics.maxLayer - layer
          employee.managerCount = metrics.managerCount
          employee.icCount = metrics.icCount
          employee.managerCost = formatCost(metrics.totalManagerCost)
          employee.icCost = formatCost(metrics.totalICCost)
          
          // Calculate manager cost ratio as management costs divided by total costs
          const totalCost = metrics.totalManagerCost + metrics.totalICCost
          employee.managerCostRatio = totalCost > 0 ? metrics.totalManagerCost / totalCost : 0

          employee.score = `${employee.children.length} / ${metrics.totalDescendants}`

          // Process children
          employee.children.forEach(child => calculateEmployeeMetrics(child, layer + 1))
        }

        calculateEmployeeMetrics(rootEmployee.value)

        // Sort children by department and title
        const sortChildren = (employee) => {
          employee.children.sort((a, b) => {
            if (a.department === b.department) {
              return a.title.localeCompare(b.title)
            }
            return a.department.localeCompare(b.department)
          })
          employee.children.forEach(sortChildren)
        }
        sortChildren(rootEmployee.value)

        loading.value = false
      } catch (err) {
        console.error('Data processing error:', err)
        error.value = `Error processing data: ${err.message}`
        loading.value = false
      }
    }

    onMounted(() => {
      // Load and parse the CSV file from the correct location
      fetch('/data/Giga_Corp.csv')
        .then(response => {
          if (!response.ok) {
            throw new Error('Failed to load CSV file')
          }
          return response.text()
        })
        .then(csvText => {
          Papa.parse(csvText, {
            skipEmptyLines: true,
            complete: (results) => {
              processEmployeeData(results.data)
            },
            error: (error) => {
              console.error('CSV parsing error:', error)
              error.value = `Error parsing CSV: ${error.message}`
              loading.value = false
            }
          })
        })
        .catch(error => {
          console.error('Data loading error:', error)
          error.value = `Error loading CSV: ${error.message}`
          loading.value = false
        })
    })

    return {
      rootEmployee,
      loading,
      error
    }
  }
}
</script>

<style>
.org-chart-container {
  @apply bg-gray-50;
  min-height: 100vh;
  width: 100vw;
  display: flex;
  flex-direction: column;
  align-items: center;
  overflow: hidden;
}

.org-chart-wrapper {
  width: 100%;
  height: calc(100vh - 4rem);
  overflow: auto;
  padding: 2rem;
}

.org-chart {
  min-width: min-content;
  min-height: min-content;
  display: flex;
  justify-content: center;
  padding: 2rem;
  margin: 0 auto;
  width: max-content;
}

.chart-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 0 100vw;
  margin: 0 auto;
  width: max-content;
  position: relative;
}

/* Update the children container styles */
.children-container {
  position: relative;
  width: max-content;
  padding-top: 2.5rem;
  margin: 0 auto;
}

.children-wrapper {
  display: flex;
  gap: 2rem;
  justify-content: center;
  width: max-content;
  margin: 0 auto;
}

/* Scrollbar styling */
.org-chart-wrapper {
  scrollbar-width: thin;
  scrollbar-color: #CBD5E0 #EDF2F7;
}

.org-chart-wrapper::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

.org-chart-wrapper::-webkit-scrollbar-track {
  background: #EDF2F7;
  border-radius: 4px;
}

.org-chart-wrapper::-webkit-scrollbar-thumb {
  background-color: #CBD5E0;
  border-radius: 4px;
  border: 2px solid #EDF2F7;
}

.org-chart-wrapper::-webkit-scrollbar-thumb:hover {
  background-color: #A0AEC0;
}

/* Department color classes */
.department-engineering { @apply bg-blue-100 border-blue-200; }
.department-operations { @apply bg-gray-100 border-gray-200; }
.department-finance { @apply bg-green-100 border-green-200; }
.department-marketing { @apply bg-pink-100 border-pink-200; }
.department-sales { @apply bg-yellow-100 border-yellow-200; }
.department-hr { @apply bg-purple-100 border-purple-200; }
.department-product { @apply bg-orange-100 border-orange-200; }
.department-success { @apply bg-indigo-100 border-indigo-200; }

/* Avatar styles */
.avatar {
  @apply rounded-full flex items-center justify-center text-sm font-semibold;
  width: 2.5rem;
  height: 2.5rem;
}

.avatar-engineering { @apply bg-blue-500 text-white; }
.avatar-operations { @apply bg-gray-500 text-white; }
.avatar-finance { @apply bg-green-500 text-white; }
.avatar-marketing { @apply bg-pink-500 text-white; }
.avatar-sales { @apply bg-yellow-500 text-white; }
.avatar-hr { @apply bg-purple-500 text-white; }
.avatar-product { @apply bg-orange-500 text-white; }
.avatar-success { @apply bg-indigo-500 text-white; }

/* Connection lines */
.org-chart-line {
  @apply border-gray-300;
  position: absolute;
  border-style: solid;
  pointer-events: none;
}

.org-chart-line-vertical {
  @apply border-l;
  width: 2px;
}

.org-chart-line-horizontal {
  @apply border-t;
  height: 2px;
}

/* Metrics badges */
.metric-badge {
  @apply px-2 py-1 rounded-full text-xs font-medium;
  background-color: rgba(0, 0, 0, 0.05);
}

/* Card animations */
.employee-card {
  @apply transition-all duration-200 ease-in-out;
  transform-origin: center top;
}

.employee-card:hover {
  @apply shadow-lg;
  transform: translateY(-2px);
}

/* Loading animation */
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: .5; }
}

.animate-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}
</style>