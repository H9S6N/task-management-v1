<script setup>
import { useStore } from '../../store.js'
import { pointsFmt, statusLabel, statusClass } from '../../utils.js'

const store = useStore()
const stats = [
  ['👥', store.activeEmployees.filter(e => e.status === 'active').length, 'موظف نشط'],
  ['🪙', store.tasks.filter(t => t.status === 'available').length, 'مهمة متاحة'],
  ['🔧', store.tasks.filter(t => ['accepted', 'in_progress'].includes(t.status)).length, 'قيد التنفيذ'],
  ['⏳', store.pendingTasks.length, 'بانتظار الاعتماد'],
  ['✅', store.tasks.filter(t => t.status === 'completed').length, 'مكتملة'],
  ['⭐', pointsFmt(store.tasks.filter(t => t.pointsAwarded).reduce((s, t) => s + (Number(t.points) || 0), 0)), 'مجموع النقاط الموزعة'],
]
</script>

<template>
  <section>
    <h2 class="sec-title">📊 لوحة التحكم</h2>
    <div class="stats-grid">
      <div v-for="(s, i) in stats" :key="i" class="stat">
        <div class="num">{{ s[0] }} {{ s[1] }}</div>
        <div class="lbl">{{ s[2] }}</div>
      </div>
    </div>
    <div class="dash-cols">
      <div class="card">
        <h3>⏳ بانتظار الاعتماد</h3>
        <div class="mini-list">
          <template v-if="store.pendingTasks.length">
            <div v-for="t in store.pendingTasks.slice(0, 6)" :key="t.id" class="mini-item" @click="store.setSection('approvals')">
              <span class="t">#{{ t.no }} {{ t.title }}</span>
              <span :class="statusClass(t.status)">{{ statusLabel(t.status) }}</span>
            </div>
            <p v-if="store.pendingTasks.length > 6" class="muted">+{{ store.pendingTasks.length - 6 }} أخرى في قسم الاعتماد</p>
          </template>
          <div v-else class="empty"><span class="big">🎉</span>لا يوجد شيء بانتظار الاعتماد</div>
        </div>
      </div>
      <div class="card">
        <h3>🆕 أحدث المهام</h3>
        <div class="mini-list">
          <template v-if="store.tasks.length">
            <div v-for="t in store.tasks.slice(0, 6)" :key="t.id" class="mini-item" @click="store.setSection('tasks')">
              <span class="t">#{{ t.no }} {{ t.title }}</span>
              <span :class="statusClass(t.status)">{{ statusLabel(t.status) }}</span>
            </div>
          </template>
          <div v-else class="empty"><span class="big">🗂️</span>لا توجد مهام بعد — أنشئ أول مهمة</div>
        </div>
      </div>
    </div>
  </section>
</template>
