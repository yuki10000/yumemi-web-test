<template>
  <div class="home">
    <h1>Home</h1>
    <PopulationChart msg="Population data will be displayed here" />
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import PopulationChart from '@/components/PopulationChart.vue'

export default defineComponent({
  name: 'HomeView',
  components: {
    PopulationChart,
  },
})
</script>
