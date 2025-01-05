<template>
  <div>
    <div>This is a population chart</div>
    <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>
    <div v-else>
      <!-- <pre>{{ JSON.stringify(finalData, null, 2) }}</pre> -->
      Successfully fetched data
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import axios from 'axios'

interface Prefecture {
  prefCode: number
  prefName: string
}

interface PopulationData {
  label: string
  data: Array<{ year: number; value: number }>
}

interface PopulationComposition {
  boundaryYear: number
  data: PopulationData[]
}

interface FinalData {
  prefCode: number
  prefName: string
  total: PopulationData | undefined
  young: PopulationData | undefined
  working: PopulationData | undefined
  old: PopulationData | undefined
}

export default defineComponent({
  name: 'PopulationChart',
  data() {
    return {
      prefectureList: [] as Prefecture[],
      finalData: [] as FinalData[],
      loading: true,
      error: null as string | null,
    }
  },
  async mounted() {
    try {
      await this.fetchPrefectureData()
      await this.fetchAllPopulationCompositionData()
    } catch (error) {
      this.error = 'Failed to fetch data'
    } finally {
      this.loading = false
      console.log(this.finalData)
    }
  },
  methods: {
    async fetchPrefectureData(): Promise<void> {
      try {
        const response = await axios.get<{ result: Prefecture[] }>(
          'https://yumemi-frontend-engineer-codecheck-api.vercel.app/api/v1/prefectures',
          {
            headers: {
              'X-API-KEY': '8FzX5qLmN3wRtKjH7vCyP9bGdEaU4sYpT6cMfZnJ',
            },
          },
        )
        this.prefectureList = response.data.result
      } catch (error) {
        this.error = 'Failed to fetch prefecture data'
        throw error
      }
    },
    async fetchPopulationCompositionData(
      prefCode: number,
    ): Promise<PopulationComposition> {
      try {
        const response = await axios.get<{ result: PopulationComposition }>(
          'https://yumemi-frontend-engineer-codecheck-api.vercel.app/api/v1/population/composition/perYear',
          {
            headers: {
              'X-API-KEY': '8FzX5qLmN3wRtKjH7vCyP9bGdEaU4sYpT6cMfZnJ',
            },
            params: { prefCode },
          },
        )
        return response.data.result
      } catch (error) {
        this.error = `Failed to fetch population composition data for prefCode ${prefCode}`
        throw error
      }
    },
    async fetchAllPopulationCompositionData(): Promise<void> {
      const promises = this.prefectureList.map(async (pref) => {
        const populationData = await this.fetchPopulationCompositionData(
          pref.prefCode,
        )
        return {
          prefCode: pref.prefCode,
          prefName: pref.prefName,
          total: populationData.data.find((item) => item.label === '総人口'),
          young: populationData.data.find((item) => item.label === '年少人口'),
          working: populationData.data.find(
            (item) => item.label === '生産年齢人口',
          ),
          old: populationData.data.find((item) => item.label === '老年人口'),
        }
      })

      this.finalData = await Promise.all(promises)
    },
  },
})
</script>

<style scoped></style>
