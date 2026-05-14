<script setup>
import { ref, computed, onMounted } from 'vue'
import Button from 'primevue/button'
import DataTable from 'primevue/datatable'
import Column from 'primevue/column'
import Tag from 'primevue/tag'
import { corporateCollaboratorApiService } from '../../infrastructure/corporate-collaborator-api.service'
import { apiService } from '../../../shared/infrastructure/api.service'

const collaborators = ref([])
const company = ref(null)
const loading = ref(true)
const sending = ref(false)
const sent = ref(false)

// Calculados basados en los estados de tu db.json ('ACTIVE', 'INVITED')
const sentCount = computed(() => collaborators.value.filter((c) => c.status === 'INVITED' || c.status === 'ACTIVE').length)
const pendingCount = computed(() => collaborators.value.filter((c) => c.status === 'INVITED').length)
const errorCount = computed(() => 0)
const remainingLicenses = computed(() => {
  if (!company.value) return 0
  return company.value.totalLicenses - company.value.usedLicenses
})

async function loadData() {
  loading.value = true
  try {
    const [cols, companies] = await Promise.all([
      corporateCollaboratorApiService.fetchCollaborators(),
      apiService.get('/companies'),
    ])
    collaborators.value = cols
    // Tomamos la empresa NexTech (id: 1) según tu db.json
    company.value = companies.find(c => c.id === 1) || companies[0]
  } finally {
    loading.value = false
  }
}

async function handleSendInvitations() {
  sending.value = true
  try {
    // Simulamos que al dar clic, todos los que faltan pasan a estar invitados
    collaborators.value = await corporateCollaboratorApiService.sendInvitations()
    sent.value = true
  } finally {
    sending.value = false
  }
}

function formatTime(isoString) {
  if (!isoString) return '—'
  return new Date(isoString).toLocaleTimeString('es-PE', { hour: '2-digit', minute: '2-digit' })
}

const getSeverity = (status) => {
  return status === 'ACTIVE' ? 'success' : 'warning';
};

const getStatusLabel = (status) => {
  return status === 'ACTIVE' ? 'Activo' : 'Pendiente aceptar';
};

onMounted(loadData)
</script>

<template>
  <section class="bt-corporate-dashboard">

    <header class="bt-dashboard-heading">
      <div>
        <p class="microcopy">US15 · ESCENARIO 1</p>
        <h1>Envío de invitaciones</h1>
        <p class="text-muted">Correos enviados a todos los colaboradores</p>
      </div>
      <div class="bt-progress-badge" :class="{ 'bt-progress-badge--done': sent }">
        <span>{{ sentCount }}/{{ collaborators.length }}</span>
        <span v-if="sent">✓</span>
      </div>
    </header>

    <p v-if="loading" class="text-muted">Cargando colaboradores...</p>

    <template v-else>

      <div v-if="sent" class="bt-panel-note" style="border-radius: 8px; padding: 12px 16px; background: var(--bt-success-soft); border-color: var(--bt-success); margin-bottom: 20px;">
        ✓ <strong>{{ sentCount }} invitaciones enviadas</strong> — Cada colaborador recibirá sus credenciales de acceso.
      </div>

      <section class="bt-kpi-grid mb-4">
        <article class="bt-kpi-card bt-kpi-card--primary">
          <span>Enviadas OK</span>
          <strong>{{ sentCount }}</strong>
        </article>
        <article class="bt-kpi-card">
          <span>Pendientes de aceptar</span>
          <strong>{{ pendingCount }}</strong>
        </article>
        <article class="bt-kpi-card">
          <span>Con error</span>
          <strong style="color: var(--bt-danger)">{{ errorCount }}</strong>
        </article>
        <article class="bt-kpi-card bt-kpi-card--success">
          <span>Licencias restantes</span>
          <strong>{{ remainingLicenses }}</strong>
        </article>
      </section>

      <div v-if="!sent" class="mb-4">
        <Button
            :label="sending ? 'Enviando...' : `Enviar invitaciones a todos (${collaborators.length})`"
            :disabled="sending"
            class="bt-dashboard-export p-button-primary"
            @click="handleSendInvitations"
        />
      </div>

      <div class="card mt-4">
        <DataTable :value="collaborators" responsiveLayout="scroll" :paginator="true" :rows="5" class="p-datatable-sm shadow-2 border-round">
          <Column field="name" header="COLABORADOR" :sortable="true"></Column>
          <Column field="email" header="EMAIL"></Column>
          <Column field="status" header="ESTADO">
            <template #body="slotProps">
              <Tag :value="getStatusLabel(slotProps.data.status)" :severity="getSeverity(slotProps.data.status)" />
            </template>
          </Column>
          <Column field="sentAt" header="ENVIADO">
            <template #body="slotProps">
              {{ slotProps.data.sentAt ? 'Hoy ' + formatTime(slotProps.data.sentAt) : '—' }}
            </template>
          </Column>
        </DataTable>
      </div>

    </template>
  </section>
</template>