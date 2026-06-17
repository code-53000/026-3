<script setup lang="ts">
import { computed } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { ArrowLeft } from 'lucide-vue-next';
import { useKiteStore } from '@/composables/useKiteStore';
import { useFlightStore } from '@/composables/useFlightStore';
import { formatWingspan, formatDate, formatDuration, getKiteTypeEmoji } from '@/utils/format';
import type { Kite } from '@/types';

const route = useRoute();
const router = useRouter();
const kiteStore = useKiteStore();
const flightStore = useFlightStore();

const kiteIds = computed(() => {
  const idsParam = route.query.ids as string;
  if (!idsParam) return [];
  return idsParam.split(',');
});

const kites = computed(() => {
  return kiteIds.value
    .map((id) => kiteStore.getKiteById(id))
    .filter((k): k is Kite => k !== undefined);
});

interface FlightStats {
  totalFlights: number;
  avgPerformance: number | null;
  totalDuration: number;
  avgDuration: number | null;
  lastFlightDate: string | null;
}

function getFlightStats(kiteId: string): FlightStats {
  const flights = flightStore.getFlightsByKiteId(kiteId);
  if (flights.length === 0) {
    return {
      totalFlights: 0,
      avgPerformance: null,
      totalDuration: 0,
      avgDuration: null,
      lastFlightDate: null,
    };
  }
  const totalPerf = flights.reduce((sum, f) => sum + f.performance, 0);
  const totalDur = flights.reduce((sum, f) => sum + f.duration, 0);
  return {
    totalFlights: flights.length,
    avgPerformance: Math.round((totalPerf / flights.length) * 10) / 10,
    totalDuration: totalDur,
    avgDuration: Math.round((totalDur / flights.length) * 10) / 10,
    lastFlightDate: flights[0].date,
  };
}

const statsMap = computed(() => {
  const map = new Map<string, FlightStats>();
  kites.value.forEach((k) => {
    map.set(k.id, getFlightStats(k.id));
  });
  return map;
});

type CompareDirection = 'higher' | 'lower';

interface CompareRow {
  label: string;
  values: string[];
  bestIndices: number[];
  isNumeric: boolean;
}

const compareRows = computed(() => {
  const rows: CompareRow[] = [];

  rows.push({
    label: '名称',
    values: kites.value.map((k) => k.name),
    bestIndices: [],
    isNumeric: false,
  });

  rows.push({
    label: '类型',
    values: kites.value.map((k) => k.type),
    bestIndices: [],
    isNumeric: false,
  });

  rows.push({
    label: '骨架材料',
    values: kites.value.map((k) => k.frameMaterial),
    bestIndices: [],
    isNumeric: false,
  });

  {
    const vals = kites.value.map((k) => k.wingspan);
    const display = vals.map((v) => formatWingspan(v));
    const maxVal = Math.max(...vals);
    rows.push({
      label: '翼展',
      values: display,
      bestIndices: vals.reduce<number[]>((acc, v, i) => v === maxVal ? [...acc, i] : acc, []),
      isNumeric: true,
    });
  }

  rows.push({
    label: '适合风力档位',
    values: kites.value.map((k) => k.suitableWindLevel),
    bestIndices: [],
    isNumeric: false,
  });

  {
    const vals = kites.value.map((k) => k.repairCount);
    const display = vals.map((v) => `${v} 次`);
    const minVal = Math.min(...vals);
    rows.push({
      label: '修复次数',
      values: display,
      bestIndices: vals.reduce<number[]>((acc, v, i) => v === minVal ? [...acc, i] : acc, []),
      isNumeric: true,
    });
  }

  {
    const vals = kites.value.map((k) => statsMap.value.get(k.id)!.totalFlights);
    const display = vals.map((v) => `${v} 次`);
    const maxVal = Math.max(...vals);
    rows.push({
      label: '总放飞次数',
      values: display,
      bestIndices: vals.reduce<number[]>((acc, v, i) => v === maxVal ? [...acc, i] : acc, []),
      isNumeric: true,
    });
  }

  {
    const vals = kites.value.map((k) => statsMap.value.get(k.id)!.avgPerformance);
    const display = vals.map((v) => v !== null ? `${v} ★` : '暂无数据');
    const numericVals = vals.filter((v): v is number => v !== null);
    const maxVal = numericVals.length > 0 ? Math.max(...numericVals) : null;
    const bestIndices = maxVal !== null
      ? vals.reduce<number[]>((acc, v, i) => v === maxVal ? [...acc, i] : acc, [])
      : [];
    rows.push({
      label: '平均表现',
      values: display,
      bestIndices,
      isNumeric: true,
    });
  }

  {
    const vals = kites.value.map((k) => statsMap.value.get(k.id)!.totalDuration);
    const display = vals.map((v) => formatDuration(v));
    const maxVal = Math.max(...vals);
    rows.push({
      label: '累计放飞时长',
      values: display,
      bestIndices: vals.reduce<number[]>((acc, v, i) => v === maxVal ? [...acc, i] : acc, []),
      isNumeric: true,
    });
  }

  {
    const vals = kites.value.map((k) => statsMap.value.get(k.id)!.avgDuration);
    const display = vals.map((v) => v !== null ? formatDuration(v) : '暂无数据');
    const numericVals = vals.filter((v): v is number => v !== null);
    const maxVal = numericVals.length > 0 ? Math.max(...numericVals) : null;
    const bestIndices = maxVal !== null
      ? vals.reduce<number[]>((acc, v, i) => v === maxVal ? [...acc, i] : acc, [])
      : [];
    rows.push({
      label: '平均放飞时长',
      values: display,
      bestIndices,
      isNumeric: true,
    });
  }

  {
    const vals = kites.value.map((k) => statsMap.value.get(k.id)!.lastFlightDate);
    const display = vals.map((v) => v !== null ? formatDate(v) : '暂无数据');
    const dateVals = vals.filter((v): v is string => v !== null);
    const maxDate = dateVals.length > 0 ? dateVals.reduce((a, b) => a > b ? a : b) : null;
    const bestIndices = maxDate !== null
      ? vals.reduce<number[]>((acc, v, i) => v === maxDate ? [...acc, i] : acc, [])
      : [];
    rows.push({
      label: '最近放飞日期',
      values: display,
      bestIndices,
      isNumeric: true,
    });
  }

  return rows;
});

