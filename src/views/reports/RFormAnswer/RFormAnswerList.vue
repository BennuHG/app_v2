<script setup>
import { ref, computed, watch, onMounted } from 'vue';
import { useRouter, useRoute } from 'vue-router';
import RFormAnswerFilters from './RFormAnswerFilters.vue';
import RFormAnswerTableCards from './RFormAnswerTableCards.vue';
import RFormAnswerCharts from './RFormAnswerCharts.vue';
import {
  mdiChevronUp,
  mdiChevronDown,
  mdiDotsHorizontal,
  mdiEye,
  mdiDomainOff,
  mdiArrowLeft,
  mdiFileChartCheckOutline,
  mdiMicrosoftExcel,
  mdiFilePdfBox
} from '@mdi/js';
import axios from '@/utils/axios';
import { useAuthStore } from '@/stores/auth';

const route = useRoute();
const router = useRouter();

const props = defineProps({
  formId: {
    type: [String, Number],
    required: false
  },
  isMobile: Boolean,
  isLoading: Boolean
});

const auth = useAuthStore();

function hasPermission(permission) {
  return Array.isArray(auth.user?.permissions) && auth.user.permissions.includes(permission);
}

const items = ref([]);
const loading = ref(false);

const sortBy = ref('folio');
const sortDesc = ref(false);
const page = ref(1);
const itemsPerPage = ref(10);

const formData = ref(null);

// NUEVO: Estado para la paginación del backend
const pagination = ref({
  current_page: 1,
  last_page: 1,
  per_page: 10,
  total: 0,
  from: null,
  to: null
});

// Filtros reactivos
const filters = ref({
  search: '',
  report_status: null,
  user_id: null,
  folio: '',
  score_min: null,
  score_max: null,
  start_date: null,
  end_date: null
});

const resolvedFormId = computed(() => {
  const val = props.formId ?? route.params.formId ?? route.params.form;
  return val && val !== '0' && val !== 0 && val !== '' ? val : null;
});

const hasRating = computed(() => {
  const raw = formData.value?.form?.has_rating ?? formData.value?.has_rating ?? false;
  return raw === true || raw === 1 || raw === '1';
});

const fetchFormData = async () => {
  if (!resolvedFormId.value) return;
  try {
    const { data } = await axios.get(`/forms/${resolvedFormId.value}`);
    formData.value = data.data || data.form || data;
  } catch (e) {
    formData.value = null;
  }
};

