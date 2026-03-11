<template>
  <div class="address-fields">
    <!-- Mensaje si no hay permiso de ubicación -->
    <div v-if="isCreate && !geoPermissionGranted" class="text-error" style="margin-bottom: 12px">
      Debes permitir el acceso a tu ubicación en el navegador para usar esta función.
    </div>

    <!-- Calle con botón de búsqueda Google Maps -->
    <div class="field-group single" style="position: relative">
      <v-text-field
        ref="autocompleteInput"
        v-model="fields.street"
        label="Calle"
        variant="outlined"
        color="primary"
        :placeholder="placeholder || 'Ingresa una calle'"
        hide-details
        density="compact"
        @focus="initAutocomplete"
        @keydown.enter="onFieldsEnter"
        :error-messages="addressError"
      />
    </div>

    <!-- Número exterior e interior -->
    <div class="field-group">
      <v-text-field
        v-model="fields.outdoor_number"
        label="Número exterior"
        variant="outlined"
        color="primary"
        hide-details
        density="compact"
        @keydown.enter="onFieldsEnter"
        :error-messages="addressError"
      />
      <v-text-field
        v-model="fields.indoor_number"
        label="Número interior"
        variant="outlined"
        color="primary"
        hide-details
        density="compact"
        :error-messages="addressError"
      />
    </div>

    <!-- Colonia y Código Postal -->
    <div class="field-group">
      <v-text-field
        v-model="fields.neighborhood"
        label="Colonia"
        variant="outlined"
        color="primary"
        hide-details
        density="compact"
        @keydown.enter="onFieldsEnter"
        :error-messages="addressError"
      />
      <v-text-field
        v-model="fields.postal_code"
        label="Código Postal"
        variant="outlined"
        color="primary"
        hide-details
        density="compact"
        @keydown.enter="onFieldsEnter"
        :error-messages="addressError"
      />
    </div>

    <!-- Ciudad y Estado -->
    <div class="field-group">
      <v-text-field
        v-model="fields.city"
        label="Ciudad"
        variant="outlined"
        color="primary"
        hide-details
        density="compact"
        @keydown.enter="onFieldsEnter"
        :error-messages="addressError"
      />
      <v-text-field
        v-model="fields.state"
        label="Estado"
        variant="outlined"
        color="primary"
        hide-details
        density="compact"
        @keydown.enter="onFieldsEnter"
        :error-messages="addressError"
      />
    </div>

    <!-- País -->
    <div class="field-group single">
      <div style="display: flex; flex-direction: column">
        <v-autocomplete
          v-model="fields.country"
          :items="countries"
          label="País"
          variant="outlined"
          color="primary"
          hide-details
          density="compact"
          v-model:search-input="countrySearch"
          :item-title="'name'"
          :item-value="'name'"
          clearable
          :return-object="false"
          :filter="customCountryFilter"
          @keydown.enter="onFieldsEnter"
          :error-messages="addressError"
        />
        <div v-if="addressError" class="text-error" style="font-size: 0.78rem; margin-top: 6px; margin-bottom: 0">
          {{ addressError }}
        </div>
      </div>
    </div>

    <!-- Radio de geolocalización -->
    <div class="field-group single">
      <v-text-field
        v-model.number="fields.geofence_radius"
        label="Radio de geolocalización (metros)"
        type="number"
        min="0"
        variant="outlined"
        color="primary"
        hide-details
        density="compact"
      />
    </div>

    <!-- Mapa SIEMPRE visible -->
    <div ref="mapContainer" class="map-preview" />
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted, nextTick, computed } from 'vue';

const props = defineProps({
  placeholder: { type: String, default: '' },
  initialValue: { type: Object, default: () => ({}) },
  mode: { type: String, default: 'edit' }, // 'create' | 'edit'
  addressError: { type: String, default: '' },
  sessionLocation: { type: Object, default: () => ({}) }
});
const emit = defineEmits(['update:parsedAddress']);

const isCreate = computed(() => (props.mode || 'edit') === 'create');
const geoPermissionGranted = computed(() => localStorage.getItem('geo_permission_granted') === '1');

const autocompleteInput = ref(null);
const autocomplete = ref(null);
const mapContainer = ref(null);
const map = ref(null);
const marker = ref(null);
const circle = ref(null);
const geocoder = ref(null);