function goBack() {
  router.back();
}

function viewDetail(id: string) {
  router.push(`/kites/${id}`);
}
</script>

<template>
  <div>
    <button
      @click="goBack"
      class="flex items-center gap-2 text-ink-light hover:text-primary mb-6 transition-colors"
    >
      <ArrowLeft class="w-5 h-5" />
      返回列表
    </button>

    <div class="mb-6">
      <h1 class="text-2xl font-serif font-bold text-ink">风筝对比</h1>
      <p class="text-ink-light text-sm mt-1">并排查看 {{ kites.length }} 只风筝的详细数据</p>
    </div>

    <div v-if="kites.length < 2" class="text-center py-16 bg-white rounded-xl shadow-soft border border-primary/5">
      <div class="text-5xl mb-4">🪁</div>
      <h3 class="text-lg font-serif font-semibold text-ink mb-2">请至少选择 2 只风筝进行对比</h3>
      <p class="text-ink-light mb-4">返回风筝列表，开启对比模式选择风筝</p>
      <button
        @click="router.push('/kites')"
        class="inline-flex items-center gap-2 px-5 py-2.5 bg-primary text-white rounded-lg hover:bg-primary-dark transition-colors"
      >
        前往风筝列表
      </button>
    </div>

    <div v-else class="bg-white rounded-xl shadow-soft border border-primary/5 overflow-hidden">
      <div class="overflow-x-auto">
        <table class="w-full min-w-[600px]">
          <thead>
            <tr class="bg-gradient-to-r from-primary/5 to-secondary/5 border-b border-primary/10">
              <th class="text-left px-5 py-4 text-sm font-semibold text-ink-light w-36 sticky left-0 bg-gradient-to-r from-primary/5 to-primary/5">
                对比项
              </th>
              <th
                v-for="kite in kites"
                :key="kite.id"
                class="px-5 py-4 text-center min-w-[160px]"
              >
                <div
                  class="cursor-pointer hover:text-primary transition-colors"
                  @click="viewDetail(kite.id)"
                >
                  <span class="text-3xl block mb-1">{{ getKiteTypeEmoji(kite.type) }}</span>
                  <span class="text-sm font-semibold text-ink">{{ kite.name }}</span>
                </div>
              </th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="(row, rowIdx) in compareRows"
              :key="row.label"
              :class="rowIdx % 2 === 0 ? 'bg-white' : 'bg-paper/30'"
            >
              <td class="px-5 py-3.5 text-sm font-medium text-ink-light sticky left-0" :class="rowIdx % 2 === 0 ? 'bg-white' : 'bg-paper/30'">
                {{ row.label }}
              </td>
              <td
                v-for="(val, colIdx) in row.values"
                :key="colIdx"
                class="px-5 py-3.5 text-sm text-center"
                :class="{
                  'font-semibold': row.bestIndices.includes(colIdx),
                  'text-accent': row.bestIndices.includes(colIdx) && row.isNumeric,
                  'text-ink': !row.bestIndices.includes(colIdx) || !row.isNumeric,
                  'text-ink-light': val === '暂无数据',
                  'italic': val === '暂无数据',
                }"
              >
                <span
                  v-if="row.bestIndices.includes(colIdx) && row.isNumeric"
                  class="inline-flex items-center gap-1"
                >
                  <span class="text-accent text-xs">▲</span>
                  {{ val }}
                </span>
                <template v-else>{{ val }}</template>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>
