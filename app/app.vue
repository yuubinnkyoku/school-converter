<template>
  <!-- 初期テーマ適用（FOUC回避） -->
  <Head>
    <Script tag="script" children="(function(){try{var k='theme';var s=localStorage.getItem(k);if(!s){s=window.matchMedia&&window.matchMedia('(prefers-color-scheme: dark)').matches?'dark':'light'}document.documentElement.dataset.theme=s;document.documentElement.classList.toggle('dark',s==='dark')}catch(e){}})()"/>
  </Head>

  <UApp>
    <UContainer class="py-10">
      <UCard>
        <template #header>
          <div class="flex items-center justify-between gap-2">
            <h1 class="text-xl font-semibold">学校アルファベット略称変換ツール</h1>

            <div class="flex items-center gap-2">
              <UBadge color="primary" variant="soft">Nuxt UI</UBadge>

              <!-- テーマトグル -->
              <UTooltip text="テーマ切替">
                <UButton
                  variant="soft"
                  :icon="isDark ? 'i-heroicons-moon-20-solid' : 'i-heroicons-sun-20-solid'"
                  @click="toggleTheme"
                  :aria-label="isDark ? 'ダークに設定済み' : 'ライトに設定済み'"
                />
              </UTooltip>
            </div>
          </div>
        </template>

        <div class="space-y-6">
          <UForm :state="state" @submit="onSubmit">
            <UFormGroup
              name="input"
              label="略称+番号（例: TKB137, GAKU2021）"
              help="学校略称と期や入学年度を入力してください"
            >
              <div class="flex gap-2">
                <UInput
                  v-model="state.input"
                  placeholder="例: TKB137, GAKU2021"
                  icon="i-heroicons-academic-cap"
                  autofocus
                  @keyup.enter="onSubmit()"
                />
                <UButton color="primary" icon="i-heroicons-arrow-right-circle" @click="onSubmit">
                  変換
                </UButton>
              </div>
            </UFormGroup>
          </UForm>

          <div class="space-y-3">
            <UDivider label="結果" />
            <div v-if="result" class="space-y-1">
              <div class="flex items-center gap-2">
                <UIcon name="i-heroicons-building-library" class="text-primary" />
                <span class="font-medium">学校名:</span>
                <span>{{ result?.schoolName }}</span>
              </div>
              <div class="flex items-center gap-2">
                <UIcon name="i-heroicons-user" class="text-primary" />
                <span class="font-medium">区分:</span>
                <span>{{ result?.gradeOrDivision }}</span>
              </div>
              <div class="text-xs text-gray-500">
                入力: <code>{{ lastInput }}</code>
              </div>
            </div>
            <UAlert
              v-else
              title="未入力または不正な形式です"
              description="略称は英字2〜4文字、続いて数字（例: TKB137, GAKU2021）の形式で入力してください。"
              icon="i-heroicons-exclamation-triangle"
              color="warning"
              variant="soft"
            />
          </div>
        </div>
      </UCard>

      <div class="mt-6 text-xs text-gray-500 space-x-1">
        <span>対応略称例:</span>
        <UBadge color="neutral" variant="soft">TKB: 筑波大学附属中学校・高等学校</UBadge>
        <UBadge color="neutral" variant="soft">GAKU: 学習院大学</UBadge>
      </div>
    </UContainer>

    <NuxtPage />
  </UApp>
</template>

<script setup lang="ts">
import schoolsData from '~/data/schools.yaml'

// @ts-ignore
const { school_types } = schoolsData

// YAML データの読み込み
// Nuxt 4 では @rollup/plugin-yaml 経由でそのままオブジェクトとして import 可能
// 型はローカル定義で補完
// eslint/ts プラグインの型解決を助けるため拡張子まで含めて参照
const THEME_KEY = 'theme'
const getSystemPrefersDark = () =>
  typeof window !== 'undefined' &&
  window.matchMedia &&
  window.matchMedia('(prefers-color-scheme: dark)').matches

