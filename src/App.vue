<template>
  <div class="min-h-screen bg-gray-100">
    <main class="container mx-auto">
      <OrgChart />
    </main>
  </div>
</template>

<script>
import OrgChart from './components/OrgChart.vue'

export default {
  name: 'App',
  components: {
    OrgChart
  }
}
</script>

<style>
@import './assets/base.css';
</style>
