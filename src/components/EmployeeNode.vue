<template>
  <div class="employee-node">
    <!-- Employee Card -->
    <div 
      class="employee-card"
      :class="{ 'has-children': employee.children?.length > 0 }"
      :data-employee-id="employee.id"
      @click="toggleExpand"
    >
      <!-- Avatar Circle -->
      <div 
        class="avatar-circle"
        :class="getDepartmentColorClass(employee.department)"
      >
        {{ getInitials(employee.name) }}
      </div>

      <div class="card-content">
        <!-- Basic Info -->
        <div class="info-section">
          <h3 class="name">{{ employee.name }}</h3>
          <p class="title">{{ employee.title }}</p>
          <p class="department-line">{{ employee.department }}</p>
          <p class="location-line">{{ employee.location }}</p>
        </div>

        <!-- Metrics Grid -->
        <div class="metrics-section">
          <!-- Layer Info -->
          <div class="metric-row">
            <span class="metric-badge">Layer: <span class="font-bold">{{ employee.layer }}</span></span>
            <span class="metric-badge">Reporting layers: <span class="font-bold">{{ employee.reportingLayers }}</span></span>
          </div>

          <!-- Manager & IC Info -->
          <div class="metric-row">
            <span class="metric-badge">Manager count: <span class="font-bold">{{ employee.managerCount }}</span></span>
            <span class="metric-badge">IC count: <span class="font-bold">{{ employee.icCount }}</span></span>
          </div>

          <!-- Cost Info -->
          <div class="metric-row">
            <span class="metric-badge">Manager cost: <span class="font-bold">{{ employee.managerCost }}</span></span>
            <span class="metric-badge">IC cost: <span class="font-bold">{{ employee.icCost }}</span></span>
          </div>

          <!-- Ratio -->
          <div class="metric-row">
            <span class="metric-badge">Manager cost ratio: <span class="font-bold">{{ employee.managerCostRatio.toFixed(2) }}</span></span>
          </div>

          <!-- Score -->
          <div class="score-badge">
            {{ employee.score }}
          </div>
        </div>
      </div>
    </div>

    <!-- Children Container -->
    <transition name="expand">
      <div 
        v-if="isNodeExpanded && employee.children?.length"
        class="children-container"
      >
        <div class="children-wrapper">
          <EmployeeNode
            v-for="(child, index) in employee.children"
            :key="child.id"
            :employee="child"
            class="child-node"
            :style="{ '--child-index': index }"
          />
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import { ref, inject, computed, watch, onMounted } from 'vue'

export default {
  name: 'EmployeeNode',
  props: {
    employee: {
      type: Object,
      required: true
    }
  },
  emits: ['tree-width-update'],
  setup(props, { emit }) {
    const expandedNodes = inject('expandedNodes')
    const toggleNodeExpansion = inject('toggleNodeExpansion')

    const isNodeExpanded = computed(() => {
      return expandedNodes.value.has(props.employee.id)
    })

    // Watch for changes in children visibility and emit width updates
    watch(() => isNodeExpanded.value, (newVal) => {
      if (newVal && props.employee.children) {
        emit('tree-width-update', props.employee.children.length)
      }
    }, { immediate: true })

    // Emit initial width for root level
    onMounted(() => {
      if (props.employee.children) {
        emit('tree-width-update', props.employee.children.length)
      }
    })

    const toggleExpand = () => {
      toggleNodeExpansion(props.employee.id, !isNodeExpanded.value)
    }

    const getInitials = (name) => {
      return name
        .split(' ')
        .map(word => word[0])
        .join('')
        .toUpperCase()
    }

    const getDepartmentColorClass = (department) => {
      const dept = department?.toLowerCase() || ''
      
      if (dept.includes('engineering')) return 'bg-blue-700'
      if (dept.includes('sales')) return 'bg-yellow-600'
      if (dept.includes('human')) return 'bg-purple-700'
      if (dept.includes('success')) return 'bg-orange-700'
      if (dept.includes('marketing')) return 'bg-pink-700'
      if (dept.includes('product')) return 'bg-indigo-700'
      if (dept.includes('operations')) return 'bg-gray-700'
      if (dept.includes('finance') || dept.includes('accounting')) return 'bg-green-700'
      if (dept.includes('intelligence') || dept.includes('business')) return 'bg-cyan-700'
      if (dept.includes('legal') || dept.includes('compliance')) return 'bg-violet-700'
      if (dept.includes('technology') || dept.includes('it')) return 'bg-blue-700'
      if (dept.includes('research') || dept.includes('development')) return 'bg-indigo-700'
      
      // Default color for any unmatched department
      return 'bg-slate-700'
    }

    return {
      isNodeExpanded,
      toggleExpand,
      getInitials,
      getDepartmentColorClass
    }
  }
}
</script>

