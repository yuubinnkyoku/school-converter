<template>
  <UContainer class="p-4">
    <UCard>
      <template #header>
        <h1 class="text-2xl font-bold">学校・学年変換ツール</h1>
      </template>

      <p class="mb-4">高校・大学の略称と数字の組み合わせ（例: TKB137）を入力してください。</p>

      <div class="flex gap-2">
        <UInput v-model="inputCode" placeholder="例: TKB137" class="flex-grow" />
        <UButton @click="convert">変換</UButton>
      </div>

      <template #footer v-if="result">
        <UAlert :title="result.school" :description="result.grade" />
      </template>
    </UCard>
  </UContainer>
</template>

<script setup lang="ts">
import { ref } from 'vue'

const inputCode = ref('TKB137')
const result = ref<{ school: string; grade: string } | null>(null)

// --- Data ---
interface School {
  name: string;
  // Formula: entryYearJHS1 = classNumber + entryYearConstant
  entryYearConstant: number;
}

const schoolData: Record<string, School> = {
  'TKB': {
    name: '筑波大学附属中学校・高等学校',
    entryYearConstant: 1884, // Calculated based on TKB137 being HS1 in academic year 2024
  },
  'KSEI': {
    name: '開成中学校・高等学校',
    entryYearConstant: 1871, // This is a placeholder value
  },
  'AZB': {
    name: '麻布中学校・高等学校',
    entryYearConstant: 1895, // This is a placeholder value
  }
}

const gradeNames = [
  '中学1年生', '中学2年生', '中学3年生',
  '高校1年生', '高校2年生', '高校3年生'
];

// --- Logic ---
const convert = () => {
  const code = inputCode.value.toUpperCase().trim();
  if (!code) {
    result.value = null;
    return;
  }

  const match = code.match(/^([A-Z]+)(\d+)$/);
  if (!match) {
    result.value = { school: '無効な形式です', grade: '例: TKB137' };
    return;
  }

  const [, schoolCode, classNumberStr] = match;
  const classNumber = parseInt(classNumberStr, 10);

  const school = schoolData[schoolCode];
  if (!school) {
    result.value = { school: '不明な学校コードです', grade: `コード: ${schoolCode}` };
    return;
  }

  // Calculate current academic year (starts in April)
  const now = new Date();
  const currentYear = now.getFullYear();
  const currentMonth = now.getMonth(); // 0-11
  const academicYear = currentMonth < 3 ? currentYear - 1 : currentYear;

  // Calculate grade
  const entryYearJHS1 = classNumber + school.entryYearConstant;
  const yearsSinceEntry = academicYear - entryYearJHS1;

  let grade = '';
  if (yearsSinceEntry >= 0 && yearsSinceEntry < gradeNames.length) {
    grade = gradeNames[yearsSinceEntry];
  } else if (yearsSinceEntry >= gradeNames.length) {
    grade = '卒業';
  } else {
    grade = '入学前';
  }

  result.value = {
    school: school.name,
    grade: grade
  };
}
</script>
