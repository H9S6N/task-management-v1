<script setup>
/* تفاصيل مهمة + التقارير المرفوعة + الاعتماد/الرفض/الإلغاء */
import { ref, computed, onMounted } from 'vue'
import { useStore } from '../../store.js'
import { statusLabel, statusClass, pointsFmt, fmtDateTime } from '../../utils.js'
import { loadPhotoUrl } from '../../api.js'
import Modal from '../Modal.vue'

const store = useStore()
const props = defineProps({ taskId: String, openReject: { type: Boolean, default: false } })
const emit = defineEmits(['close', 'edit'])

const task = computed(() => store.getTask(props.taskId))
const subs = ref([])
const photoUrls = ref({})   // "subNo-idx" -> blob url
const showReject = ref(props.openReject)
const rejectReason = ref('')
const confirmAction = ref(null)   // 'approve' | 'cancel' | 'unassign'

onMounted(async () => {
  try {
    const r = await store.getSubmissions(props.taskId)
    subs.value = r.submissions || []
    for (const s of subs.value) {
      (s.photos || []).forEach(async (p, i) => {
        try { photoUrls.value[`${s.submissionNumber}-${i}`] = await loadPhotoUrl(p) } catch { /* */ }
      })
    }
  } catch (e) { /* */ }
})

const timeline = computed(() => {
  const t = task.value
  if (!t) return []
  return [
    t.createdAt && `🕒 أُنشئت: ${fmtDateTime(t.createdAt)}`,
    t.acceptedAt && `🤝 قُبلت: ${fmtDateTime(t.acceptedAt)}`,
    t.submittedAt && `📤 رُفع التقرير: ${fmtDateTime(t.submittedAt)}`,
    t.reviewedAt && `${t.status === 'completed' ? '✅' : '❌'} رُوجعت: ${fmtDateTime(t.reviewedAt)}`,
    t.completedAt && `🏁 اكتملت: ${fmtDateTime(t.completedAt)}`,
  ].filter(Boolean)
})

async function doApprove() {
  confirmAction.value = null
  await store.approveTask(props.taskId)
  emit('close')
}
async function doReject() {
  if (rejectReason.value.trim().length < 3) { store.toast('اكتب سبب الرفض', 'err'); return }
  await store.rejectTask(props.taskId, rejectReason.value.trim())
  showReject.value = false
  emit('close')
}
async function doUnassign() {
  confirmAction.value = null
  await store.unassignTask(props.taskId)
  emit('close')
}
async function doCancel() {
  confirmAction.value = null
  await store.cancelTask(props.taskId)
  emit('close')
}
</script>

