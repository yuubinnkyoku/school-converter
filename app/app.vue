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
              label="略称+番号（例: TKB137など）"
              help="高校の略称・何期生かを表す数字を入力してください"
            >
              <div class="flex gap-2">
                <UInput
                  v-model="state.input"
                  placeholder="例: TKB137"
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
              description="略称は英字2〜4文字、続いて数字（例: TKB137）の形式で入力してください。"
              icon="i-heroicons-exclamation-triangle"
              color="warning"
              variant="soft"
            />
          </div>
        </div>
      </UCard>

      <div class="mt-6 text-xs text-gray-500">
        対応略称例:
        <UBadge color="neutral" variant="soft">TKB: 筑波大学附属中学校・高等学校</UBadge>
      </div>
    </UContainer>

    <NuxtPage />
  </UApp>
</template>

<script setup lang="ts">
// YAML データの読み込み
// Nuxt 4 では @rollup/plugin-yaml 経由でそのままオブジェクトとして import 可能
// 型はローカル定義で補完
// eslint/ts プラグインの型解決を助けるため拡張子まで含めて参照
const schoolsModule = {
  abbreviations: {
    // YAML直取り込みが型解決/ビルドに重くなる環境向けフォールバック:
    // 実体は runtime で fetch するのでダミー
  },
  baselines: {}
} as unknown

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

// YAML はビルド時 import を見送り、静的 JSON に変換したファイルを参照する設計へ変更。
// まずは既存の単一校(TKB)の要件：学校別 high1_cohort に対応するため、
// この場で学校個別の基準を直接定義し、将来JSON化で差し替え可能な形にする。
type SchoolCfg = { name: string; section: 'secondary'; high1_cohort: number; thresholds?: { before: number; after: number } }
const SCHOOLS: Record<string, SchoolCfg> = {
  TKB: { name: '筑波大学附属中学校・高等学校', section: 'secondary', high1_cohort: 136 }
}
const SECTION_DEFAULT = {
  secondary: { high1_cohort: 136, thresholds: { before: 3, after: -3 } }
} as const

function convert(inputRaw: string): ConvertResult | null {
  // 全角→半角へ正規化してから大文字化（例: ｔｋｂ１３７ → TKB137）
  const normalizeToAsciiUpper = (s: string) => (s ?? '').normalize('NFKC').trim().toUpperCase()
  const input = normalizeToAsciiUpper(inputRaw)
  // 数字桁数の上限を撤廃（1桁以上の任意長）
  const m = input.match(/^([A-Z]{2,4})(\d+)$/)
  if (!m) return null

  const abbr = m[1] as string
  const cohort = Number(m[2]) // 数字は「何期生」

  const abbrCfg = SCHOOLS[abbr]
  if (!abbrCfg) {
    return {
      schoolName: '未対応（大学は後日実装）',
      gradeOrDivision: '未対応'
    }
  }

  const schoolName = abbrCfg.name
  const section = abbrCfg.section
  const sectionBase = SECTION_DEFAULT[section]

  if (Number.isFinite(cohort) && cohort > 0) {
    // 学校固有を優先、なければセクション既定
    const BASELINE_HIGH1 = abbrCfg.high1_cohort ?? sectionBase.high1_cohort
    const before = abbrCfg.thresholds?.before ?? sectionBase.thresholds.before
    const after = abbrCfg.thresholds?.after ?? sectionBase.thresholds.after

    const d = cohort - BASELINE_HIGH1
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

  // 参照済みの定数を明示的に使用して未使用警告を抑止
  void SECTION_DEFAULT.secondary
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