<template>
  <div class="home">
    <h1>都道府県別人口推移グラフ</h1>
    <PopulationChart />
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