<template>
  <Modal @close="emit('close')">
    <template v-if="task">
      <h3>#{{ task.no }} {{ task.title }}</h3>
      <div class="tcard-top" style="margin-bottom:8px">
        <span :class="statusClass(task.status)">{{ statusLabel(task.status) }}</span>
        <span class="points-chip">⭐ {{ pointsFmt(task.points) }}</span>
        <span class="st" :class="'st-' + task.assignmentType">{{ task.assignmentType === 'direct' ? 'تكليف مباشر' : 'تلقائي' }}</span>
      </div>

      <div class="detail-sec">
        <div class="lbl">التفاصيل</div>
        <p>🏷️ النوع: {{ task.taskTypeName }} {{ store.getTaskType(task.taskTypeId)?.isActive === false ? '(متوقف)' : '' }}</p>
        <p v-if="task.description">📝 {{ task.description }}</p>
        <p>👤 الموظف: <b>{{ task.assignedToName || '— لم يُقبل بعد' }}</b></p>
        <p v-if="task.requiresReport">📄 يتطلب تقرير</p>
        <p v-if="task.requiresPhotos">📷 تتطلب صور {{ task.photosMandatory ? '(إلزامية)' : '(اختيارية)' }}</p>
        <p v-if="task.pointsAwarded">⭐ النقاط أُضيفت</p>
      </div>

      <div class="detail-sec">
        <div class="lbl">الجدول الزمني</div>
        <div class="timeline"><div v-for="(line, i) in timeline" :key="i">{{ line }}</div></div>
      </div>

      <div v-if="task.rejectionReason" class="reject-box">سبب الرفض: {{ task.rejectionReason }}</div>

      <h3 style="margin-top:14px">📤 التقارير المرفوعة ({{ subs.length }})</h3>
      <div v-if="subs.length">
        <div v-for="s in subs" :key="s.submissionNumber" class="detail-sec">
          <div class="lbl">المحاولة {{ s.submissionNumber }} — {{ fmtDateTime(s.submittedAt) }}</div>
          <p v-if="s.reportText" style="margin-top:4px;white-space:pre-wrap">{{ s.reportText }}</p>
          <p v-else class="muted">بدون نص</p>
          <div v-if="s.photos && s.photos.length" class="photos-grid">
            <img v-for="(p, i) in s.photos" :key="i" :src="photoUrls[`${s.submissionNumber}-${i}`]"
                 alt="صورة" @click="store.showLightbox(photoUrls[`${s.submissionNumber}-${i}`])">
          </div>
        </div>
      </div>
      <div v-else class="detail-sec"><p class="muted">لا توجد تقارير مرفوعة بعد</p></div>

      <!-- تأكيدات -->
      <div v-if="confirmAction === 'approve'" class="detail-sec" style="background:var(--ok-bg)">
        سيُعتمد الإنجاز وتُضاف <b>{{ pointsFmt(task.points) }}</b> نقطة لـ {{ task.assignedToName }}. متابعة؟
        <div class="form-actions">
          <button class="btn btn-primary" @click="doApprove">✅ نعم، اعتماد</button>
          <button class="btn btn-outline" @click="confirmAction = null">تراجع</button>
        </div>
      </div>
      <div v-if="confirmAction === 'unassign'" class="detail-sec">
        ستعود المهمة إلى "متاحة" ويمكن لأي موظف مؤهل قبولها من جديد. متابعة؟
        <div class="form-actions">
          <button class="btn btn-primary" @click="doUnassign">نعم</button>
          <button class="btn btn-outline" @click="confirmAction = null">تراجع</button>
        </div>
      </div>
      <div v-if="confirmAction === 'cancel'" class="detail-sec" style="background:var(--danger-bg)">
        سيتم إلغاء المهمة نهائياً. متابعة؟
        <div class="form-actions">
          <button class="btn btn-danger" @click="doCancel">نعم، إلغاء المهمة</button>
          <button class="btn btn-outline" @click="confirmAction = null">تراجع</button>
        </div>
      </div>

      <!-- نموذج الرفض -->
      <template v-if="showReject">
        <h3 style="margin-top:14px">❌ رفض إنجاز المهمة #{{ task.no }}</h3>
        <p class="muted">سيُعاد الأمر للموظف {{ task.assignedToName || '' }} لتعديل التقرير وإعادة إرساله</p>
        <label>سبب الرفض (إلزامي)</label>
        <textarea v-model="rejectReason" rows="3" placeholder="مثال: الصور غير واضحة، أكمل التقرير..."></textarea>
        <div class="form-actions">
          <button class="btn btn-danger" @click="doReject">رفض المهمة</button>
          <button class="btn btn-outline" @click="showReject = false">إلغاء</button>
        </div>
      </template>

      <div v-else class="form-actions">
        <template v-if="task.status === 'pending_approval'">
          <button class="btn btn-primary" @click="confirmAction = 'approve'">✅ اعتماد وإضافة النقاط</button>
          <button class="btn btn-danger" @click="showReject = true">❌ رفض</button>
        </template>
        <template v-if="['accepted', 'in_progress'].includes(task.status)">
          <button class="btn btn-outline" @click="confirmAction = 'unassign'">↩️ إلغاء قبول الموظف</button>
        </template>
        <template v-if="!['completed', 'cancelled'].includes(task.status)">
          <button class="btn btn-outline" @click="emit('edit', task.id)">✏️ تعديل المهمة</button>
          <button class="btn btn-ghost" @click="confirmAction = 'cancel'">🚫 إلغاء المهمة</button>
        </template>
        <button class="btn btn-outline" @click="emit('close')">إغلاق</button>
      </div>
    </template>
  </Modal>
</template>
