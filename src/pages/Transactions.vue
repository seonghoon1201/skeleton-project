<template>
  <div class="transaction-page font-hakgyo">
    <div class="filter-bar font-hakgyo">
      <div class="day-title">
        {{ dayjs(dateRange[0]).format('YYYY.MM.DD') }} ~
        {{ dayjs(dateRange[1]).format('YYYY.MM.DD') }} / 총
        {{ transactionStore.filteredBudgets.length }}건
      </div>

      <div class="filters">
        <!-- ✅ 날짜 선택 UI -->
        <div class="custom-date-container">
          <button @click="moveMonth(-1)" class="arrow-btn">
            <i class="fa fa-chevron-left"></i>
          </button>
          <VueDatePicker
            v-model="dateRange"
            range
            format="yyyy-MM-dd"
            :teleport="true"
            :clearable="false"
            :enable-time-picker="false"
            @update:model-value="applyFilters"
            class="date-picker-box"
          />
          <button @click="moveMonth(1)" class="arrow-btn">
            <i class="fa fa-chevron-right"></i>
          </button>
        </div>

        <!-- 카테고리 -->
        <div class="category-filter">
          <select v-model="selectedCategory" @change="updateCategory">
            <option value="">전체 카테고리</option>
            <optgroup label="수입">
              <option
                v-for="income in incomeCategories"
                :key="income.id"
                :value="income.name"
              >
                {{ income.name }}
              </option>
            </optgroup>
            <optgroup label="지출">
              <option
                v-for="expense in expenseCategories"
                :key="expense.id"
                :value="expense.name"
              >
                {{ expense.name }}
              </option>
            </optgroup>
          </select>
        </div>
      </div>
    </div>

    <!-- 요약 -->
    <div class="summary font-hakgyo">
      <div class="summary-box">
        <p>수입</p>
        {{ format(summary.income) }}
      </div>
      <div class="summary-box">
        <p>지출</p>
        {{ format(summary.expense) }}
      </div>
      <div class="summary-box">
        <p>순이익</p>
        {{ format(summary.net) }}
      </div>
    </div>

    <!-- 리스트 -->
    <TransactionList
      :transactions="groupedBudgets"
      @edit="openEditModal"
      @delete="deleteBudget"
    />

    <!-- 거래 추가 버튼 -->
    <button
      type="button"
      class="btn rounded-pill px-4 py-2 text-black d-flex align-items-center gap-2"
      style="
        position: fixed;
        bottom: 20px;
        right: 20px;
        z-index: 1000;
        background-color: #b3e5fc;
      "
      @click="openAddModal"
      data-bs-toggle="modal"
      data-bs-target="#addModal"
    >
      <i class="fa-solid fa-pen-to-square"></i> 거래 추가
    </button>

    <!-- 모달 -->
    <Teleport to="body">
      <TransactionModal
        @close="modalVisible = false"
        @added="transactionStore.fetchBudgets()"
      />
      <TransactionEditModal
        :selectedBudget="selectedBudget"
        @close="modalVisible = false"
        @updated="
          (updatedBudget) =>
            transactionStore.updateBudgets(updatedBudget.id, updatedBudget)
        "
      />
    </Teleport>
  </div>
</template>

<script setup>
import { useTransactionStore } from '@/store/transactionStore';
import { useCategoryStore } from '@/store/categoryStore';
import TransactionList from '@/components/Transaction/TransactionList.vue';
import TransactionModal from '@/components/Transaction/TransactionModal.vue';
import TransactionEditModal from '@/components/Transaction/TransactionEditModal.vue';
import { ref, computed, onMounted } from 'vue';
import VueDatePicker from '@vuepic/vue-datepicker';
import * as bootstrap from 'bootstrap';
import dayjs from 'dayjs';

// 스토어
const transactionStore = useTransactionStore();
const categoryStore = useCategoryStore();

// 날짜 초기값
const dateRange = ref([
  dayjs().startOf('month').toDate(),
  dayjs().endOf('month').toDate(),
]);

const modalVisible = ref(false);
const selectedBudget = ref(null);
const selectedCategory = ref('');

// 계산된 값
const groupedBudgets = computed(() => transactionStore.groupByDate);
const summary = computed(() => transactionStore.summary);
const incomeCategories = computed(() => categoryStore.incomeCategories);
const expenseCategories = computed(() => categoryStore.expenseCategories);

// 초기 로딩
onMounted(async () => {
  await transactionStore.fetchBudgets();
  await categoryStore.fetchCategories();
  applyFilters();
});

// 메서드
function openAddModal() {
  selectedBudget.value = null;
  modalVisible.value = true;
}
function openEditModal(budget) {
  selectedBudget.value = budget;
  const modal = new bootstrap.Modal(document.getElementById('modifyModal'));
  modal.show();
  modalVisible.value = true;
}
async function deleteBudget(id) {
  const confirmed = window.confirm('삭제하시겠습니까?');
  if (!confirmed) return;
  await transactionStore.deleteBudget(id);
}
function moveMonth(offset) {
  const currentStart = dayjs(dateRange.value[0]);
  const newStart = currentStart.add(offset, 'month').startOf('month');
  const newEnd = newStart.endOf('month');
  dateRange.value = [newStart.toDate(), newEnd.toDate()];
  applyFilters();
}
function updateCategory() {
  transactionStore.setCategoryFilter(selectedCategory.value);
}
function applyFilters() {
  const [start, end] = dateRange.value;
  if (!start || !end) return;
  transactionStore.setDateRange(
    dayjs(start).format('YYYY-MM-DD'),
    dayjs(end).format('YYYY-MM-DD')
  );
  transactionStore.setCategoryFilter(selectedCategory.value);
}
function format(value) {
  return parseInt(value).toLocaleString() + '원';
}
</script>

<style scoped>
.transaction-page {
  padding: 20px;
  font-family: 'Pretendard', sans-serif;
  background-color: #fff;
}

.day-title {
  text-align: center;
  font-size: 22px;
  font-weight: 500;
  color: #444;
  margin-bottom: 10px;
}

.filters {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
}
@media (min-width: 640px) {
  .filters {
    flex-direction: row;
    justify-content: center;
    align-items: center;
  }
}

.custom-date-container {
  display: flex;
  align-items: center;
  justify-content: center;
  background: #fff;
  border: 2px solid #cce7ff;
  border-radius: 12px;
  padding: 8px 16px;
  gap: 12px;
  min-height: 40px;
}

.date-picker-box {
  width: 260px;
  font-size: 15px;
  border: none !important;
  box-shadow: none !important;
  display: flex;
  align-items: center;
  justify-content: center;
}

.date-picker-box ::v-deep(.dp__input) {
  border: none !important;
  background-image: none !important;
  background: transparent !important;
  padding: 0 !important;
  box-shadow: none !important;
  font-size: 16px;
  text-align: center !important;
  color: #333;
  width: 100%;
}

.date-picker-box ::v-deep(svg) {
  display: none !important;
}

.arrow-btn {
  background: none;
  border: none;
  color: #4285f4;
  font-size: 16px;
  cursor: pointer;
}

.category-filter select {
  padding: 6px 10px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 14px;
}

.summary {
  display: flex;
  justify-content: space-between;
  gap: 12px;
  margin: 16px 0;
}

.summary-box {
  flex: 1;
  background: #e6f3ff;
  border: 2px solid #b4daff;
  border-radius: 12px;
  padding: 12px;
  text-align: center;
  font-weight: bold;
  color: #333;
}
.summary-box p {
  margin-bottom: 6px;
  font-size: 14px;
}
</style>
