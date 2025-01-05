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
      <highcharts :options="chartOptions"></highcharts>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'
import axios from 'axios'
import HighchartsVue from 'highcharts-vue'
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
  components: {
    highcharts: HighchartsVue.HighchartsVue, // HighchartsVue の登録
  },
  setup() {
    // データ
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
    const loading = ref(true)
    const error = ref<string | null>(null)

    // 都道府県データを取得
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

    // 人口構成データを取得
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

    // すべての人口構成データを取得
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

    // チェックボックスの切り替え処理
    const togglePrefecture = (pref: FinalData) => {
      if (!pref.total) return

      const seriesIndex = chartOptions.value.series.findIndex(
        (series) => series.name === pref.prefName,
      )

      if (seriesIndex !== -1) {
        // 該当のシリーズを削除
        chartOptions.value.series.splice(seriesIndex, 1)
      } else {
        // 新しいシリーズを追加
        chartOptions.value.series.push({
          type: 'line',
          name: pref.prefName,
          data: pref.total.data.map((item) => [item.year, item.value]),
        })
      }
    }

    // 初期化
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
      loading,
      error,
      togglePrefecture,
    }
  },
})
</script>

<style scoped>
/* 必要に応じてスタイルを追加 */
</style>
