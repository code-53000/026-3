<script setup lang="ts">
import { ref, computed } from 'vue';
import { useRouter } from 'vue-router';
import { Plus, GitCompareArrows, X, Check } from 'lucide-vue-next';
import { useKiteStore } from '@/composables/useKiteStore';
import { useFlightStore } from '@/composables/useFlightStore';
import { useFilter } from '@/composables/useFilter';
import KiteFilter from '@/components/kite/KiteFilter.vue';
import KiteCard from '@/components/kite/KiteCard.vue';
import type { Kite } from '@/types';

const router = useRouter();
const kiteStore = useKiteStore();
const flightStore = useFlightStore();
const { filters, hasActiveFilters, setKiteType, setSuitableWindLevel, setSearchText, clearFilters } = useFilter();

const compareMode = ref(false);
const selectedIds = ref<Set<string>>(new Set());

const selectedCount = computed(() => selectedIds.value.size);

function toggleCompareMode() {
  compareMode.value = !compareMode.value;
  if (!compareMode.value) {
    selectedIds.value.clear();
  }
}

function toggleSelection(kiteId: string) {
  const next = new Set(selectedIds.value);
  if (next.has(kiteId)) {
    next.delete(kiteId);
  } else {
    if (next.size >= 4) return;
    next.add(kiteId);
  }
  selectedIds.value = next;
}

function isSelected(kiteId: string): boolean {
  return selectedIds.value.has(kiteId);
}

function startCompare() {
  if (selectedCount.value < 2) {
    alert('请至少选择 2 只风筝进行对比');
    return;
  }
  if (selectedCount.value > 4) {
    alert('最多只能选择 4 只风筝进行对比');
    return;
  }
  const ids = Array.from(selectedIds.value).join(',');
  router.push({ path: '/kites/compare', query: { ids } });
}

function getFlightCount(kiteId: string): number {
  return flightStore.getFlightsByKiteId(kiteId).length;
}

function handleEdit(id: string) {
  if (compareMode.value) return;
  router.push(`/kites/${id}/edit`);
}

function handleDelete(id: string) {
  if (compareMode.value) return;
  if (confirm('确定要删除这只风筝吗？相关的放飞记录也会被删除。')) {
    flightStore.deleteFlightsByKiteId(id);
    kiteStore.deleteKite(id);
    selectedIds.value.delete(id);
  }
}

function addNewKite() {
  router.push('/kites/new');
}
</script>

<template>
  <div>
    <div class="flex items-center justify-between mb-6">
      <div>
        <h1 class="text-2xl font-serif font-bold text-ink">风筝档案</h1>
        <p class="text-ink-light text-sm mt-1">记录您的每一只风筝</p>
      </div>
      <div class="flex items-center gap-3">
        <button
          @click="toggleCompareMode"
          :class="[
            'flex items-center gap-2 px-4 py-2.5 rounded-lg transition-colors shadow-soft',
            compareMode
              ? 'bg-accent text-white hover:bg-accent-dark'
              : 'bg-primary/10 text-primary hover:bg-primary/20'
          ]"
        >
          <GitCompareArrows class="w-5 h-5" />
          {{ compareMode ? '退出对比' : '风筝对比' }}
        </button>
        <button
          v-if="!compareMode"
          @click="addNewKite"
          class="flex items-center gap-2 px-5 py-2.5 bg-primary text-white rounded-lg hover:bg-primary-dark transition-colors shadow-soft"
        >
          <Plus class="w-5 h-5" />
          添加风筝
        </button>
      </div>
    </div>

    <div
      v-if="compareMode"
      class="bg-accent/10 border border-accent/20 rounded-xl p-4 mb-6 flex items-center justify-between"
    >
      <div class="text-sm text-ink">
        <span class="font-medium">已选择 {{ selectedCount }} / 4 只风筝</span>
        <span class="text-ink-light ml-2">（需选择 2 ~ 4 只）</span>
      </div>
      <div class="flex items-center gap-3">
        <button
          @click="selectedIds.clear()"
          class="px-3 py-1.5 text-sm text-ink-light hover:text-ink transition-colors"
        >
          清空选择
        </button>
        <button
          @click="startCompare"
          :class="[
            'flex items-center gap-2 px-4 py-2 rounded-lg transition-colors text-sm',
            selectedCount >= 2 && selectedCount <= 4
              ? 'bg-accent text-white hover:bg-accent-dark'
              : 'bg-gray-200 text-gray-400 cursor-not-allowed'
          ]"
          :disabled="selectedCount < 2 || selectedCount > 4"
        >
          <Check class="w-4 h-4" />
          开始对比
        </button>
      </div>
    </div>

    <KiteFilter
      v-model:searchText="filters.searchText"
      v-model:selectedType="filters.kiteType"
      v-model:selectedWindLevel="filters.suitableWindLevel"
      :hasActiveFilters="hasActiveFilters"
      @clear="clearFilters"
    />

    <div v-if="kiteStore.filteredKites(filters).length > 0" class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
      <div
        v-for="kite in kiteStore.filteredKites(filters)"
        :key="kite.id"
        class="relative"
      >
        <div
          v-if="compareMode"
          @click="toggleSelection(kite.id)"
          :class="[
            'absolute top-3 right-3 z-10 w-7 h-7 rounded-full border-2 flex items-center justify-center transition-all cursor-pointer',
            isSelected(kite.id)
              ? 'bg-accent border-accent text-white'
              : 'bg-white border-gray-300 hover:border-accent'
          ]"
        >
          <Check v-if="isSelected(kite.id)" class="w-4 h-4" />
        </div>
        <div :class="{ 'ring-2 ring-accent rounded-xl': compareMode && isSelected(kite.id) }">
          <KiteCard
            :kite="kite"
            :flightCount="getFlightCount(kite.id)"
            @edit="handleEdit"
            @delete="handleDelete"
          />
        </div>
      </div>
    </div>

    <div v-else class="text-center py-16">
      <div class="text-6xl mb-4">🪁</div>
      <h3 class="text-xl font-serif font-semibold text-ink mb-2">
        {{ hasActiveFilters ? '没有符合条件的风筝' : '还没有添加风筝' }}
      </h3>
      <p class="text-ink-light mb-6">
        {{ hasActiveFilters ? '试试调整筛选条件' : '开始记录您的第一只风筝吧' }}
      </p>
      <button
        v-if="!hasActiveFilters"
        @click="addNewKite"
        class="inline-flex items-center gap-2 px-6 py-3 bg-primary text-white rounded-lg hover:bg-primary-dark transition-colors"
      >
        <Plus class="w-5 h-5" />
        添加第一只风筝
      </button>
    </div>
  </div>
</template>
