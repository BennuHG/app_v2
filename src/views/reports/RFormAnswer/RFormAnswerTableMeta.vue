<script setup>
const props = defineProps({
  items: {
    type: Array,
    default: () => []
  },
  page: Number,
  itemsPerPage: Number,
  sortBy: String,
  sortDesc: Boolean,
  hasRating: {
    type: Boolean,
    default: false
  },
  organizationId: Number, // Nueva propiedad para identificar la organización
  formId: Number // Nueva propiedad para identificar el formulario
});
const emit = defineEmits(['update:page', 'sort']);
</script>

<template>
  <div>
    <v-table density="comfortable" class="fixed-table elevation-1 rounded-lg">
      <thead>
        <tr>
          <th @click="emit('sort', 'folio')" class="cursor-pointer folio-header">
            Folio
            <slot name="sort-icon" :column="'folio'" />
          </th>
          <th
            v-if="Number(organizationId) === 3 && Number(formId) === 5"
            @click="emit('sort', 'direccion')"
            class="cursor-pointer direccion-header"
          >
            Dirección
            <slot name="sort-icon" :column="'direccion'" />
          </th>
          <th v-else @click="emit('sort', 'name')" class="cursor-pointer name-header">
            Usuario
            <slot name="sort-icon" :column="'name'" />
          </th>
          <!-- Nueva columna CIAC: Solo para organización 3 y formulario 5 -->
          <th v-if="Number(organizationId) === 3 && Number(formId) === 5" @click="emit('sort', 'ciac')" class="cursor-pointer ciac-header">
            CIAC / Origen
            <slot name="sort-icon" :column="'ciac'" />
          </th>
          <th @click="emit('sort', 'answer_date')" class="cursor-pointer answer-date-header">
            Fecha de Respuesta
            <slot name="sort-icon" :column="'answer_date'" />
          </th>
          <th @click="emit('sort', 'status')" class="cursor-pointer status-header">
            Estado
            <slot name="sort-icon" :column="'status'" />
          </th>

          <!-- Header de Puntaje SOLO si hay rating -->
          <th v-if="props.hasRating" @click="emit('sort', 'score')" class="cursor-pointer score-header">
            Puntaje
            <slot name="sort-icon" :column="'score'" />
          </th>

          <th class="actions-header" style="width: 60px"></th>
        </tr>
      </thead>

      <tbody>
        <slot name="rows" :hasRating="props.hasRating" />
      </tbody>
    </v-table>
  </div>
</template>

<style scoped src="@/styles/rform_answer.css"></style>