const isHydrating = ref(false);
const mapBooted = ref(false);
const markerBooted = ref(false);
const sessionHandled = ref(false);

//Flags para ubi del pin
const userMovedPin = ref(false);

const countrySearch = ref('');
const countries = [{ name: 'México' }, { name: 'Estados Unidos' }, { name: 'Canadá' }, { name: 'España' }, { name: 'Argentina' }];

const fields = ref({
  street: '',
  outdoor_number: '',
  indoor_number: '',
  neighborhood: '',
  postal_code: '',
  city: '',
  state: '',
  country: '',
  latitude: null,
  longitude: null,
  geofence_radius: 100
});

/* ============== Helpers =============== */
const normalize = (str = '') =>
  str
    .normalize('NFD')
    .replace(/\p{Diacritic}/gu, '')
    .toLowerCase();

const customCountryFilter = (item, queryText) => normalize(item.name).includes(normalize(queryText));

const hasCoords = (val) => val != null && val !== '' && !Number.isNaN(Number(val));

const extractSessionLatLng = () => {
  const s = props.sessionLocation || {};
  const lat = hasCoords(s.lat) ? Number(s.lat) : hasCoords(s.latitude) ? Number(s.latitude) : null;
  const lng = hasCoords(s.lng) ? Number(s.lng) : hasCoords(s.longitude) ? Number(s.longitude) : null;
  return { lat, lng };
};

const hasSufficientAddress = () => {
  const f = fields.value;
  const hasStreet = !!f.street;
  const hasCityState = !!(f.city || f.state);
  const hasCP = !!f.postal_code;
  const onlyCountry = !!f.country && !hasStreet && !hasCityState && !hasCP;
  return !onlyCountry && (hasStreet || hasCityState || hasCP);
};

const getFullAddressFromFields = () =>
  [
    fields.value.street,
    fields.value.outdoor_number,
    fields.value.neighborhood,
    fields.value.city,
    fields.value.state,
    fields.value.postal_code,
    fields.value.country
  ]
    .filter(Boolean)
    .join(', ');

/* ====== Google API ====== */
function ensureGeocoder() {
  if (!geocoder.value && window.google?.maps?.Geocoder) {
    geocoder.value = new google.maps.Geocoder();
  }
}

function geocodeAddress(address, onSuccess) {
  ensureGeocoder();
  if (!address?.trim()) return;
  geocoder.value.geocode({ address }, (results, status) => {
    if (status === 'OK' && results?.[0]) {
      const loc = results[0].geometry.location;
      onSuccess?.({ lat: loc.lat(), lng: loc.lng(), result: results[0] });
    }
  });
}

function reverseGeocode(lat, lng, onSuccess) {
  ensureGeocoder();
  geocoder.value.geocode({ location: { lat, lng } }, (results, status) => {
    if (status === 'OK' && results?.[0]) onSuccess?.(results[0]);
    else onSuccess?.(null);
  });
}

/* ====== Mapa / Marker / Circle ====== */
function bootMapVisual(center = { lat: 19.4326, lng: -99.1332 }) {
  if (mapBooted.value || !window.google?.maps) return;
  map.value = new google.maps.Map(mapContainer.value, {
    center,
    zoom: 16,
    clickableIcons: false,
    streetViewControl: false,
    mapTypeControl: false
  });
  mapBooted.value = true;

  map.value.addListener('click', (e) => {
    const lat = e.latLng.lat();
    const lng = e.latLng.lng();
    setMarkerPosition({ lat, lng }, true);
    fields.value.latitude = lat;
    fields.value.longitude = lng;
    userMovedPin.value = true;
    reverseGeocode(lat, lng, (place) => {
      if (place) fillFieldsFromPlace(place);
    });
    updateCircle();
    emit('update:parsedAddress', { ...fields.value });
  });
}

function ensureMarker() {
  if (markerBooted.value || !mapBooted.value) return;
  if (google.maps.marker?.AdvancedMarkerElement) {
    marker.value = new google.maps.marker.AdvancedMarkerElement({ map: map.value, position: map.value.getCenter() });
  } else {
    marker.value = new google.maps.Marker({ map: map.value, position: map.value.getCenter(), draggable: true });

    marker.value.addListener('dragend', (e) => {
      const dLat = e.latLng.lat();
      const dLng = e.latLng.lng();
      setMarkerPosition({ lat: dLat, lng: dLng }, false);
      fields.value.latitude = dLat;
      fields.value.longitude = dLng;
      userMovedPin.value = true;
      reverseGeocode(dLat, dLng, (place) => {
        if (place) fillFieldsFromPlace(place);
      });
      updateCircle();
      emit('update:parsedAddress', { ...fields.value });
    });
  }
  markerBooted.value = true;
  updateCircle();
}