const fetchAnswers = async () => {
  loading.value = true;
  try {
    if (!resolvedFormId.value) {
      items.value = [];
      loading.value = false;
      return;
    }
    const params = {
      search: filters.value.search,
      report_status: filters.value.report_status,
      user_id: filters.value.user_id,
      folio: filters.value.folio,
      score_min: filters.value.score_min,
      score_max: filters.value.score_max,
      start_date: filters.value.start_date,
      end_date: filters.value.end_date,
      page: page.value,
      per_page: itemsPerPage.value
    };

    Object.keys(params).forEach((key) => {
      if (params[key] === null || params[key] === '' || params[key] === undefined) {
        delete params[key];
      }
    });

    const { data } = await axios.get(`/forms/${resolvedFormId.value}/responses/reports`, {
      params
    });

    // Guardar datos de paginación del backend
    pagination.value = {
      current_page: data.current_page ?? 1,
      last_page: data.last_page ?? 1,
      per_page: data.per_page ?? itemsPerPage.value,
      total: data.total ?? 0,
      from: data.from ?? null,
      to: data.to ?? null
    };

    // La respuesta backend ahora siempre es paginada y responses es un array
    const responses = Array.isArray(data.responses) ? data.responses : data.responses?.data || data.responses || [];

    console.log(responses);

    if (!responses.length) {
      items.value = [];
      loading.value = false;
      return;
    }

    const extractCIAC = (fieldResponses) => {
      if (!Array.isArray(fieldResponses)) return null;

      const ciacField = fieldResponses.find((fr) => {
        const fieldLabel = fr?.field_label || fr?.fieldLabel || fr?.field_name || '';
        return fieldLabel === 'Número de CIAC';
      });

      return ciacField?.value || null;
    };

    const extractOrigen = (fieldResponses) => {
      if (!Array.isArray(fieldResponses)) return null;

      const origenField = fieldResponses.find((fr) => {
        const fieldLabel = fr?.field_label || fr?.fieldLabel || fr?.field_name || fr?.form_field?.label || fr?.field?.label || '';
        return fieldLabel === 'Origen';
      });

      const raw = origenField?.value;
      if (raw === null || raw === undefined || raw === '') return null;

      let arr = raw;
      if (typeof raw === 'string') {
        const trimmed = raw.trim();
        if (trimmed.startsWith('[')) {
          try {
            arr = JSON.parse(raw);
          } catch {
            return raw;
          }
        } else if (trimmed.includes(',')) {
          arr = trimmed.split(',').map((s) => s.trim());
        } else {
          arr = trimmed ? [trimmed] : [];
        }
      }
      if (!Array.isArray(arr)) return String(raw);
      const joined = arr.filter(Boolean).join(', ');
      return joined || null;
    };

    const extractAddress = (resp, fieldResponses) => {
      const loc = resp.response?.location;
      if (loc?.address && typeof loc.address === 'string') return loc.address;
      if (resp.response?.address && typeof resp.response.address === 'string') return resp.response.address;
      if (!Array.isArray(fieldResponses)) return null;
      const geoField = fieldResponses.find((fr) => {
        const type = fr?.field?.type || fr?.form_field?.type || '';
        const label = (
          fr?.field_label ||
          fr?.fieldLabel ||
          fr?.field_name ||
          fr?.form_field?.label ||
          fr?.field?.label ||
          ''
        ).toLowerCase();
        return type === 'geolocation' || label.includes('dirección') || label.includes('direccion') || label.includes('address');
      });
      if (!geoField?.value) return null;
      const v = geoField.value;
      if (typeof v === 'string') {
        try {
          const parsed = JSON.parse(v);
          if (parsed?.address) return parsed.address;
          if (parsed?.street || parsed?.city)
            return [
              parsed.street,
              parsed.outdoor_number,
              parsed.neighborhood,
              parsed.city,
              parsed.state,
              parsed.postal_code,
              parsed.country
            ]
              .filter(Boolean)
              .join(', ');
        } catch {
          return v;
        }
      }
      if (typeof v === 'object' && v !== null) {
        if (v.address) return v.address;
        return [v.street, v.outdoor_number, v.neighborhood, v.city, v.state, v.postal_code, v.country].filter(Boolean).join(', ') || null;
      }
      return null;
    };

    // Solo mostrar reportes, no respuestas sin reporte
    items.value = responses.flatMap((resp) => {
      // Validación para evitar errores si resp.response o resp.reports no existen
      if (!resp.response || !resp.reports || !resp.reports.length) {
        return [];
      }

      const userName =
        resp.response?.user?.name ||
        resp.response?.user_name ||
        resp.user?.name ||
        resp.user_name ||
        (resp.response?.user && typeof resp.response.user === 'string' ? resp.response.user : undefined) ||
        '—';

      // Extraer field_responses para buscar el campo CIAC
      const fieldResponses = resp.response?.field_responses || resp.response?.fieldResponses || [];

      const base = {
        folio: resp.response.folio,
        name: userName,
        answer_date: resp.response.submitted_at || '—',
        score: resp.response.score,
        form_id: resp.response.form_id,
        additional_field_response: extractCIAC(fieldResponses),
        origen: extractOrigen(fieldResponses),
        address: extractAddress(resp, fieldResponses)
      };

      return resp.reports.map((report) => ({
        id: report.id, // id del reporte
        ...base,
        status: report.status,
        ponderacion:
          hasRating.value && report.ponderacion !== null && report.ponderacion !== undefined && Number(report.ponderacion) !== 0
            ? report.ponderacion
            : null,
        report,
        raw: { ...resp, report }
      }));
    });
  } catch (e) {
    console.error('Error al obtener reportes:', e);
    items.value = [];
    pagination.value = {
      current_page: 1,
      last_page: 1,
      per_page: itemsPerPage.value,
      total: 0,
      from: null,
      to: null
    };
  } finally {
    loading.value = false;
  }
};

// Redirección a 403 si no tiene permiso report.view
onMounted(async () => {
  if (!hasPermission('report.view')) {
    router.replace('/403');
    return;
  }
  await fetchFormData();
  await fetchAnswers();
});

