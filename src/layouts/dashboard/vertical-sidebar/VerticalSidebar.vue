<script setup lang="ts">
import { computed, ref, onMounted } from 'vue';
import { useCustomizerStore } from '../../../stores/customizer';
import sidebarItems from './sidebarItem';
import { useAuthStore } from '@/stores/auth';
import axiosInstance from '@/utils/axios';

import NavGroup from './NavGroup/NavGroup.vue';
import NavItem from './NavItem/NavItem.vue';
import NavCollapse from './NavCollapse/NavCollapse.vue';

const customizer = useCustomizerStore();
const auth = useAuthStore();

const isHovered = ref(false);
const hasDashboardAccess = ref<boolean | null>(null);

const userRoles = computed(() => auth.user?.roles || []);
const permissions = computed(() => auth.user?.permissions || []);
const hasOrgViewAny = computed(() => permissions.value.includes('organization.viewAny'));
const hasBusinessViewAny = computed(() => permissions.value.includes('business.viewAny'));
const hasBusinessUnitViewAny = computed(() => permissions.value.includes('businessUnit.viewAny'));

const hasFormViewAny = computed(() => permissions.value.includes('form.create'));
const hasFormView = computed(() => permissions.value.includes('form.view'));
const hasResponseViewAny = computed(() => permissions.value.includes('form_response.viewAny'));
const hasBusinessUnitGroupViewAny = computed(() => permissions.value.includes('businessUnitGroup.viewAny'));
const hasUserViewAny = computed(() => permissions.value.includes('user.viewAny'));
const hasRoleViewAny = computed(() => permissions.value.includes('role.viewAny'));
const hasReportViewAny = computed(() => permissions.value.includes('report.viewAny'));

const shouldShowLogo = computed(() => {
  if (!customizer.mini_sidebar) return true;
  if (customizer.mini_sidebar && isHovered.value) return true;
  return false;
});

function getOrgRoute() {
  return '/organizaciones';
}
function getBusinessRoute() {
  return '/empresas';
}
function getBusinessUnitRoute() {
  return '/ubicaciones';
}
function getBusinessUnitGroupRoute() {
  return '/grupos-de-ubicaciones';
}
function getUserRoute() {
  return '/usuarios';
}
function getRolesRoute() {
  return '/roles';
}

const checkDashboardAccess = async () => {
  const user = auth.user;
  if (!user) {
    hasDashboardAccess.value = false;
    return;
  }

  if (user.roles?.includes('admin') || user.roles?.includes('superadmin')) {
    hasDashboardAccess.value = true;
    return;
  }

  try {
    const { data } = await axiosInstance.get('/my-forms', {
      params: { page: 1, per_page: 1 }
    });

    const hasForms = (data.data?.length || 0) > 0;
    hasDashboardAccess.value = hasForms;
  } catch (error) {
    console.error('Error checking dashboard access:', error);
    hasDashboardAccess.value = false;
  }
};

function canViewReports(user: any): boolean {
  if (!user) return false;

  // Superadmin
  if (user.roles?.includes('superadmin')) return true;

  // Admin: debe tener organización asignada
  if (user.roles?.includes('admin')) {
    return !!user.organization_id;
  }

  if (user.permissions?.includes('report.update')) {
    return true;
  }

  // Sponsor: debe tener empresa asignada
  if (user.roles?.includes('sponsor')) {
    return !!user.business_id;
  }

  // Auditor o supervisor: debe tener formularios vinculados
  if (user.roles?.some((r: string) => ['auditor', 'supervisor'].includes(r))) {
    // Asume que has_linked_forms se obtiene y guarda en el usuario
    return !!user.has_linked_forms;
  }

  // Permiso explícito
  if (user.permissions?.includes('report.view')) {
    return !!user.organization_id;
  }

  return false;
}

