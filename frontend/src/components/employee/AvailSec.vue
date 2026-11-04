<script setup>
import { ref } from 'vue'
import { useStore } from '../../store.js'
import { pointsFmt, fmtDateTime } from '../../utils.js'

const store = useStore()
const busyId = ref('')

async function accept(t) {
  if (!(await store.confirm('قبول المهمة', 'أسرع من يقبلها يحصل عليها — هل تريد قبولها الآن؟'))) return
  busyId.value = t.id
  try {
    const r = await store.acceptTask(t.id)
    store.toast(r.message || 'تم قبول المهمة', 'ok')
  } catch { /* التوست من store */ }
  busyId.value = ''
}
</script>

<template>
  <section>
    <h2 class="sec-title">🪙 المهام المتاحة</h2>
    <p class="muted">أسرع من يقبل المهمة يحصل عليها</p>

    <div class="cards-list">
      <div v-for="t in store.myAvailableTasks" :key="t.id" class="tcard" style="border-right-color: var(--info)">
        <div class="tcard-top">
          <span class="tcard-title">#{{ t.no }} {{ t.title }}</span>
          <span class="points-chip">⭐ {{ pointsFmt(t.points) }}</span>
          <span class="st" :class="'st-' + t.assignmentType">
            {{ t.assignmentType === 'direct' ? 'مكلّف بها' : (t.offeredTo && t.offeredTo.length ? '⚡ متاحة لك' : 'متاحة للجميع') }}
          </span>
        </div>
        <div class="tcard-meta">
          <span>🏷️ {{ t.taskTypeName }}</span>
          <span v-if="t.description">📝 {{ t.description }}</span>
          <span>🕒 {{ fmtDateTime(t.createdAt) }}</span>
          <span v-if="t.requiresReport">📄 تقرير مطلوب</span>
          <span v-if="t.requiresPhotos">📷 صور {{ t.photosMandatory ? 'إلزامية' : 'اختيارية' }}</span>
        </div>
        <div class="tcard-actions">
          <button class="btn btn-primary" :disabled="busyId === t.id" @click="accept(t)">🤝 قبول المهمة</button>
        </div>
      </div>
      <div v-if="!store.myAvailableTasks.length" class="empty">
        <span class="big">🪙</span>لا توجد مهام متاحة حالياً<br>
        <span class="muted">ستصلك مهام جديدة — وعليك إشعار تيليكرام إن كان مفعلاً</span>
      </div>
    </div>
  </section>
</template>