const isDark = ref<boolean>(false)

onMounted(() => {
  // CSR 初期化（SSR では Head スクリプトで適用済み）
  const saved = localStorage.getItem(THEME_KEY)
  const current = saved ?? (getSystemPrefersDark() ? 'dark' : 'light')
  isDark.value = current === 'dark'
  document.documentElement.dataset.theme = current
  document.documentElement.classList.toggle('dark', isDark.value)
})

function applyTheme(mode: 'light' | 'dark') {
  document.documentElement.dataset.theme = mode
  document.documentElement.classList.toggle('dark', mode === 'dark')
  localStorage.setItem(THEME_KEY, mode)
  isDark.value = mode === 'dark'
}

function toggleTheme() {
  applyTheme(isDark.value ? 'light' : 'dark')
}

// 入力状態
const state = reactive({
  input: ''
})

// 結果データ型
type ConvertResult = {
  schoolName: string
  gradeOrDivision: string
}

// 最終入力と結果
const lastInput = ref<string>('')

const result = ref<ConvertResult | null>(null)

function findSchoolByAbbr(abbr: string) {
  for (const schoolTypeName in school_types) {
    const schoolType = school_types[schoolTypeName]
    const { schools } = schoolType
    for (const schoolName in schools) {
      if (schools[schoolName].abbr === abbr) {
        return {
          schoolName,
          ...schools[schoolName],
          ...schoolType,
          school_type: schoolTypeName
        }
      }
    }
  }
  return null
}

function convert(inputRaw: string): ConvertResult | null {
  // 全角→半角へ正規化してから大文字化（例: ｔｋｂ１３７ → TKB137）
  const normalizeToAsciiUpper = (s: string) => (s ?? '').normalize('NFKC').trim().toUpperCase()
  const input = normalizeToAsciiUpper(inputRaw)
  // 数字桁数の上限を撤廃（1桁以上の任意長）
  const m = input.match(/^([A-Z]{2,4})(\d+)$/)
  if (!m) return null

  const abbr = m[1] as string
  const num = Number(m[2])

  const schoolInfo = findSchoolByAbbr(abbr)

  if (!schoolInfo) {
    return {
      schoolName: '未対応の学校です',
      gradeOrDivision: '未対応'
    }
  }

  const { schoolName, high1_cohort, thresholds, school_type } = schoolInfo
  const { before, after } = thresholds

  if (school_type === '大学') {
    const admissionYear = num
    if (Number.isFinite(admissionYear) && admissionYear > 1900) {
      const currentYear = new Date().getFullYear()
      const d = currentYear - admissionYear
      if (d <= before) return { schoolName, gradeOrDivision: '入学前' }
      if (d >= after) return { schoolName, gradeOrDivision: '卒業' }
      return { schoolName, gradeOrDivision: `学部${d + 1}年` }
    }
  } else {
    const cohort = num
    if (Number.isFinite(cohort) && cohort > 0) {
      const d = cohort - high1_cohort
      if (d >= before) return { schoolName, gradeOrDivision: '入学前' }
      if (d <= after) return { schoolName, gradeOrDivision: '卒業後' }

      switch (d) {
        case 3: return { schoolName, gradeOrDivision: '中1' }
        case 2: return { schoolName, gradeOrDivision: '中2' }
        case 1: return { schoolName, gradeOrDivision: '中3' }
        case 0: return { schoolName, gradeOrDivision: '高1' }
        case -1: return { schoolName, gradeOrDivision: '高2' }
        case -2: return { schoolName, gradeOrDivision: '高3' }
        default: return { schoolName, gradeOrDivision: '学年不明' }
      }
    }
  }

  return { schoolName, gradeOrDivision: '学年不明' }
}

function onSubmit() {
  lastInput.value = state.input
  result.value = convert(state.input)
}
</script>

<style scoped>
/* 追加の軽微なスタイル（必要なら） */
</style>