function setMarkerPosition({ lat, lng }, center = true) {
  if (!mapBooted.value) {
    bootMapVisual({ lat, lng });
  }
  ensureMarker();

  const pos = { lat, lng };
  if (marker.value.setPosition) {
    marker.value.setPosition(pos);
  } else {
    marker.value.position = pos;
    marker.value.map = map.value;
  }
  if (center && map.value) map.value.setCenter(pos);
  updateCircle();
}

function updateCircle() {
  if (!mapBooted.value) return;
  const lat = Number(fields.value.latitude);
  const lng = Number(fields.value.longitude);
  if (Number.isNaN(lat) || Number.isNaN(lng)) return;

  const radius = Number(fields.value.geofence_radius) || 0;
  const center = { lat, lng };

  if (!circle.value) {
    circle.value = new google.maps.Circle({
      map: map.value,
      center,
      radius,
      fillColor: '#1976d2',
      fillOpacity: 0.15,
      strokeColor: '#1976d2',
      strokeOpacity: 0.5,
      strokeWeight: 2
    });
  } else {
    circle.value.setCenter(center);
    circle.value.setRadius(radius);
  }
}

/* ====== Places Autocomplete ====== */
let autocompleteInitialized = false;
function initAutocomplete() {
  if (autocompleteInitialized) return;
  const inputEl = autocompleteInput.value?.$el?.querySelector('input');
  if (!inputEl || !window.google?.maps?.places) return;
  autocompleteInitialized = true;

  autocomplete.value = new window.google.maps.places.Autocomplete(inputEl, { types: ['geocode'] });
  autocomplete.value.addListener('place_changed', () => {
    const place = autocomplete.value.getPlace();
    fillFieldsFromPlace(place);

    if (place?.geometry?.location) {
      const lat = place.geometry.location.lat();
      const lng = place.geometry.location.lng();
      fields.value.latitude = lat;
      fields.value.longitude = lng;
      userMovedPin.value = true;
      bootMapVisual({ lat, lng });
      setMarkerPosition({ lat, lng }, true);
      emit('update:parsedAddress', { ...fields.value });
    } else {
      const addr = place?.formatted_address || getFullAddressFromFields();
      if (addr) {
        geocodeAddress(addr, ({ lat, lng }) => {
          fields.value.latitude = lat;
          fields.value.longitude = lng;
          userMovedPin.value = true;
          bootMapVisual({ lat, lng });
          setMarkerPosition({ lat, lng }, true);
          emit('update:parsedAddress', { ...fields.value });
        });
      }
    }
  });
}
function focusAutocomplete() {
  nextTick(() => {
    const inputEl = autocompleteInput.value?.$el?.querySelector('input');
    if (inputEl) inputEl.focus();
    initAutocomplete();
  });
}

/* ====== Fill fields from place (solo textos) ====== */
function fillFieldsFromPlace(place) {
  const components = place?.address_components || [];
  const getComponent = (type) => components.find((c) => c.types.includes(type))?.long_name || '';

  fields.value.street = getComponent('route') || fields.value.street;
  fields.value.outdoor_number = getComponent('street_number') || fields.value.outdoor_number;
  fields.value.neighborhood = getComponent('sublocality') || getComponent('neighborhood') || fields.value.neighborhood;
  fields.value.postal_code = getComponent('postal_code') || fields.value.postal_code;
  fields.value.city = getComponent('locality') || getComponent('administrative_area_level_2') || fields.value.city;
  fields.value.state = getComponent('administrative_area_level_1') || fields.value.state;

  const countryLong = getComponent('country') || fields.value.country;
  if (countryLong) {
    const matched = countries.find((c) => normalize(c.name) === normalize(countryLong));
    fields.value.country = matched ? matched.name : countryLong;
  }

  emit('update:parsedAddress', { ...fields.value });
}

