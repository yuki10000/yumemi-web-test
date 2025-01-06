<template>
  <div>
    <!-- データ読み込み中に「Loading...」を表示 -->
    <div v-if="loading">Loading...</div>
    <!-- エラーが発生した場合、エラーメッセージを表示 -->
    <div v-else-if="error">{{ error }}</div>
    <!-- データが正常に読み込まれた場合のコンテンツ -->
    <div v-else>
      <!-- 都道府県選択セクション -->
      <h3>都道府県を選択してください</h3>
      <div class="checkbox-container">
        <div v-for="pref in finalData" :key="pref.prefCode">
          <!-- 都道府県ごとのチェックボックス -->
          <label>
            <input
              type="checkbox"
              :value="pref.prefCode"
              @change="togglePrefecture(pref)"
            />
            {{ pref.prefName }}
            <!-- 都道府県名を表示 -->
          </label>
        </div>
      </div>

      <!-- 人口種類選択セクション -->
      <h3>人口の種類を選択してください</h3>
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

      <!-- チャート表示部分 -->
      <highcharts :options="chartOptions"></highcharts>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref } from 'vue'
import axios from 'axios'
import Highcharts from 'highcharts'

// 都道府県データのインターフェース
interface Prefecture {
  prefCode: number
  prefName: string
}

// 人口データのインターフェース
interface PopulationData {
  label: string
  data: Array<{ year: number; value: number }>
}

// 人口構成データのインターフェース
interface PopulationComposition {
  boundaryYear: number
  data: PopulationData[]
}

// 最終データのインターフェース
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
    // Highchartsの初期設定
    Highcharts.setOptions({
      lang: {
        decimalPoint: '.', // 小数点の形式
        thousandsSep: ',', // 3桁区切りのセパレーター
      },
    })

    // チャートのオプション
    const chartOptions = ref({
      chart: {
        type: 'line', // 折れ線線グラフの指定
      },
      title: {
        text: '人口推移', // チャートタイトル
      },
      xAxis: {
        title: {
          text: '年', // X軸のラベル
        },
        type: 'linear',
      },
      yAxis: {
        title: {
          text: '人口', // Y軸のラベル
        },
      },
      series: [] as Highcharts.SeriesOptionsType[], // データシリーズ
    })

    // 都道府県リスト
    const prefectureList = ref<Prefecture[]>([])
    // 最終データ
    const finalData = ref<FinalData[]>([])
    // 選択された都道府県のセット
    const selectedPrefectures = ref<Set<number>>(new Set())
    // 読み込み中フラグ
    const loading = ref(true)
    // エラーメッセージ
    const error = ref<string | null>(null)
    // 選択された人口種類
    const selectedType = ref<'total' | 'young' | 'working' | 'old'>('total')

    // 人口種類の定義
    const populationTypes: {
      key: 'total' | 'young' | 'working' | 'old'
      label: string
    }[] = [
      { key: 'total', label: '総人口' },
      { key: 'young', label: '年少人口' },
      { key: 'working', label: '生産年齢人口' },
      { key: 'old', label: '老年人口' },
    ]

    // 都道府県データを取得する非同期関数
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
        prefectureList.value = response.data.result // データをprefectureListに格納
      } catch {
        error.value = 'Failed to fetch prefecture data' // エラー時のメッセージ
        throw new Error(error.value)
      }
    }

    // 人口構成データを取得する非同期関数
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
            params: { prefCode }, // 都道府県コードを指定
          },
        )
        return response.data.result
      } catch {
        error.value = `Failed to fetch population composition data for prefCode ${prefCode}`
        throw new Error(error.value)
      }
    }

    // すべての人口構成データを取得する非同期関数
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

      finalData.value = await Promise.all(promises) // すべてのデータをまとめて格納
    }

    // 都道府県の選択状態を切り替える関数
    const togglePrefecture = (pref: FinalData) => {
      if (selectedPrefectures.value.has(pref.prefCode)) {
        selectedPrefectures.value.delete(pref.prefCode) // 選択解除
      } else {
        selectedPrefectures.value.add(pref.prefCode) // 選択追加
      }
      updateChartSeries() // チャートを更新
    }

    // チャートのシリーズデータを更新する関数
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
            data: typeData.data.map((item) => [item.year, item.value]), // 年と値をマッピング
          }
        })
        .filter((series) => series !== null) as Highcharts.SeriesOptionsType[]
    }

    // 人口種類を切り替える関数
    const changePopulationType = (
      type: 'total' | 'young' | 'working' | 'old',
    ) => {
      selectedType.value = type // 人口種類を変更
      updateChartSeries() // チャートを更新
    }

    // 初期化処理
    const init = async () => {
      try {
        await fetchPrefectureData()
        await fetchAllPopulationCompositionData()
      } catch (e) {
        console.error(e)
      } finally {
        loading.value = false // 読み込み終了
      }
    }

    init() // 初期化処理の実行

    // テンプレートで使用するデータと関数を返却
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
.checkbox-container {
  display: flex; /* チェックボックスを横並びに配置 */
  justify-content: center;
  flex-wrap: wrap;
  gap: 10px; /* 要素間の余白 */
  margin: 30px;
}
.buttons {
  margin-bottom: 20px;
}

button {
  margin-right: 10px;
  padding: 5px 10px;
  border: none;
  border-radius: 5px;
  background-color: #f9f9f9;
  cursor: pointer;
}

button.active {
  background-color: #007bff;
  color: #fff;
  border-color: #007bff;
}
</style>