// Sidebar SOLO muestra los apartados si el usuario es superadmin o tiene el permiso viewAny
const sidebarMenu = computed(() => {
  return sidebarItems
    .map((item) => {
      // Filtrar "Inicio" (Dashboard) basado en acceso
      if (item.title === 'Inicio') {
        // Si aún no se ha verificado, no mostrar (evita flash)
        if (hasDashboardAccess.value === null) {
          return null;
        }
        // Solo mostrar si tiene acceso
        return hasDashboardAccess.value ? item : null;
      }

      if (item.title === 'Organizaciones') {
        const show = userRoles.value.includes('superadmin') || hasOrgViewAny.value;
        return show ? { ...item, to: getOrgRoute() } : null;
      }
      if (item.title === 'Empresas') {
        const show = userRoles.value.includes('superadmin') || hasBusinessViewAny.value;
        return show ? { ...item, to: getBusinessRoute() } : null;
      }
      if (item.title === 'Ubicaciones') {
        // Dropdown: solo muestra hijos que el usuario puede ver
        const children = (item.children || [])
          .map((child) => {
            if (child.title === 'Ubicaciones') {
              const show = userRoles.value.includes('superadmin') || hasBusinessUnitViewAny.value;
              return show ? { ...child, to: getBusinessUnitRoute() } : null;
            }
            if (child.title === 'Grupos de Ubicación') {
              const show = userRoles.value.includes('superadmin') || hasBusinessUnitGroupViewAny.value;
              return show ? { ...child, to: getBusinessUnitGroupRoute() } : null;
            }
            return child;
          })
          .filter(Boolean);
        // Solo muestra el dropdown si hay al menos un hijo visible
        return children.length > 0 ? { ...item, children } : null;
      }
      if (item.title === 'Usuarios') {
        // Dropdown Usuarios con Usuarios y Roles & Permisos
        const children = (item.children || [])
          .map((child) => {
            if (child.title === 'Usuarios') {
              const show = userRoles.value.includes('superadmin') || hasUserViewAny.value;
              return show ? { ...child, to: getUserRoute() } : null;
            }
            if (child.title === 'Roles & Permisos') {
              // SOLO si tiene el permiso role.viewAny o es superadmin
              const show = userRoles.value.includes('superadmin') || hasRoleViewAny.value;
              return show ? { ...child, to: getRolesRoute() } : null;
            }
            return child;
          })
          .filter(Boolean);
        return children.length > 0 ? { ...item, children } : null;
      }
      if (item.title === 'Usuarios') {
        const show = userRoles.value.includes('superadmin') || hasUserViewAny.value;
        return show ? { ...item, to: '/usuarios' } : null;
      }
      if (item.title === 'Formularios') {
        const show = userRoles.value.includes('superadmin') || hasFormViewAny.value;
        return show ? { ...item, to: '/formularios' } : null;
      }
      if (item.title === 'Reportes') {
        const show = canViewReports(auth.user);
        return show ? { ...item, to: '/reportes' } : null;
      }
      return item;
    })
    .filter(Boolean);
});

onMounted(async () => {
  await checkDashboardAccess();
});
</script>

<template>
  <v-navigation-drawer
    v-if="auth.user"
    left
    v-model="customizer.Sidebar_drawer"
    elevation="0"
    rail-width="60"
    mobile-breakpoint="lg"
    app
    class="leftSidebar"
    :rail="customizer.mini_sidebar"
    expand-on-hover
    @mouseenter="isHovered = true"
    @mouseleave="isHovered = false"
  >
    <div class="pa-5 text-center">
      <v-img v-if="shouldShowLogo" :src="'/Logotipo1.svg'" alt="Tasker Logo" max-height="40" contain class="mx-auto" />
      <v-img v-else :src="'/favicon.svg'" alt="Tasker Logo" max-height="40" contain class="mx-auto" />
    </div>
    <perfect-scrollbar class="scrollnavbar">
      <v-list aria-busy="true" aria-label="menu list">
        <template v-for="(item, i) in sidebarMenu.filter((item) => item !== null)" :key="i">
          <NavGroup :item="item" v-if="item.header" :key="item.title" />
          <v-divider class="my-1" v-else-if="item.divider" />
          <NavCollapse class="leftPadding" :item="item" :level="0" v-else-if="item.children" />
          <NavItem :item="item" v-else />
        </template>
      </v-list>
    </perfect-scrollbar>
  </v-navigation-drawer>
</template>
