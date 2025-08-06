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
            <h1 class="text-xl font-semibold">学校コード変換ツール</h1>

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
              label="略称+番号（例: TKB137, WSD205, KGU101 など）"
              help="高校・大学の略称＋学年/学部を表す数字を入力してください"
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
              description="略称は英字2〜4文字、続いて数字1〜3桁（例: TKB137）の形式で入力してください。"
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
        <UBadge color="neutral" variant="soft">WSD: 早稲田大学</UBadge>
        <UBadge color="neutral" variant="soft">KGU: 関西学院大学</UBadge>
      </div>
    </UContainer>

    <NuxtPage />
  </UApp>
</template>

<script setup lang="ts">
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

/**
 * 変換ロジック
 * 仕様（初期実装の想定）:
 * - 入力は「略称(英字2-4)+数字(1-3桁)」例: TKB137
 * - 略称 -> 正式名称のマッピングを用意
 * - 数字 -> 学年・学部などを推定
 *
 * 暫定ルール:
 * - TKB: 筑波大学附属中学校・高等学校
 *    - 1xx: 中学 年=下2桁の十の位? 簡易に「3桁目で中/高を判定」
 *    - 100-199: 中学(1-9年は存在しないため実運用で見直し). ここでは末尾の1桁を学年とみなす
 *    - 200-299: 高校(末尾の1桁を学年とみなす)
 *    - 例: 137 -> 中学3年, 245 -> 高校5年(実在しない学年の可能性は残るがデモとして)
 * - WSD: 早稲田大学
 *    - 100-199: 学部 学年=末尾1桁（例: 105 -> 学部1年）
 * - KGU: 関西学院大学
 *    - 100-199: 学部 学年=末尾1桁
 *
 * 実運用では学校ごとに正確なコード体系に合わせてロジックを拡張してください。
 */
function convert(inputRaw: string): ConvertResult | null {
  // 全角→半角へ正規化してから大文字化（例: ｔｋｂ１３７ → TKB137）
  const normalizeToAsciiUpper = (s: string) => (s ?? '').normalize('NFKC').trim().toUpperCase()
  const input = normalizeToAsciiUpper(inputRaw)
  const m = input.match(/^([A-Z]{2,4})(\d{1,3})$/)
  if (!m) return null

  const abbr = m[1] as string
  const cohort = Number(m[2]) // 数字は「何期生」

  // 今は中高のみ対応（大学は未対応）
  const schoolMap = {
    TKB: '筑波大学附属中学校・高等学校'
  } as const

  type Abbr = keyof typeof schoolMap
  const isKnownAbbr = (a: string): a is Abbr => (a as string) in schoolMap

  if (!isKnownAbbr(abbr)) {
    return {
      schoolName: '未対応（大学は後日実装）',
      gradeOrDivision: '未対応'
    }
  }

  const schoolName: string = String(schoolMap[abbr])

  // 学年/入学前/卒業後判定（中高のみ）
  // 基準: 137期 → 中3
  // d = cohort - 137
  // d ≥ +3 → 入学前
  // d = +2 → 中1, +1 → 中2, 0 → 中3
  // d = -1 → 高1, -2 → 高2, -3 → 高3
  // d ≤ -4 → 卒業後
  if (Number.isFinite(cohort) && cohort > 0) {
    const anchor = 137
    const d = cohort - anchor

    if (d >= 3) return { schoolName, gradeOrDivision: '入学前' }
    if (d <= -4) return { schoolName, gradeOrDivision: '卒業後' }

    switch (d) {
      case 2: return { schoolName, gradeOrDivision: '中1' }
      case 1: return { schoolName, gradeOrDivision: '中2' }
      case 0: return { schoolName, gradeOrDivision: '中3' }
      case -1: return { schoolName, gradeOrDivision: '高1' }
      case -2: return { schoolName, gradeOrDivision: '高2' }
      case -3: return { schoolName, gradeOrDivision: '高3' }
      default: return { schoolName, gradeOrDivision: '学年不明' }
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