const toggleSort = (column) => {
  if (sortBy.value === column) {
    sortDesc.value = !sortDesc.value;
  } else {
    sortBy.value = column;
    sortDesc.value = false;
  }
};

const formatDate = (dateString) => {
  if (!dateString) return '—';
  const date = new Date(dateString);
  return date.toLocaleDateString('es-ES', {
    day: '2-digit',
    month: '2-digit',
    year: 'numeric'
  });
};

const sortedItems = computed(() => {
  return [...(items.value || [])].sort((a, b) => {
    const aVal = a[sortBy.value]?.toString().toLowerCase() ?? '';
    const bVal = b[sortBy.value]?.toString().toLowerCase() ?? '';
    return aVal.localeCompare(bVal) * (sortDesc.value ? -1 : 1);
  });
});

// YA NO USAR paginación local, solo mostrar los items del backend
// const paginatedItems = computed(() => {
//   const start = (page.value - 1) * itemsPerPage.value;
//   return sortedItems.value.slice(start, start + itemsPerPage.value);
// });

const viewAnswer = ({ formId, reportId }) => {
  // Protección por permiso report.view
  if (!hasPermission('report.view')) {
    router.replace('/403');
    return;
  }
  router.push({
    name: 'Report Answer Show',
    params: { formId, reportId }
  });
};

const goToIndex = () => {
  router.push({ name: 'Reports' });
};

const closeReport = async (reportId) => {
  try {
    await axios.put(`/reports/${reportId}`, { status: 'closed' });
    await fetchAnswers();
  } catch (e) {
    console.error('Error cerrando el reporte:', e);
  }
};

