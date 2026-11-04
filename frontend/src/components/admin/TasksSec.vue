<script setup>
import { ref, computed } from 'vue'
import { useStore } from '../../store.js'
import { TASK_STATUSES, statusLabel, statusClass, pointsFmt, fmtDateTime } from '../../utils.js'
import TaskFormModal from './TaskFormModal.vue'
import TaskDetailModal from './TaskDetailModal.vue'

const store = useStore()
const filter = ref('all')
const showForm = ref(false)
const editTask = ref(null)
const detailId = ref(null)

const counts = computed(() => {
  const c = { all: store.tasks.length }
  TASK_STATUSES.forEach(s => c[s] = store.tasks.filter(t => t.status === s).length)
  return c
})
const list = computed(() =>
  filter.value === 'all' ? store.tasks : store.tasks.filter(t => t.status === filter.value)
)

function borderColor(t) {
  if (t.status === 'completed') return 'var(--ok)'
  if (t.status === 'rejected' || t.status === 'cancelled') return 'var(--danger)'
  if (t.status === 'pending_approval') return 'var(--purple)'
  if (t.status === 'available') return 'var(--info)'
  return 'var(--warn)'
}

function openEdit(t) {
  detailId.value = null
  editTask.value = t
  showForm.value = true
}
</script>

<template>
  <section>
    <div class="sec-head">
      <h2 class="sec-title">🗂️ المهام</h2>
      <button class="btn btn-primary" @click="showForm = true">➕ مهمة جديدة</button>
    </div>

    <div class="filter-bar">
      <button v-for="[k, lbl] in [['all', 'الكل'], ...TASK_STATUSES.map(s => [s, statusLabel(s)])]" :key="k"
              class="chip" :class="{ active: filter === k }" @click="filter = k">
        {{ lbl }} ({{ counts[k] || 0 }})
      </button>
    </div>

    <div class="cards-list">
      <div v-for="t in list" :key="t.id" class="tcard" :style="{ borderRightColor: borderColor(t) }" @click="detailId = t.id">
        <div class="tcard-top">
          <span class="tcard-title">#{{ t.no }} {{ t.title }}</span>
          <span class="points-chip">⭐ {{ pointsFmt(t.points) }}</span>
          <span :class="statusClass(t.status)">{{ statusLabel(t.status) }}</span>
          <span class="st" :class="'st-' + t.assignmentType">{{ t.assignmentType === 'direct' ? 'تكليف مباشر' : 'تلقائي' }}</span>
        </div>
        <div class="tcard-meta">
          <span>🏷️ {{ t.taskTypeName }}</span>
          <span v-if="t.assignedToName">👤 {{ t.assignedToName }}</span>
          <span v-else-if="t.offeredTo && t.offeredTo.length">📤 معروضة على {{ t.offeredTo.length }} موظف</span>
          <span>🕒 {{ fmtDateTime(t.createdAt) }}</span>
          <span v-if="t.rejectionReason">❌ {{ t.rejectionReason }}</span>
        </div>
      </div>
      <div v-if="!list.length" class="empty"><span class="big">🗂️</span>لا توجد مهام بهذا التصنيف</div>
    </div>

    <TaskFormModal v-if="showForm" :edit-task="editTask" @close="showForm = false; editTask = null" />
    <TaskDetailModal v-if="detailId" :task-id="detailId" @close="detailId = null" @edit="openEdit(store.getTask($event))" />
  </section>
</template>