/* ====== Enter en fields ====== */
function onFieldsEnter() {
  if (isHydrating.value) return;
  const address = getFullAddressFromFields();
  if (!hasSufficientAddress() || !address.trim()) return;

  geocodeAddress(address, ({ lat, lng }) => {
    fields.value.latitude = lat;
    fields.value.longitude = lng;
    bootMapVisual({ lat, lng });
    setMarkerPosition({ lat, lng }, true);
    emit('update:parsedAddress', { ...fields.value });
  });
}

/* ====== Flujo CREATE con prioridades ====== */
function initCreateFlow() {
  const { lat: sLat, lng: sLng } = extractSessionLatLng();
  if (hasCoords(sLat) && hasCoords(sLng)) {
    fields.value.latitude = Number(sLat);
    fields.value.longitude = Number(sLng);
    bootMapVisual({ lat: fields.value.latitude, lng: fields.value.longitude });
    setMarkerPosition({ lat: fields.value.latitude, lng: fields.value.longitude }, true);

    // Obtener dirección y rellenar los campos
    reverseGeocode(fields.value.latitude, fields.value.longitude, (place) => {
      if (place) fillFieldsFromPlace(place);
    });

    sessionHandled.value = true;
    emit('update:parsedAddress', { ...fields.value });
    return;
  }

  const latCache = localStorage.getItem('geo_lat');
  const lngCache = localStorage.getItem('geo_lng');
  if (latCache && lngCache) {
    const lat = Number(latCache);
    const lng = Number(lngCache);
    fields.value.latitude = lat;
    fields.value.longitude = lng;
    bootMapVisual({ lat, lng });
    setMarkerPosition({ lat, lng }, true);

    // Obtener dirección y rellenar los campos
    reverseGeocode(lat, lng, (place) => {
      if (place) fillFieldsFromPlace(place);
    });

    emit('update:parsedAddress', { ...fields.value });
    return;
  }

  // Si no hay ubicación previa, inicializa el mapa en una ubicación predeterminada
  bootMapVisual({ lat: 19.4326, lng: -99.1332 });
}
/* ====== Watchers ====== */
watch(
  () => fields.value.geofence_radius,
  () => {
    updateCircle();
    emit('update:parsedAddress', { ...fields.value });
  }
);

watch(
  () => [fields.value.latitude, fields.value.longitude],
  ([lat, lng], [oldLat, oldLng]) => {
    if (isHydrating.value) return;
    if (hasCoords(lat) && hasCoords(lng)) {
      if (Number(lat) !== Number(oldLat) || Number(lng) !== Number(oldLng)) {
        if (!mapBooted.value) {
          bootMapVisual({ lat: Number(lat), lng: Number(lng) });
        }
        setMarkerPosition({ lat: Number(lat), lng: Number(lng) }, true);
        emit('update:parsedAddress', { ...fields.value });
      }
    }
  }
);

watch(
  () => props.sessionLocation,
  (loc) => {
    if (!isCreate.value) return;
    if (sessionHandled.value) return;
    const { lat, lng } = extractSessionLatLng();
    if (hasCoords(lat) && hasCoords(lng)) {
      fields.value.latitude = Number(lat);
      fields.value.longitude = Number(lng);
      bootMapVisual({ lat: fields.value.latitude, lng: fields.value.longitude });
      setMarkerPosition({ lat: fields.value.latitude, lng: fields.value.longitude }, true);
      sessionHandled.value = true;
      emit('update:parsedAddress', { ...fields.value });
    }
  },
  { immediate: true, deep: true }
);

