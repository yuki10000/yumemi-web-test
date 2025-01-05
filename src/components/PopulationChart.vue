<template>
  <div>
    <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>
    <div v-else>
      <h3>Select Prefectures</h3>
      <div v-for="pref in finalData" :key="pref.prefCode">
        <label>
          <input
            type="checkbox"
            :value="pref.prefCode"
            @change="togglePrefecture(pref)"
          />
          {{ pref.prefName }}
        </label>
      </div>

      <div class="buttons">
        <button
          v-for="type in populationTypes"
          :key="type.key"
          :class="{ active: selectedType === type.key }"
          @click="changePopulationType(type.key)"
        >
          {{ type.label }}
        </button>
      </div>

      <highcharts :options="chartOptions"></highcharts>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'
import axios from 'axios'
import Highcharts from 'highcharts'

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
  setup() {
    Highcharts.setOptions({
      lang: {
        decimalPoint: '.',
        thousandsSep: ',',
      },
    })

    const chartOptions = ref({
      chart: {
        type: 'line',
      },
      title: {
        text: 'Population Trends',
      },
      xAxis: {
        title: {
          text: 'Year',
        },
        type: 'linear',
      },
      yAxis: {
        title: {
          text: 'Population',
        },
      },
      series: [] as Highcharts.SeriesOptionsType[],
    })

    const prefectureList = ref<Prefecture[]>([])
    const finalData = ref<FinalData[]>([])
    const selectedPrefectures = ref<Set<number>>(new Set())
    const loading = ref(true)
    const error = ref<string | null>(null)
    const selectedType = ref<'total' | 'young' | 'working' | 'old'>('total')

    const populationTypes: {
      key: 'total' | 'young' | 'working' | 'old'
      label: string
    }[] = [
      { key: 'total', label: '総人口' },
      { key: 'young', label: '年少人口' },
      { key: 'working', label: '生産年齢人口' },
      { key: 'old', label: '老年人口' },
    ]

    const fetchPrefectureData = async (): Promise<void> => {
      try {
        const response = await axios.get<{ result: Prefecture[] }>(
          'https://yumemi-frontend-engineer-codecheck-api.vercel.app/api/v1/prefectures',
          {
            headers: {
              'X-API-KEY': '8FzX5qLmN3wRtKjH7vCyP9bGdEaU4sYpT6cMfZnJ',
            },
          },
        )
        prefectureList.value = response.data.result
      } catch {
        error.value = 'Failed to fetch prefecture data'
        throw new Error(error.value)
      }
    }

    const fetchPopulationCompositionData = async (
      prefCode: number,
    ): Promise<PopulationComposition> => {
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
      } catch {
        error.value = `Failed to fetch population composition data for prefCode ${prefCode}`
        throw new Error(error.value)
      }
    }

    const fetchAllPopulationCompositionData = async (): Promise<void> => {
      const promises = prefectureList.value.map(async (pref) => {
        const populationData = await fetchPopulationCompositionData(
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

      finalData.value = await Promise.all(promises)
    }

    const togglePrefecture = (pref: FinalData) => {
      if (selectedPrefectures.value.has(pref.prefCode)) {
        selectedPrefectures.value.delete(pref.prefCode)
      } else {
        selectedPrefectures.value.add(pref.prefCode)
      }
      updateChartSeries()
    }

    const updateChartSeries = () => {
      chartOptions.value.series = Array.from(selectedPrefectures.value)
        .map((prefCode) => {
          const pref = finalData.value.find((p) => p.prefCode === prefCode)
          if (!pref) return null
          const typeData = pref[selectedType.value]
          if (!typeData) return null
          return {
            type: 'line',
            name: pref.prefName,
            data: typeData.data.map((item) => [item.year, item.value]),
          }
        })
        .filter((series) => series !== null) as Highcharts.SeriesOptionsType[]
    }

    const changePopulationType = (
      type: 'total' | 'young' | 'working' | 'old',
    ) => {
      selectedType.value = type
      updateChartSeries() // チェックされている都道府県のデータを切り替え
    }

    const init = async () => {
      try {
        await fetchPrefectureData()
        await fetchAllPopulationCompositionData()
      } catch (e) {
        console.error(e)
      } finally {
        loading.value = false
      }
    }

    init()

    return {
      chartOptions,
      prefectureList,
      finalData,
      selectedPrefectures,
      loading,
      error,
      togglePrefecture,
      selectedType,
      changePopulationType,
      populationTypes,
    }
  },
})
</script>

<style scoped>
.buttons {
  margin-bottom: 20px;
}

button {
  margin-right: 10px;
  padding: 5px 10px;
  border: 1px solid #ccc;
  background-color: #f9f9f9;
  cursor: pointer;
}

button.active {
  background-color: #007bff;
  color: #fff;
  border-color: #007bff;
}
</style>
