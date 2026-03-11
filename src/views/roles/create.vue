<script setup>
import { ref, computed, watch, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { mdiArrowLeft, mdiInformationSlabCircleOutline } from '@mdi/js';
import axios from '@/utils/axios';
import { useAuthStore } from '@/stores/auth';
import { permissionTranslations } from '@/utils/permissionTranslations';
import { permissionDescriptions } from '@/utils/permissionsDescriptions';
import PermissionFilter from './permissionfilter.vue';

const auth = useAuthStore();
const router = useRouter();

function isSuperadmin() {
  return auth.user?.roles?.includes('superadmin');
}

const isMobile = ref(window.innerWidth < 600);
window.addEventListener('resize', () => {
  isMobile.value = window.innerWidth < 600;
});

const name = ref('');
const organizationId = ref(isSuperadmin() ? null : (auth.user?.organization_id ?? null));
const organizations = ref([]);
const orgLoading = ref(false);
const orgSearch = ref('');
const allPermissions = ref([]);
const filteredPermissions = ref([]);

// --- INTEGRACIÓN: permisos por default ---
const defaultPermissions = ['organization.view', 'business.view', 'businessUnit.view', 'report.update', 'report.view'];
const selectedPermissions = ref([...defaultPermissions]);
// --- FIN INTEGRACIÓN ---

const loading = ref(false);
const saving = ref(false);
const search = ref('');
const page = ref(1);
const itemsPerPage = ref(10);

const modelNames = {
  user: 'Usuario',
  role: 'Rol',
  permission: 'Permiso',
  businessUnit: 'Ubicación',
  businessUnitGroup: 'Grupo de Ubicaciones',
  organization: 'Organización',
  business: 'Negocio',
  form: 'Formulario'
};

const canCreate = computed(() => auth.user?.permissions?.includes('role.create'));

function groupPermissions(perms) {
  return perms
    .filter(
      (perm) =>
        perm.name && !perm.name.startsWith('form_field.') && !perm.name.startsWith('permission.') && !perm.name.startsWith('person.')
    )
    .map((perm) => {
      let model = perm.name.split('.')[0] || perm.model || perm.model_name || 'Otro';
      if (model === 'form_response') model = 'form';
      return {
        id: perm.id,
        name: perm.name,
        model,
        modelLabel: modelNames[model] || model,
        label: permissionTranslations[perm.name] || permissionTranslations[perm.name?.split('.')[1]] || perm.name
      };
    });
}

// Ordena los permisos: seleccionados primero (en el orden de selectedPermissions), luego el resto
const permissionRows = computed(() => {
  let perms = groupPermissions(allPermissions.value);
  if (search.value) {
    const s = search.value.toLowerCase();
    perms = perms.filter(
      (p) => p.modelLabel.toLowerCase().includes(s) || p.label.toLowerCase().includes(s) || p.name.toLowerCase().includes(s)
    );
  }
  filteredPermissions.value = perms;
  // Ordena: seleccionados primero (en el orden de selectedPermissions), luego el resto
  const selected = [];
  const unselected = [];
  for (const perm of perms) {
    if (selectedPermissions.value.includes(perm.name)) {
      selected.push(perm);
    } else {
      unselected.push(perm);
    }
  }
  // Ordena los seleccionados según el orden en selectedPermissions
  selected.sort((a, b) => selectedPermissions.value.indexOf(a.name) - selectedPermissions.value.indexOf(b.name));
  const ordered = [...selected, ...unselected];
  const start = (page.value - 1) * itemsPerPage.value;
  return ordered.slice(start, start + itemsPerPage.value);
});

const totalPages = computed(() => Math.ceil(filteredPermissions.value.length / itemsPerPage.value));

onMounted(async () => {
  if (!canCreate.value) return;
  loading.value = true;
  try {
    const { data } = await axios.get('/permissions');
    allPermissions.value = data.data || data;
    // --- INTEGRACIÓN: solo selecciona los permisos default que existan en la lista ---
    selectedPermissions.value = defaultPermissions.filter((p) => (allPermissions.value || []).some((perm) => perm.name === p));
    // --- FIN INTEGRACIÓN ---
    if (isSuperadmin()) {
      orgLoading.value = true;
      const orgRes = await axios.get('/organizations', {
        params: { search: orgSearch.value }
      });
      organizations.value = orgRes.data.data || orgRes.data;
      orgLoading.value = false;
    }
  } catch (e) {
    allPermissions.value = [];
    organizations.value = [];
  } finally {
    loading.value = false;
  }
});

watch(search, () => {
  page.value = 1;
});

watch(orgSearch, async (val) => {
  if (isSuperadmin()) {
    orgLoading.value = true;
    try {
      const orgRes = await axios.get('/organizations', {
        params: { search: val }
      });
      organizations.value = orgRes.data.data || orgRes.data;
    } catch (e) {
      organizations.value = [];
    } finally {
      orgLoading.value = false;
    }
  }
});

watch([itemsPerPage, page], () => {
  permissionRows.value;
});

function togglePermission(name) {
  // No permitir quitar los permisos por default
  if (defaultPermissions.includes(name)) return;
  const idx = selectedPermissions.value.indexOf(name);
  if (idx !== -1) {
    selectedPermissions.value = selectedPermissions.value.filter((n) => n !== name);
  } else {
    // Agrega al inicio para que aparezca arriba
    selectedPermissions.value = [name, ...selectedPermissions.value];
  }
}

function handlePageChange(val) {
  page.value = val;
}

async function saveRole() {
  if (!canCreate.value || !name.value || !organizationId.value || selectedPermissions.value.length === 0) return;
  saving.value = true;
  try {
    await axios.post('/roles', {
      name: name.value,
      organization_id: organizationId.value,
      permissions: selectedPermissions.value
    });
    router.push('/roles');
  } catch (e) {
  } finally {
    saving.value = false;
  }
}
</script>

<template>
  <div v-if="canCreate">
    <v-container fluid>
      <!-- Header -->
      <v-row class="align-center mb-6" no-gutters>
        <v-col cols="auto" class="d-flex align-center">
          <v-btn icon variant="text" class="px-3 py-2" style="border-radius: 8px; border: 1px solid #ccc" @click="router.back()">
            <v-icon :icon="mdiArrowLeft" />
          </v-btn>
          <h3 class="font-weight-medium ml-3 mb-0">Agregar Rol</h3>
        </v-col>
      </v-row>

      <v-form class="mb-10">
        <!-- Información General -->
        <v-row>
          <v-col cols="12">
            <h4 class="font-weight-bold mb-3">Información General</h4>
            <v-divider class="mb-6" />
          </v-col>
        </v-row>

        <!-- Nombre del rol (full width) -->
        <v-row>
          <v-col cols="12">
            <v-label>Nombre<span class="text-error">*</span></v-label>
            <v-text-field v-model="name" variant="outlined" color="primary" required class="mt-2 mb-4" />
          </v-col>
        </v-row>
        <!-- Organización (full width, solo superadmin) -->
        <v-row v-if="isSuperadmin()">
          <v-col cols="12">
            <v-label>Organización <span class="text-error">*</span></v-label>
            <v-select
              v-model="organizationId"
              :items="organizations"
              :item-title="(org) => `${org.folio ?? org.id} - ${org.legal_name ?? org.name}`"
              item-value="id"
              variant="outlined"
              color="primary"
              required
              class="mt-2 mb-4"
              :loading="orgLoading"
              :searchable="true"
              clearable
            />
          </v-col>
        </v-row>

        <v-row>
          <v-col cols="12">
            <v-divider class="mb-6" />
          </v-col>
        </v-row>

        <!-- Filtro de permisos (full width) -->
        <v-row>
          <v-col cols="12">
            <PermissionFilter v-model="search" />
          </v-col>
        </v-row>

        <!-- Permisos como tabla en desktop -->
        <v-row v-if="!isMobile">
          <v-col>
            <v-table class="rounded-lg elevation-1 uniform-table" style="width: 100%">
              <thead>
                <tr>
                  <th class="font-weight-bold text-center uniform-col" style="width: 80px"></th>
                  <th class="font-weight-bold text-center uniform-col" style="width: 200px">Modelo</th>
                  <th class="font-weight-bold text-center uniform-col" style="width: 220px">Permiso</th>
                  <th class="font-weight-bold text-center uniform-col" style="width: 60px"></th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="perm in permissionRows"
                  :key="perm.id"
                  class="row-clickable"
                  :class="{ 'selected-row': selectedPermissions.includes(perm.name) }"
                  :style="{ cursor: defaultPermissions.includes(perm.name) ? 'not-allowed' : 'pointer' }"
                  @click="togglePermission(perm.name)"
                >
                  <td class="text-center uniform-col" style="padding-left: 0; padding-right: 8px">
                    <div class="d-flex justify-center align-center" style="height: 100%">
                      <v-checkbox
                        :model-value="selectedPermissions.includes(perm.name)"
                        @update:model-value="togglePermission(perm.name)"
                        color="primary"
                        hide-details
                        :ripple="false"
                        class="pa-0 ma-0"
                        :disabled="defaultPermissions.includes(perm.name)"
                        style="--v-checkbox-bg: #1976d2; --v-checkbox-checked-color: #fff"
                      />
                    </div>
                  </td>
                  <td class="text-center uniform-col" style="vertical-align: middle">
                    {{ perm.modelLabel }}
                  </td>
                  <td class="text-center uniform-col" style="vertical-align: middle">
                    <v-chip color="primary" text-color="white" class="ma-0 pa-2" style="font-weight: 500; font-size: 0.95em">
                      {{ perm.label }}
                    </v-chip>
                  </td>
                  <td class="text-center uniform-col" style="vertical-align: middle">
                    <v-tooltip location="top" :text="permissionDescriptions[perm.name] || 'Sin descripción'">
                      <template #activator="{ props }">
                        <v-icon v-bind="props" :icon="mdiInformationSlabCircleOutline" color="primary" style="cursor: pointer" />
                      </template>
                    </v-tooltip>
                  </td>
                </tr>
                <tr v-if="permissionRows.length === 0">
                  <td colspan="4" class="text-center text-medium-emphasis">No hay permisos disponibles.</td>
                </tr>
              </tbody>
            </v-table>
          </v-col>
        </v-row>
        <!-- Permisos como cards en mobile, estilo tabla -->
        <v-row v-else>
          <v-col cols="12" v-for="perm in permissionRows" :key="perm.id">
            <v-card
              class="permission-card"
              :class="{ 'selected-row': selectedPermissions.includes(perm.name) }"
              @click="togglePermission(perm.name)"
              elevation="1"
              :style="{
                cursor: defaultPermissions.includes(perm.name) ? 'not-allowed' : 'pointer',
                marginBottom: '20px',
                padding: '20px 16px'
              }"
            >
              <v-row no-gutters>
                <!-- Columna izquierda: checkbox centrado -->
                <v-col cols="auto" class="d-flex align-center justify-center" style="width: 64px">
                  <v-checkbox
                    :model-value="selectedPermissions.includes(perm.name)"
                    @update:model-value="togglePermission(perm.name)"
                    color="primary"
                    hide-details
                    :ripple="false"
                    class="pa-0 ma-0"
                    :disabled="defaultPermissions.includes(perm.name)"
                    style="--v-checkbox-bg: #1976d2; --v-checkbox-checked-color: #fff"
                  />
                </v-col>
                <!-- Columna derecha: datos alineados verticalmente -->
                <v-col class="d-flex flex-column justify-center" style="padding-left: 16px">
                  <div>
                    <span class="font-weight-medium" style="min-width: 100px; font-size: 0.85em">{{ perm.modelLabel }}</span>
                  </div>
                  <div>
                    <v-chip
                      color="primary"
                      text-color="white"
                      class="ma-0 pa-1"
                      style="
                        font-weight: 500;
                        font-size: 0.75em;
                        min-width: 32px;
                        padding-left: 3px;
                        padding-right: 3px;
                        margin-top: 2px;
                        margin-left: 0;
                      "
                    >
                      {{ perm.label }}
                    </v-chip>
                  </div>
                  <div class="text-caption text-medium-emphasis" style="padding-top: 4px">
                    {{ permissionDescriptions[perm.name] || 'Sin descripción' }}
                  </div>
                </v-col>
              </v-row>
            </v-card>
          </v-col>
          <v-col cols="12" v-if="permissionRows.length === 0">
            <v-alert type="info" variant="outlined">No hay permisos disponibles.</v-alert>
          </v-col>
        </v-row>
        <!-- Paginación centrada -->
        <v-row v-if="totalPages > 1">
          <v-col class="d-flex justify-center">
            <v-pagination v-model="page" :length="totalPages" @update:model-value="handlePageChange" color="primary" />
          </v-col>
        </v-row>
        <v-row>
          <v-col cols="12" class="d-flex justify-end">
            <v-btn
              color="primary"
              :loading="saving"
              :disabled="!canCreate || !name || !organizationId || selectedPermissions.length === 0"
              @click="saveRole"
              class="mt-6"
            >
              <template v-slot:loader>
                <v-progress-circular indeterminate color="white" size="20" />
              </template>
              {{ saving ? 'Guardando...' : 'Guardar Rol' }}
            </v-btn>
          </v-col>
        </v-row>
      </v-form>
    </v-container>
  </div>
</template>

<style scoped src="@/styles/roles.css"></style>