const downloadReportPdf = async (answer) => {
  if (!answer?.id) return;
  try {
    const params = {};
    if (answer.local_created_at) params.local_created_at = answer.local_created_at;

    const res = await axios.get(`/reports/${answer.id}/download-pdf`, {
      params,
      responseType: 'arraybuffer'
    });
    const blob = new Blob([res.data], { type: 'application/pdf' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `Reporte_${answer.folio || answer.id}.pdf`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  } catch {
    alert('No se pudo descargar el PDF.');
  }
};

watch(
  () => [resolvedFormId.value],
  async () => {
    await fetchFormData();
    await fetchAnswers();
    page.value = 1;
  }
);

watch(
  () => ({ ...filters.value }),
  async () => {
    page.value = 1;
    await fetchAnswers();
  }
);

const handleSearch = (val) => {
  filters.value.search = val;
  page.value = 1;
  fetchAnswers();
};

const handleFilter = (newFilters) => {
  filters.value = { ...filters.value, ...newFilters };
  page.value = 1;
  fetchAnswers();
};

const formatScope = (scope) => {
  switch (scope) {
    case 'organization':
      return 'Organización';
    case 'business':
      return 'Empresa';
    case 'business_unit':
      return 'Ubicación';
    case 'business_unit_group':
      return 'Grupo';
    default:
      return scope;
  }
};
const formatFrequency = (freq) => {
  switch (freq) {
    case 'once_per_day':
      return 'Una vez por día';
    case 'multiple_per_day':
      return 'Múltiples veces por día';
    default:
      return freq;
  }
};
const mapRoleName = (name) => {
  if (!name) return '';
  if (name === 'superadmin') return 'Super Administrador';
  if (name === 'admin') return 'Administrador';
  if (name === 'sponsor') return 'Sponsor';
  return name;
};

const activeTab = ref('table');

const downloadingExcel = ref(false);
const downloadExcel = async () => {
  if (!resolvedFormId.value || downloadingExcel.value) return;
  downloadingExcel.value = true;
  try {
    const params = {
      search: filters.value.search,
      report_status: filters.value.report_status,
      user_id: filters.value.user_id,
      folio: filters.value.folio,
      score_min: filters.value.score_min,
      score_max: filters.value.score_max,
      start_date: filters.value.start_date,
      end_date: filters.value.end_date
    };
    Object.keys(params).forEach((key) => {
      if (params[key] === null || params[key] === '' || params[key] === undefined) {
        delete params[key];
      }
    });
    const query = new URLSearchParams(params).toString();
    let url = `/forms/${resolvedFormId.value}/reports/export-excel`;
    if (query) url += `?${query}`;

    const res = await axios.get(url, { responseType: 'blob' });
    const blob = new Blob([res.data], {
      type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
    });
    const a = document.createElement('a');
    a.href = window.URL.createObjectURL(blob);
    a.download = `reporte_formulario_${resolvedFormId.value}.xlsx`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    window.URL.revokeObjectURL(a.href);
  } catch (e) {
    alert('No se pudo descargar el Excel.');
  } finally {
    downloadingExcel.value = false;
  }
};

// NUEVO: función para cambiar de página usando la paginación del backend
const handlePageChange = async (newPage) => {
  if (page.value !== newPage) {
    page.value = newPage;
    await fetchAnswers();
  }
};
</script>

<template>
  <v-container fluid>
    <v-row class="align-center mb-6" no-gutters>
      <v-col cols="auto" class="d-flex align-center">
        <v-btn
          icon
          variant="text"
          class="px-3 py-2"
          style="border-radius: 8px; border: 1px solid #ccc; min-width: 44px; height: 44px"
          @click="goToIndex"
        >
          <v-icon :icon="mdiArrowLeft" />
        </v-btn>
        <!-- Desktop: muestra "Reportes - ..." / Mobile: muestra "R-Nombre" -->
        <h3 class="font-weight-medium ml-3 mb-0 d-none d-md-block" v-if="formData">
          Reportes - {{ formData.name || formData.form?.name || 'Formulario' }}
        </h3>
        <h3 class="font-weight-medium ml-3 mb-0 d-block d-md-none" v-if="formData">
          R-{{ formData.name || formData.form?.name || 'Formulario' }}
        </h3>
      </v-col>
      <v-spacer />
      <v-col cols="auto" class="d-flex align-center">
        <!-- Dropdown de opciones -->
        <v-menu location="bottom end">
          <template #activator="{ props }">
            <v-btn
              v-bind="props"
              variant="text"
              class="px-3 py-2"
              style="border-radius: 8px; border: 1px solid #ccc; min-width: 44px; height: 44px"
            >
              Opciones
              <v-icon :icon="mdiChevronDown" end size="18" />
            </v-btn>
          </template>
          <v-list class="custom-dropdown elevation-1 rounded-lg" style="min-width: 200px">
            <v-list-item :loading="downloadingExcel" @click="downloadExcel">
              <template #prepend>
                <v-icon :icon="mdiMicrosoftExcel" size="18" />
              </template>
              <v-list-item-title>Descargar Excel</v-list-item-title>
            </v-list-item>
          </v-list>
        </v-menu>
      </v-col>
    </v-row>

    <!-- Información general del formulario -->
    <v-row>
      <v-col cols="12" md="4" class="d-flex justify-center align-center">
        <template v-if="formData?.logo">
          <v-img :src="formData.logo" max-width="320" max-height="320" class="rounded-lg logo-avatar" alt="Logo" style="background: none" />
        </template>
        <template v-else>
          <div
            class="d-flex align-center justify-center logo-avatar"
            style="width: 320px; height: 320px; background-color: #f5f5f5; border-radius: 12px"
          >
            <span class="text-medium-emphasis">Sin logo</span>
          </div>
        </template>
      </v-col>
      <v-col cols="12" md="8">
        <v-card-title class="font-weight-bold text-h6 pb-2" style="padding-left: 0.5rem">Información general</v-card-title>
        <v-table class="rounded-lg elevation-1 card-shadow">
          <tbody>
            <tr>
              <td class="font-weight-bold text-subtitle-1" style="width: 1%; white-space: nowrap">Folio</td>
              <td>
                <span v-if="formData?.folio">{{ formData.folio }}</span>
                <span v-else>No disponible</span>
              </td>
            </tr>
            <tr>
              <td class="font-weight-bold text-subtitle-1" style="width: 1%; white-space: nowrap">Nombre</td>
              <td>{{ formData?.name || 'No disponible' }}</td>
            </tr>
            <tr>
              <td class="font-weight-bold text-subtitle-1" style="width: 1%; white-space: nowrap">Descripción</td>
              <td>{{ formData?.description || 'No disponible' }}</td>
            </tr>
            <tr>
              <td class="font-weight-bold text-subtitle-1" style="width: 1%; white-space: nowrap">Alcance</td>
              <td>
                <template v-if="formData?.assignment_scope === 'organization' && formData?.organization">
                  <a :href="`/organizaciones/${formData.organization.id}`" style="color: #1976d2; text-decoration: underline">
                    {{ formData.organization.folio }} -
                    {{ formData.organization.legal_name || formData.organization.name }}
                  </a>
                </template>
                <template v-else-if="formData?.assignment_scope === 'business' && formData?.business">
                  <a :href="`/negocios/${formData.business.id}`" style="color: #1976d2; text-decoration: underline">
                    {{ formData.business.folio }} - {{ formData.business.name }}
                  </a>
                </template>
                <template v-else-if="formData?.assignment_scope === 'business_unit' && formData?.business_unit">
                  <a :href="`/ubicaciones/${formData.business_unit.id}`" style="color: #1976d2; text-decoration: underline">
                    {{ formData.business_unit.folio }} - {{ formData.business_unit.name }}
                  </a>
                </template>
                <template v-else-if="formData?.assignment_scope === 'business_unit_group' && formData?.business_unit_group">
                  <a :href="`/grupos/${formData.business_unit_group.id}`" style="color: #1976d2; text-decoration: underline">
                    {{ formData.business_unit_group.id }} -
                    {{ formData.business_unit_group.name }}
                  </a>
                </template>
                <template v-else>
                  {{ formatScope(formData?.assignment_scope) }}
                </template>
              </td>
            </tr>
            <tr>
              <td class="font-weight-bold text-subtitle-1" style="width: 1%; white-space: nowrap">Frecuencia</td>
              <td>{{ formatFrequency(formData?.frequency) }}</td>
            </tr>
            <tr>
              <td class="font-weight-bold text-subtitle-1" style="width: 1%; white-space: nowrap">¿Tiene ponderación?</td>
              <td>{{ formData?.has_rating ? 'Sí' : 'No' }}</td>
            </tr>
            <tr>
              <td class="font-weight-bold text-subtitle-1" style="width: 1%; white-space: nowrap">Creado por</td>
              <td>
                <span v-if="formData?.created_by">{{ formData.created_by.name || formData.created_by.email }}</span>
                <span v-else>No disponible</span>
              </td>
            </tr>
          </tbody>
        </v-table>
      </v-col>
    </v-row>

    <!-- Filtros entre información general y tabs -->
    <div style="padding-top: 32px">
      <RFormAnswerFilters
        :key="formData?.id"
        :hasRating="hasRating"
        :activeFilters="filters.value"
        @search="handleSearch"
        @filter="handleFilter"
      />
    </div>

    <!-- Tabs para tabla/cards y gráficas -->
    <v-tabs v-model="activeTab" class="mt-6 mb-4">
      <v-tab value="table">Respuestas</v-tab>
      <v-tab value="charts">Gráficas</v-tab>
    </v-tabs>

    <v-window v-model="activeTab">
      <v-window-item value="table">
        <RFormAnswerTableCards
          :items="sortedItems"
          :sortedItems="sortedItems"
          :page="pagination.current_page"
          :itemsPerPage="pagination.per_page"
          :sortBy="sortBy"
          :sortDesc="sortDesc"
          :hasRating="hasRating"
          :loading="loading"
          :organizationId="formData?.organization?.id"
          :formId="Number(resolvedFormId) || null"
          @update:page="handlePageChange"
          @sort="toggleSort"
          @view="viewAnswer"
          @closeReport="closeReport"
          @downloadPdf="downloadReportPdf"
        />
        <div class="d-flex justify-center mt-4">
          <v-pagination
            v-if="pagination.last_page > 1"
            :length="pagination.last_page"
            :model-value="pagination.current_page"
            @update:model-value="handlePageChange"
            :total-visible="7"
            color="primary"
          />
        </div>
      </v-window-item>
      <v-window-item value="charts">
        <RFormAnswerCharts :items="items" :hasRating="hasRating" :loading="loading" :form-id="resolvedFormId" :filters="filters" />
      </v-window-item>
    </v-window>
  </v-container>
</template>

<style scoped src="@/styles/report_answer_list.css"></style>
