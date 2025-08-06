<template>
  <UApp>
    <UContainer class="py-10">
      <UCard>
        <template #header>
          <div class="flex items-center justify-between">
            <h1 class="text-xl font-semibold">学校コード変換ツール</h1>
            <UBadge color="primary" variant="soft">Nuxt UI</UBadge>
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
                <span>{{ result.schoolName }}</span>
              </div>
              <div class="flex items-center gap-2">
                <UIcon name="i-heroicons-user" class="text-primary" />
                <span class="font-medium">区分:</span>
                <span>{{ result.gradeOrDivision }}</span>
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
  const input = (inputRaw ?? '').trim().toUpperCase()
  const m = input.match(/^([A-Z]{2,4})(\d{1,3})$/)
  if (!m) return null

  const abbr = m[1] as string
  const num = Number(m[2])

  const schoolMap = {
    TKB: '筑波大学附属中学校・高等学校',
    WSD: '早稲田大学',
    KGU: '関西学院大学'
  } as const

  type Abbr = keyof typeof schoolMap
  const isKnownAbbr = (a: string): a is Abbr => (a as string) in schoolMap

  if (!isKnownAbbr(abbr)) return null
  const schoolName: (typeof schoolMap)[Abbr] = schoolMap[abbr]

  // 校別ルーティング
  if (abbr === 'TKB') {
    // 100-199: 中学, 200-299: 高校 とする暫定ルール
    if (num >= 100 && num < 200) {
      const g = num % 10 || 1
      return { schoolName: String(schoolName), gradeOrDivision: `中学${g}年生` }
    }
    if (num >= 200 && num < 300) {
      const g = num % 10 || 1
      return { schoolName: String(schoolName), gradeOrDivision: `高校${g}年生` }
    }
    // 2桁やその他は暫定で中学の学年に丸める
    if (num < 100) {
      const g = Math.max(1, Math.min(3, num % 10 || 1))
      return { schoolName: String(schoolName), gradeOrDivision: `中学${g}年生` }
    }
    return { schoolName: String(schoolName), gradeOrDivision: '学年不明' }
  }

  if (abbr === 'WSD' || abbr === 'KGU') {
    // 大学 学部 学年=末尾1桁（暫定）
    const g = num % 10 || 1
    return { schoolName: String(schoolName), gradeOrDivision: `学部${g}年生` }
  }

  return null
}

function onSubmit() {
  lastInput.value = state.input
  result.value = convert(state.input)
}
</script>

<style scoped>
/* 追加の軽微なスタイル（必要なら） */
</style>