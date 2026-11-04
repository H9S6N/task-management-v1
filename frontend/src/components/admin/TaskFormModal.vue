<script setup>
import { ref, computed, watch } from 'vue'
import { useStore } from '../../store.js'
import { pointsFmt, isOnLeaveNow } from '../../utils.js'
import Modal from '../Modal.vue'

const store = useStore()
const props = defineProps({ editTask: { type: Object, default: null } })
const emit = defineEmits(['close'])

const editing = computed(() => !!props.editTask)
const types = computed(() => store.listTaskTypes(true))
const activeEmps = computed(() => store.listEmployees(false).filter(e => e.status === 'active'))

const typeId = ref(props.editTask?.taskTypeId || types.value[0]?.id || '')
const title = ref(props.editTask?.title || '')
const desc = ref(props.editTask?.description || '')
const points = ref(props.editTask ? props.editTask.points : null)
const assign = ref(props.editTask?.assignmentType || 'auto')
const empId = ref(activeEmps.value[0]?.id || '')

/* التحكم بالتوزيع التلقائي — الافتراضي من الإعدادات */
const dist = ref(store.settings.defaultDistribution || 'lowest')
const distCount = ref(Number(store.settings.defaultCandidates) || 1)

watch(typeId, id => {
  const t = store.getTaskType(id)
  if (t && !editing.value) points.value = t.defaultPoints
}, { immediate: true })

/* المؤهلون فعلياً: نشط + ليس في إجازة + غير مستثنى من النوع */
const eligible = computed(() => typeId.value
  ? activeEmps.value.filter(e =>
      !isOnLeaveNow(e) && !(e.exceptions || []).includes(typeId.value))
  : [])

const preview = computed(() => {
  const arr = eligible.value.slice()
  if (dist.value === 'lowest') {
    arr.sort((a, b) => (Number(a.totalPoints) || 0) - (Number(b.totalPoints) || 0))
    return arr.slice(0, distCount.value)
  }
  if (dist.value === 'random') {
    return arr.slice().sort(() => Math.random() - 0.5).slice(0, distCount.value)
  }
  return arr
})

async function save() {
  if (!title.value.trim()) { store.toast('اكتب عنوان المهمة', 'err'); return }
  const type = store.getTaskType(typeId.value)
  if (!editing.value && type?.directOnly && assign.value === 'auto') { store.toast('هذا النوع تكليف مباشر فقط', 'err'); return }
  try {
    const body = {
      taskTypeId: typeId.value, title: title.value.trim(),
      description: desc.value.trim(),
      points: points.value === '' || points.value == null ? null : Number(points.value),
    }
    if (editing.value) {
      await store.updateTask(props.editTask.id, body)
      store.toast('تم حفظ تعديلات المهمة ✅', 'ok')
    } else {
      body.assignmentType = assign.value
      body.createdBy = store.auth.name
      if (assign.value === 'direct') {
        body.offeredTo = [empId.value]
      } else {
        body.offeredTo = preview.value.map(e => e.id)
        body.distribution = dist.value
        body.candidateCount = distCount.value
      }
      const r = await store.createTask(body)
      store.toast(`تم إنشاء المهمة #${r.task.no} ✅ — الإشعارات تُرسل من الخادم تلقائياً 📩`, 'ok')
    }
    emit('close')
  } catch { /* التوست يظهر من store */ }
}
</script>

<template>
  <Modal @close="emit('close')">
    <h3>{{ editing ? `✏️ تعديل المهمة #${editTask.no}` : '➕ مهمة جديدة' }}</h3>

    <label>نوع المهمة</label>
    <select v-model="typeId" :disabled="editing">
      <option v-for="t in types" :key="t.id" :value="t.id">
        {{ t.name }} (⭐{{ pointsFmt(t.defaultPoints) }}{{ t.directOnly ? ' — تكليف مباشر فقط' : '' }})
      </option>
    </select>

    <label>عنوان المهمة</label>
    <input v-model="title" placeholder="مثال: كشف سوق المنصور" maxlength="120">

    <label>الوصف (اختياري)</label>
    <textarea v-model="desc" rows="2" placeholder="تفاصيل المهمة..."></textarea>

    <div class="grid-2">
      <div>
        <label>النقاط</label>
        <input v-model="points" type="number" step="0.5" min="0">
      </div>
      <div v-if="!editing">
        <label>طريقة التوزيع</label>
        <select v-model="assign">
          <option value="auto">📤 توزيع تلقائي</option>
          <option value="direct">👤 تكليف مباشر</option>
        </select>
      </div>
    </div>

    <!-- التحكم الكامل بالتوزيع التلقائي -->
    <template v-if="!editing && assign === 'auto'">
      <div class="grid-2">
        <div>
          <label>نمط التوزيع</label>
          <select v-model="dist">
            <option value="lowest">⬇️ صاحب الأقل نقاطاً أولاً</option>
            <option value="random">🎲 عشوائي</option>
            <option value="all">👥 كل المؤهلين</option>
          </select>
        </div>
        <div v-if="dist !== 'all'">
          <label>عدد المعروض عليهم (1 - {{ eligible.length }})</label>
          <input v-model.number="distCount" type="number" min="1" :max="Math.max(1, eligible.length)">
        </div>
      </div>

      <div class="detail-sec">
        <template v-if="preview.length">
          📤 ستُعرض المهمة على {{ preview.length }} من {{ eligible.length }} مؤهل:
          <b>{{ preview.map(c => c.fullName + ' (⭐' + pointsFmt(c.totalPoints) + ')').join('، ') }}</b><br>
          <span class="muted">سيصلهم إشعار تيليكرام فوراً من الخادم</span>
          <span v-if="dist === 'random'" class="muted"> — الاختيار العشوائي النهائي يتم عند الخادم</span>
        </template>
        <template v-else>
          ⚠️ <b>لا يوجد موظفون مؤهلون</b> لهذا النوع حالياً (إجازة/استثناء/توقف) — المهمة ستُنشأ متاحة وستُبلغ بالإشعار
        </template>
      </div>
    </template>

    <div v-if="!editing && assign === 'direct'">
      <label>الموظف المكلف</label>
      <select v-model="empId">
        <option v-for="e in activeEmps" :key="e.id" :value="e.id">{{ e.fullName }} (⭐{{ pointsFmt(e.totalPoints) }})</option>
      </select>
    </div>

    <div class="form-actions">
      <button class="btn btn-primary" @click="save">{{ editing ? '💾 حفظ التعديلات' : 'إنشاء وإرسال' }}</button>
      <button class="btn btn-outline" @click="emit('close')">إلغاء</button>
    </div>
  </Modal>
</template>