<style scoped>
.employee-node {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
  width: 320px;
  min-width: 320px;
  box-sizing: border-box;
  margin: 0;
  z-index: 1;
}

.employee-card {
  position: relative;
  width: 320px;
  min-width: 320px;
  min-height: 400px;
  padding: 3rem 1.25rem 1rem;
  background-color: white;
  border-radius: 0.75rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border: 1px solid #e5e7eb;
  transition: all 0.3s ease-in-out;
  z-index: 1;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  box-sizing: border-box;
  margin: 0 auto;
}

.employee-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 16px -4px rgba(0, 0, 0, 0.1);
}

.avatar-circle {
  position: absolute;
  top: -1.5rem;
  left: 50%;
  transform: translateX(-50%);
  width: 3rem;
  height: 3rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 9999px;
  color: white;
  font-size: 1.125rem;
  font-weight: 700;
  letter-spacing: 0.05em;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  border: 3px solid white;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

/* Department-specific background colors */
.avatar-circle.bg-blue-700 { background-color: #2563eb; }
.avatar-circle.bg-yellow-600 { background-color: #ca8a04; }
.avatar-circle.bg-purple-700 { background-color: #7e22ce; }
.avatar-circle.bg-orange-700 { background-color: #c2410c; }
.avatar-circle.bg-pink-700 { background-color: #be185d; }
.avatar-circle.bg-indigo-700 { background-color: #4338ca; }
.avatar-circle.bg-gray-700 { background-color: #374151; }
.avatar-circle.bg-green-700 { background-color: #15803d; }
.avatar-circle.bg-cyan-700 { background-color: #0e7490; }
.avatar-circle.bg-violet-700 { background-color: #6d28d9; }
.avatar-circle.bg-slate-700 { background-color: #334155; }

.card-content {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  height: 100%;
  width: 100%;
  box-sizing: border-box;
}

.info-section {
  text-align: center;
  min-height: 6rem;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.name {
  @apply text-lg font-bold text-gray-900;
  line-height: 1.2;
}

.title {
  @apply text-sm font-semibold text-gray-700;
  line-height: 1.2;
  margin-bottom: 0.5rem;
}

.department-line {
  @apply text-sm text-gray-600;
  line-height: 1.2;
}

.location-line {
  @apply text-sm text-gray-600;
  line-height: 1.2;
}

.metrics-section {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-top: auto;
  padding-top: 1rem;
  width: 100%;
  box-sizing: border-box;
}

.metric-row {
  display: flex;
  justify-content: center;
  gap: 0.5rem;
  min-height: 1.75rem;
  width: 100%;
  flex-wrap: wrap;
}

.score-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  padding: 0.5rem;
  background-color: #111827;
  border-radius: 9999px;
  font-size: 0.875rem;
  font-weight: 600;
  color: white;
  margin-top: 0.5rem;
}

.metric-badge {
  display: inline-flex;
  align-items: center;
  padding: 0.375rem 0.75rem;
  background-color: #f3f4f6;
  border-radius: 9999px;
  font-size: 0.75rem;
  line-height: 1rem;
  color: #4b5563;
  white-space: nowrap;
}

.children-container {
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  width: max-content;
  padding-top: 2.5rem;
  min-width: max-content;
  z-index: 1;
}

.children-wrapper {
  display: flex;
  gap: 3rem;
  justify-content: center;
  width: max-content;
  position: relative;
  margin: 0;
  /* Prevent wrapping */
  white-space: nowrap;
  /* Prevent shrinking */
  flex-shrink: 0;
}

.child-node {
  position: relative;
  /* Ensure fixed width */
  flex: 0 0 320px;
  width: 320px;
  min-width: 320px;
  /* Add consistent spacing */
  margin: 0 1rem;
  z-index: 1;
  box-sizing: border-box;
  /* Smooth animation */
  animation: slideIn 0.3s ease-out;
  animation-fill-mode: both;
}

/* Expand/Collapse Animation */
.expand-enter-active,
.expand-leave-active {
  transition: all 0.5s ease-in-out;
  max-height: 1000px;
  opacity: 1;
}

.expand-enter-from,
.expand-leave-to {
  max-height: 0;
  opacity: 0;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Connection Lines */
.children-container::before {
  content: '';
  position: absolute;
  top: 0;
  left: 50%;
  width: 2px;
  height: 2.5rem;
  background-color: #e5e7eb;
  transform: translateX(-50%);
  z-index: 0;
}

.children-wrapper::before {
  content: '';
  position: absolute;
  top: 0;
  left: 1rem;
  right: 1rem;
  height: 2px;
  background-color: #e5e7eb;
  z-index: 0;
}
</style>