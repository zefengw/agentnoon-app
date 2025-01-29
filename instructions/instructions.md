## Project Overview

Before reading anything, I want you to read through the Rules section which you must follow at all times, the current File Structure, and the Documentation Code for Vue.js.

The goal of this project is to develop a responsive web application using Vue.js and Tailwind CSS that visualizes the hierarchical reporting structure of employees within an organization. The application should allow users to expand and collapse nodes to view the reporting lines, and display the number of descendants and non-leaf descendants for each node. Additionally, the application should calculate and display management cost, individual contributor (IC) cost, management cost ratio, and total cost based on employee salaries.

## Tech Stack

- **Frontend Framework**: Vue.js
- **CSS Framework**: Tailwind CSS
- **Data Visualization**: D3.js (optional for hierarchical visualization)
- **Build Tool**: Vite (recommended for Vue.js projects)

## Project Requirements

### Functional Requirements

1. **Hierarchical Visualization**:
   - Display the hierarchical structure of employees.
   - Allow users to expand and collapse nodes to view reporting lines.
   - Display the number of descendants and non-leaf descendants for each node.

2. **Cost Calculations**:
   - **Management Cost**: Recursive sum of the salaries of descendants who have subordinates.
   - **IC Cost**: Recursive sum of the salaries of descendants who do not have any subordinates.
   - **Total Cost**: Sum of the salaries of all descendants.
   - **Management Cost Ratio**: Ratio between individual contributors and managers.

3. **Data Handling**:
   - Load employee data from a CSV file.
   - Use memoization to store the count of each person's descendants to optimize performance.

4. **User Interface**:
   - Create a clean and responsive UI using Tailwind CSS.
   - Ensure the UI is intuitive and allows easy navigation of the hierarchical structure.

### Non-Functional Requirements

1. **Performance**:
   - Ensure efficient calculation of values using memoization.
   - Optimize the rendering of the hierarchical structure to handle large datasets.

2. **Usability**:
   - Provide a user-friendly interface with clear visual cues for expanding and collapsing nodes.
   - Ensure the application is responsive and works well on different screen sizes.

# Documentation
## Loading CSV Data using Vue.js
CODE EXAMPLE:
'''
import { ref } from 'vue';
import Papa from 'papaparse';

const employees = ref([]);

function loadCSVData() {
  Papa.parse('/path/to/employees.csv', {
    header: true,
    download: true,
    dynamicTyping: true,
    complete: (results) => {
      employees.value = results.data;
    }
  });
}
'''
## Memoization for Descendant Count
CODE EXAMPLE:
'''
const descendantCounts = new Map();

function getDescendantCount(employeeId) {
  if (descendantCounts.has(employeeId)) {
    return descendantCounts.get(employeeId);
  }

  const employee = employees.value.find(e => e.id === employeeId);
  let count = 0;

  if (employee.subordinates) {
    count += employee.subordinates.length;
    employee.subordinates.forEach(subId => {
      count += getDescendantCount(subId);
    });
  }

  descendantCounts.set(employeeId, count);
  return count;
}
'''
## Cost Calculations
CODE EXAMPLE:
'''
function calculateCosts(employeeId) {
  const employee = employees.value.find(e => e.id === employeeId);
  let managementCost = 0;
  let icCost = 0;
  let totalCost = employee.salary;

  if (employee.subordinates) {
    employee.subordinates.forEach(subId => {
      const subCosts = calculateCosts(subId);
      managementCost += subCosts.managementCost;
      icCost += subCosts.icCost;
      totalCost += subCosts.totalCost;
    });

    if (employee.subordinates.length > 0) {
      managementCost += employee.salary;
    } else {
      icCost += employee.salary;
    }
  } else {
    icCost += employee.salary;
  }

  return { managementCost, icCost, totalCost };
}
'''

## Vue Component for Hierarchical Display
CODE EXAMPLE: 
''' 
<template>
  <div>
    <div v-for="employee in employees" :key="employee.id">
      <div @click="toggleNode(employee.id)">
        {{ employee.name }} ({{ getDescendantCount(employee.id) }} descendants)
      </div>
      <div v-if="expandedNodes.includes(employee.id)">
        <EmployeeNode v-for="subId in employee.subordinates" :key="subId" :employeeId="subId" />
      </div>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue';

export default {
  props: ['employeeId'],
  setup(props) {
    const expandedNodes = ref([]);

    function toggleNode(id) {
      const index = expandedNodes.value.indexOf(id);
      if (index === -1) {
        expandedNodes.value.push(id);
      } else {
        expandedNodes.value.splice(index, 1);
      }
    }

    return { expandedNodes, toggleNode };
  }
};
</script>
'''

# File Structure:
agentnoon-app
├── README.md
├── eslint.config.js
├── index.html
├── instructions
│   ├── SCR-20241127-slfy-2.png
│   └── instructions.md
├── jsconfig.json
├── package-lock.json
├── package.json
├── public
│   ├── data
│   │   └── Giga_Corp.csv
│   └── favicon.ico
├── src
│   ├── App.vue
│   ├── assets
│   │   ├── base.css
│   │   ├── logo.svg
│   │   └── main.css
│   ├── components
│   │   ├── EmployeeNode.vue
│   │   ├── OrgChart.vue
│   │   └── icons
│   │       ├── IconCommunity.vue
│   │       ├── IconDocumentation.vue
│   │       ├── IconEcosystem.vue
│   │       ├── IconSupport.vue
│   │       └── IconTooling.vue
│   └── main.js
└── vite.config.js

# Rules
- The UI should resemble the hierarchical structure shown in the attached PNG image (at ./instructions/SCR-20241127-slfy-2.png). Use Tailwind CSS to create a clean and responsive layout. Ensure that nodes are clearly distinguishable and that the expand/collapse functionality is intuitive.
- All new components should be added to the components folder and named like example-component.vue unless otherwise specified
- All new pages should be added to src/app/folder-name and named like example-page.vue unless otherwise specified
- Reference the file structure, documentation, and rules throughout the process.
- No TypeScript allowed to be used