watch(
  () => props.initialValue,
  async (val) => {
    if (isCreate.value) return;
    // Solo actualiza si el valor realmente cambió
    const keys = [
      'street',
      'outdoor_number',
      'indoor_number',
      'neighborhood',
      'postal_code',
      'city',
      'state',
      'country',
      'latitude',
      'longitude',
      'geofence_radius'
    ];
    const isDifferent = keys.some((k) => fields.value[k] !== val?.[k]);
    if (!isDifferent) return;

    isHydrating.value = true;

    Object.assign(fields.value, {
      street: '',
      outdoor_number: '',
      indoor_number: '',
      neighborhood: '',
      postal_code: '',
      city: '',
      state: '',
      country: '',
      latitude: null,
      longitude: null,
      geofence_radius: 100
    });

    if (val && typeof val === 'object') {
      Object.assign(fields.value, {
        street: val.street ?? '',
        outdoor_number: val.outdoor_number ?? '',
        indoor_number: val.indoor_number ?? '',
        neighborhood: val.neighborhood ?? '',
        postal_code: val.postal_code ?? '',
        city: val.city ?? '',
        state: val.state ?? '',
        country: val.country ?? '',
        geofence_radius: hasCoords(val.geofence_radius) ? Number(val.geofence_radius) : 100
      });

      await nextTick();

      const hasLatLng = hasCoords(val.latitude) && hasCoords(val.longitude);

      if (hasLatLng) {
        fields.value.latitude = Number(val.latitude);
        fields.value.longitude = Number(val.longitude);
        bootMapVisual({ lat: fields.value.latitude, lng: fields.value.longitude });
        setMarkerPosition({ lat: fields.value.latitude, lng: fields.value.longitude }, true);
        emit('update:parsedAddress', { ...fields.value });
      } else if (hasSufficientAddress()) {
        const addr = getFullAddressFromFields();
        geocodeAddress(addr, ({ lat, lng }) => {
          fields.value.latitude = lat;
          fields.value.longitude = lng;
          bootMapVisual({ lat, lng });
          setMarkerPosition({ lat, lng }, true);
          emit('update:parsedAddress', { ...fields.value });
        });
      } else {
        bootMapVisual({ lat: 19.4326, lng: -99.1332 });
        emit('update:parsedAddress', { ...fields.value });
      }
    } else {
      bootMapVisual({ lat: 19.4326, lng: -99.1332 });
      emit('update:parsedAddress', { ...fields.value });
    }

    isHydrating.value = false;
  },
  { immediate: true }
);

/* ====== Geolocation Watcher ====== */
let geoWatchId = null;

onMounted(() => {
  nextTick(() => {
    ensureGeocoder();

    if (isCreate.value) {
      const { lat: sLat, lng: sLng } = extractSessionLatLng();
      if (hasCoords(sLat) && hasCoords(sLng)) {
        fields.value.latitude = Number(sLat);
        fields.value.longitude = Number(sLng);
        bootMapVisual({ lat: fields.value.latitude, lng: fields.value.longitude });
        setMarkerPosition({ lat: fields.value.latitude, lng: fields.value.longitude }, true);
        sessionHandled.value = true;
        emit('update:parsedAddress', { ...fields.value });
      } else {
        initCreateFlow();
      }
    } else {
      bootMapVisual({ lat: 19.4326, lng: -99.1332 });
      emit('update:parsedAddress', { ...fields.value });
    }

    // Watcher SIEMPRE actualiza el marcador y mapa en tiempo real en modo create
    if (isCreate.value && geoPermissionGranted.value && navigator.geolocation) {
      geoWatchId = navigator.geolocation.watchPosition(
        (pos) => {
          if (userMovedPin.value) return;
          const lat = pos.coords.latitude;
          const lng = pos.coords.longitude;
          fields.value.latitude = lat;
          fields.value.longitude = lng;
          setMarkerPosition({ lat, lng }, true);
          reverseGeocode(lat, lng, (place) => {
            if (place) fillFieldsFromPlace(place);
            emit('update:parsedAddress', { ...fields.value });
          });
        },
        (err) => {},
        { enableHighAccuracy: true }
      );
    }
  });
});

onUnmounted(() => {
  // Detener watcher de geolocalización al salir de la pantalla
  if (geoWatchId !== null) {
    navigator.geolocation.clearWatch(geoWatchId);
    geoWatchId = null;
  }
});
</script>

<style scoped>
.address-fields {
  display: flex;
  flex-direction: column;
  gap: 20px;
}
.field-group {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 24px;
}
.field-group.single {
  grid-template-columns: 1fr;
}
.v-text-field,
.v-autocomplete {
  width: 100%;
}
.map-preview {
  height: 300px;
  width: 100%;
  border-radius: 8px;
  margin-top: 16px;
  box-shadow:
    0 1px 2px rgba(16, 24, 40, 0.06),
    0 1px 3px rgba(16, 24, 40, 0.1);
  border: 1px solid #e9ecef;
}
@media (max-width: 900px) {
  .field-group,
  .field-group.single {
    grid-template-columns: 1fr !important;
  }
}
</style>
