<template>
  <v-container fluid>
    <!-- Encabezado: flecha + folio - nombre -->
    <v-row class="align-center mb-6" no-gutters>
      <v-col cols="auto" class="d-flex align-center">
        <v-btn
          icon
          variant="text"
          class="px-3 py-2"
          style="border-radius: 8px; border: 1px solid #ccc; min-width: 44px; height: 44px"
          @click="goBack"
        >
          <v-icon :icon="mdiArrowLeft" />
        </v-btn>
        <h3 class="font-weight-medium ml-3 mb-0">{{ form?.folio }} - {{ form?.name || 'Formulario' }}</h3>
      </v-col>
    </v-row>

    <!-- Primera fila: logo + info general -->
    <v-row class="mb-6">
      <v-col cols="12" md="4" class="d-flex justify-center align-center">
        <template v-if="formLogo">
          <v-img :src="formLogo" max-width="320" max-height="320" class="rounded-lg" alt="Logo" style="background: none" />
        </template>
        <template v-else>
          <div
            class="d-flex align-center justify-center"
            style="width: 200px; height: 200px; background-color: #f5f5f5; border-radius: 12px"
          >
            <span class="text-medium-emphasis">Sin logo</span>
          </div>
        </template>
      </v-col>
      <v-col cols="12" md="8">
        <div class="font-weight-bold text-h6 mb-2" style="padding-left: 0.5rem">Información general</div>
        <v-card class="rounded-lg elevation-1 pa-0">
          <v-table class="rounded-lg elevation-0" style="border: none">
            <tbody>
              <tr>
                <td class="font-weight-bold text-subtitle-1">Folio</td>
                <td>
                  <span v-if="form?.folio">{{ form.folio }}</span>
                  <span v-else>No disponible</span>
                </td>
              </tr>
              <tr>
                <td class="font-weight-bold text-subtitle-1">Nombre del Formulario</td>
                <td>{{ form?.name || 'No disponible' }}</td>
              </tr>
              <tr>
                <td class="font-weight-bold text-subtitle-1">
                  <template v-if="form?.assignment_scope === 'organization'">Nombre Legal (Organización)</template>
                  <template v-else-if="form?.assignment_scope === 'business'">Nombre (Empresa)</template>
                  <template v-else-if="form?.assignment_scope === 'business_unit'">Nombre (Ubicación)</template>
                  <template v-else-if="form?.assignment_scope === 'business_unit_group'">Grupo de Unidades</template>
                  <template v-else>Nombre del Formulario</template>
                </td>
                <td>
                  <template v-if="form?.assignment_scope === 'organization'">
                    {{ organization?.legal_name || 'No disponible' }}
                  </template>
                  <template v-else-if="form?.assignment_scope === 'business'">
                    {{ business?.name || 'No disponible' }}
                  </template>
                  <template v-else-if="form?.assignment_scope === 'business_unit'">
                    {{ businessUnit?.name || 'No disponible' }}
                  </template>
                  <template v-else-if="form?.assignment_scope === 'business_unit_group'">
                    {{ businessUnitGroup?.id || 'ID' }}
                  </template>
                  <template v-else>
                    {{ form?.name || 'Formulario' }}
                  </template>
                </td>
              </tr>
              <tr v-if="form?.assignment_scope === 'business_unit_group' && businessUnit?.name">
                <td class="font-weight-bold text-subtitle-1">Ubicación del usuario</td>
                <td>{{ businessUnit?.name }}</td>
              </tr>
              <tr>
                <td class="font-weight-bold text-subtitle-1">Descripción</td>
                <td>{{ form?.description ? form.description : '-' }}</td>
              </tr>
              <tr>
                <td class="font-weight-bold text-subtitle-1">Frecuencia</td>
                <td>
                  {{
                    form?.frequency === 'once_per_day'
                      ? 'Una vez al día'
                      : form?.frequency === 'multiple_per_day'
                        ? 'Muchas veces al día'
                        : form?.frequency || 'No disponible'
                  }}
                </td>
              </tr>
              <tr v-if="form?.has_weighting && form?.max_score">
                <td class="font-weight-bold text-subtitle-1">Puntaje Máximo</td>
                <td>{{ form?.max_score }} puntos</td>
              </tr>
            </tbody>
          </v-table>
        </v-card>
      </v-col>
    </v-row>

    <!-- Mensaje de error de geolocalización o fuera de alcance -->
    <div v-if="geoCheckError" class="text-center pa-8">
      <v-icon :icon="mdiMapMarkerOff" color="red" size="48"></v-icon>
      <p class="text-h6 mt-4">{{ geoCheckError }}</p>
      <v-btn color="primary" class="mt-4" @click="showForm">Reintentar</v-btn>
    </div>

    <!-- Segunda fila: preguntas (cada pregunta en su propio card o grupo) -->
    <template v-else>
      <v-row>
        <v-col cols="12">
          <v-form ref="formRef" @submit.prevent="submitForm">
            <div v-if="isLoading" class="text-center pa-8">
              <v-progress-circular indeterminate color="primary" size="64" />
              <p class="mt-4">Cargando formulario...</p>
            </div>

            <div v-else-if="orderedFieldsAndGroups.length">
              <div v-for="(item, idx) in orderedFieldsAndGroups" :key="item.key" class="mb-4">
                <!-- Pregunta suelta -->
                <template v-if="item.type === 'field'">
                  <v-card class="rounded-lg elevation-1 pa-0 card-business-unit">
                    <v-row no-gutters>
                      <v-col
                        cols="auto"
                        class="d-flex align-center justify-center question-number-desktop"
                        style="min-width: 48px; max-width: 64px; background: #f5f5f5"
                      >
                        <div class="text-h6 font-weight-bold" style="width: 100%; text-align: center">
                          {{ idx + 1 }}
                        </div>
                      </v-col>
                      <v-col>
                        <div class="question-number-mobile mb-2">
                          <div class="text-h6 font-weight-bold question-number-mobile-inner">
                            {{ idx + 1 }}
                          </div>
                        </div>
                        <v-card-text>
                          <v-label class="mb-2 field-label">
                            {{ item.field.label }}
                            <span v-if="item.field.is_required" class="required-asterisk">*</span>
                            <span v-if="item.field.score" class="ml-2 text-caption text-grey" style="font-weight: normal">
                              ({{ item.field.score }} pts)
                            </span>
                          </v-label>

                          <div
                            v-if="item.field.description"
                            class="field-description mb-2"
                            style="white-space: pre-line; display: flex; align-items: flex-start; gap: 0.4em"
                          >
                            <v-icon size="16" color="grey" class="mr-1">mdi-information-outline</v-icon>
                            {{ item.field.description }}
                          </div>

                          <!-- Si el campo requiere evidencia -->
                          <template v-if="item.field.has_evidence">
                            <div class="mb-4">
                              <component
                                :key="'field-' + item.field.id"
                                v-if="
                                  item.field.type !== 'image' &&
                                  item.field.type !== 'document' &&
                                  item.field.type !== 'signature' &&
                                  item.field.type !== 'geolocation'
                                "
                                :is="FIELD_TYPES(item.field)"
                                v-bind="{ ...getFieldProps(item.field), label: undefined }"
                                v-model="formData[item.field.id]"
                                :rules="item.field.is_required ? [(v) => !!v || 'Este campo es requerido'] : []"
                              />
                            </div>
                            <div>
                              <v-file-input
                                :key="fileVersion[`evidence-${item.field.id}`] || 0"
                                :model-value="evidenceData[item.field.id] || []"
                                accept="image/jpeg,image/png,image/jpg"
                                capture
                                multiple
                                :counter="true"
                                :show-size="true"
                                :rules="[(v) => !item.field.is_required || evidenceData[item.field.id]?.length >= 1 || '']"
                                variant="outlined"
                                :chips="true"
                                :clearable="true"
                                label="Adjuntar evidencia"
                                :messages="['Solo se permiten imágenes: JPG, JPEG, PNG. Máx total: 20MB.']"
                                @change="onEvidenceSelected(item.field.id, $event)"
                                @click:clear="
                                  () => {
                                    evidenceData[item.field.id] = [];
                                    bumpVersion(`evidence-${item.field.id}`);
                                  }
                                "
                              />
                              <!-- Previsualización de imágenes de evidencia -->
                              <div v-if="filteredEvidence(item.field.id).length" class="d-flex flex-wrap mt-3 image-preview-row">
                                <div
                                  v-for="(file, idx) in filteredEvidence(item.field.id)"
                                  :key="'evidence-' + (file?.name || file?.url || file) + idx"
                                  style="position: relative; width: 120px; height: 120px; cursor: pointer"
                                  @click="openImageModal(getImagePreview(file))"
                                >
                                  <img
                                    :src="getImagePreview(file)"
                                    :alt="file.name || 'evidencia'"
                                    style="width: 100%; height: 100%; object-fit: cover; border-radius: 8px; border: 1px solid #eee"
                                  />
                                </div>
                              </div>
                              <!-- Imágenes del campo imagen (cuando el campo es tipo imagen y tiene evidencia) -->
                              <div
                                v-if="item.field.type === 'image' && fileData[item.field.id]?.length"
                                class="d-flex flex-wrap mt-3 image-preview-row"
                              >
                                <div
                                  v-for="(file, idx) in fileData[item.field.id]"
                                  :key="'file-' + (file?.name || idx)"
                                  style="position: relative; width: 120px; height: 120px; cursor: pointer"
                                  @click="openImageModal(getImagePreview(file))"
                                >
                                  <img
                                    :src="getImagePreview(file)"
                                    :alt="file?.name"
                                    style="width: 100%; height: 100%; object-fit: cover; border-radius: 8px; border: 1px solid #eee"
                                  />
                                  <v-btn
                                    icon
                                    size="x-small"
                                    color="red"
                                    style="position: absolute; top: 2px; right: 2px; z-index: 2; background: #fff"
                                    @click.stop="removeFile(item.field.id, idx)"
                                  >
                                    <v-icon size="16">mdi-close</v-icon>
                                  </v-btn>
                                </div>
                              </div>
                            </div>
                          </template>

                          <!-- Si no tiene evidencia, render normal -->
                          <template v-else>
                            <!-- Campo Imagen -->
                            <div v-if="item.field.type === 'image'" class="d-flex flex-column align-stretch">
                              <v-file-input
                                :key="fileVersion[item.field.id] || 0"
                                :model-value="fileData[item.field.id] || []"
                                accept="image/*"
                                multiple
                                :counter="true"
                                :show-size="true"
                                :rules="[
                                  (v) =>
                                    !item.field.is_required ||
                                    (fileData[item.field.id]?.length >= 1 &&
                                      fileData[item.field.id]?.length <= (item.field.attributes?.max_files || 4)) ||
                                    ''
                                ]"
                                @change="onFilesSelected(item.field.id, $event)"
                                variant="outlined"
                                :chips="true"
                                :clearable="true"
                                class="w-100"
                                @click:clear="clearFiles(item.field.id)"
                              />

                              <v-btn
                                v-if="isMobile"
                                variant="outlined"
                                color="primary"
                                :prepend-icon="mdiCamera"
                                @click="triggerCamera(item.field.id)"
                                class="w-100 mt-2"
                              >
                                Tomar foto con cámara
                              </v-btn>

                              <input
                                v-if="isMobile"
                                :id="`camera-${item.field.id}`"
                                type="file"
                                accept="image/*"
                                capture="environment"
                                style="display: none"
                                @change="onCameraCapture(item.field.id, $event)"
                              />

                              <div v-if="fileData[item.field.id]?.length" class="d-flex flex-wrap mt-3 image-preview-row">
                                <div
                                  v-for="(file, idx) in fileData[item.field.id]"
                                  :key="file?.name + idx"
                                  style="position: relative; width: 120px; height: 120px; cursor: pointer"
                                  @click="openImageModal(getImagePreview(file))"
                                >
                                  <img
                                    :src="getImagePreview(file)"
                                    :alt="file?.name"
                                    style="width: 100%; height: 100%; object-fit: cover; border-radius: 8px; border: 1px solid #eee"
                                  />
                                </div>
                              </div>
                            </div>

                            <!-- Campo Documento -->
                            <div v-else-if="item.field.type === 'document'" class="d-flex align-center">
                              <v-file-input
                                :key="fileVersion[item.field.id] || 0"
                                :model-value="fileData[item.field.id] || []"
                                accept="application/pdf,.doc,.docx,.xls,.xlsx,.ppt,.pptx"
                                multiple
                                :counter="true"
                                :show-size="true"
                                :rules="[
                                  (v) =>
                                    !item.field.is_required ||
                                    (fileData[item.field.id]?.length >= 1 &&
                                      fileData[item.field.id]?.length <= (item.field.attributes?.max_files || 2)) ||
                                    ''
                                ]"
                                @change="onFilesSelected(item.field.id, $event)"
                                variant="outlined"
                                :chips="true"
                                :clearable="true"
                                class="flex-grow-1"
                                @click:clear="clearFiles(item.field.id)"
                              />
                            </div>

                            <!-- Campo Documento -->
                            <div v-else-if="item.field.type === 'id'" class="d-flex flex-column align-stretch">
                              <template v-if="cameraOpenFieldId !== item.field.id">
                                <v-btn
                                  variant="outlined"
                                  color="primary"
                                  :prepend-icon="mdiCamera"
                                  @click="openDocumentCamera(item.field.id)"
                                >
                                  Tomar foto ID
                                </v-btn>
                                <div v-if="fileData[item.field.id]?.length" class="d-flex flex-wrap mt-3 image-preview-row">
                                  <div v-for="(file, idx) in fileData[item.field.id]" :key="file?.name + idx">
                                    <v-img
                                      :src="getImagePreview(file)"
                                      :alt="file?.name"
                                      style="width: 120px; height: 120px; object-fit: cover; border-radius: 8px; border: 1px solid #eee"
                                    />
                                  </div>
                                </div>
                              </template>
                              <template v-else>
                                <CameraCrop
                                  :ref="
                                    (el) => {
                                      documentCameraRef = el;
                                    }
                                  "
                                  :resolution="{ width: 1200, height: 1200 }"
                                  autoplay
                                  :show-download-button="false"
                                  :show-preview="false"
                                  :show-upload-button="false"
                                  :show-delete-button="false"
                                  class="flex-grow-1"
                                >
                                  <div class="id-frame-overlay">
                                    <div class="id-frame-inner" />
                                    <p class="id-frame-text">Coloca tu documento dentro del marco</p>
                                    <v-btn color="primary" class="id-frame-capture-btn" @click="captureDocumentPhoto()"> Capturar </v-btn>
                                  </div></CameraCrop
                                >
                                <v-btn variant="outlined" size="small" class="mt-2" @click="closeDocumentCamera()"> Cerrar cámara </v-btn>
                              </template>
                            </div>

                            <div v-else-if="item.field.type === 'barcode'" class="d-flex flex-column align-stretch">
                              <template v-if="barcodeOpenFieldId !== item.field.id">
                                <v-btn variant="outlined" color="primary" @click="openBarcodeScanner(item.field.id)"> Escanear </v-btn>
                                <div v-if="formData[item.field.id]" class="mt-2">
                                  <v-chip color="success" size="small">{{ formData[item.field.id] }}</v-chip>
                                </div>
                              </template>
                              <template v-else>
                                <BarCodeRead
                                  :field-id="item.field.id"
                                  @scanned="(val) => handleBarcodeScanned(item.field.id, val)"
                                  @close="closeBarcodeScanner()"
                                />
                              </template>
                            </div>
                            <div v-else-if="item.field.type === 'idscan'" class="d-flex flex-column align-stretch">
                              <template v-if="idScanOpenFieldId === item.field.id">
                                <PassportRead
                                  :field-id="item.field.id"
                                  @scanned="(val) => handleIdScanned(item.field.id, val)"
                                  @close="closeIdScanner()"
                                />
                              </template>
                              <template v-else>
                                <v-btn variant="outlined" color="primary" @click="openIdScanner(item.field.id)"> Escanear </v-btn>

                                <template v-if="getIdScanData(item.field.id)">
                                  <v-card variant="outlined" class="mt-3 pa-3">
                                    <div class="text-caption text-medium-emphasis mb-2">Revisa y edita los datos del documento</div>
                                    <v-text-field
                                      v-model="formData[item.field.id].documentNumber"
                                      label="Número de documento"
                                      density="compact"
                                      variant="outlined"
                                      class="mb-2"
                                      hide-details
                                    />
                                    <v-text-field
                                      v-model="formData[item.field.id].lastName"
                                      label="Apellidos"
                                      density="compact"
                                      variant="outlined"
                                      class="mb-2"
                                      hide-details
                                    />
                                    <v-text-field
                                      v-model="formData[item.field.id].firstName"
                                      label="Nombre(s)"
                                      density="compact"
                                      variant="outlined"
                                      class="mb-2"
                                      hide-details
                                    />
                                    <v-text-field
                                      v-model="formData[item.field.id].birthDate"
                                      label="Fecha de nacimiento (YYMMDD)"
                                      density="compact"
                                      variant="outlined"
                                      placeholder="990326"
                                      class="mb-2"
                                      hide-details
                                    />
                                    <v-text-field
                                      v-model="formData[item.field.id].expirationDate"
                                      label="Fecha de vencimiento (YYMMDD)"
                                      density="compact"
                                      variant="outlined"
                                      placeholder="271231"
                                      class="mb-2"
                                      hide-details
                                    />
                                    <v-text-field
                                      v-model="formData[item.field.id].nationality"
                                      label="Nacionalidad"
                                      density="compact"
                                      variant="outlined"
                                      class="mb-2"
                                      hide-details
                                    />
                                    <v-btn variant="text" size="small" color="primary" class="mt-1" @click="openIdScanner(item.field.id)">
                                      Escanear de nuevo
                                    </v-btn>
                                  </v-card>
                                </template>
                              </template>
                            </div>

                            <!-- Campo Checkbox -->
                            <div v-else-if="item.field.type === 'checkbox'">
                              <div class="checkbox-group">
                                <v-checkbox
                                  v-for="option in item.field.options"
                                  :key="option"
                                  :label="option"
                                  :value="option"
                                  v-model="formData[item.field.id]"
                                  class="ml-4"
                                />
                              </div>
                              <div
                                v-if="triedSubmit && item.field.is_required && !formData[item.field.id]?.length"
                                class="text-caption text-red mt-1"
                              >
                                Este campo es requerido
                              </div>
                            </div>

                            <!-- Campo Firma -->
                            <div v-else-if="item.field.type === 'signature'">
                              <div class="signature-container">
                                <SignaturePad
                                  :ref="
                                    (el) => {
                                      if (el) signatureRefs[item.field.id] = [el];
                                    }
                                  "
                                  @signature-changed="(signatureBlob) => handleSignature(item.field.id, signatureBlob)"
                                />
                                <div class="d-flex flex-wrap justify-end mt-2" v-if="signatureData[item.field.id]?.length">
                                  <div class="mr-2 mb-2" v-for="(file, idx) in signatureData[item.field.id]" :key="idx">
                                    <v-chip
                                      color="green"
                                      class="mr-1"
                                      size="small"
                                      closable
                                      :ripple="false"
                                      @mousedown.stop.prevent
                                      @click:close.stop="handleClearSignature(item.field.id, idx)"
                                    >
                                      <span>{{ file.name }}</span>
                                    </v-chip>
                                  </div>
                                </div>
                              </div>
                              <div
                                v-if="
                                  triedSubmit &&
                                  item.field.is_required &&
                                  (!signatureData[item.field.id] || signatureData[item.field.id].length < 1)
                                "
                                class="text-caption text-red mt-1"
                              >
                                Adjunta al menos una firma
                              </div>
                            </div>

                            <!-- Campo Semáforo (chips, texto centrado) -->
                            <div v-else-if="isSemaforo(item.field)">
                              <div class="semaforo-chips-row">
                                <v-chip
                                  v-for="option in item.field.options"
                                  :key="option"
                                  pill
                                  variant="flat"
                                  class="semaforo-chip"
                                  :style="chipStyleFilled(option, formData[item.field.id] === option)"
                                  @click="formData[item.field.id] = option"
                                  :ripple="false"
                                >
                                  <span class="semaforo-chip-grid">
                                    <span class="semaforo-check-left"></span>
                                    <span class="semaforo-text">{{ option }}</span>
                                    <span class="semaforo-check-right">
                                      <v-icon v-if="formData[item.field.id] === option" size="16">mdi-check</v-icon>
                                    </span>
                                  </span>
                                </v-chip>
                              </div>
                              <div
                                v-if="triedSubmit && item.field.is_required && !formData[item.field.id]"
                                class="text-caption text-red mt-1"
                              >
                                Este campo es requerido
                              </div>
                            </div>

                            <!-- Campo Radio (cuando NO es semáforo) -->
                            <div v-else-if="item.field.type === 'radio'">
                              <v-radio-group
                                v-model="formData[item.field.id]"
                                :rules="item.field.is_required ? [(v) => !!v || 'Este campo es requerido'] : []"
                              >
                                <v-radio v-for="option in item.field.options" :key="option" :label="option" :value="option" class="ml-4" />
                              </v-radio-group>
                            </div>

                            <!-- Campo geolocalización manual -->
                            <div v-else-if="item.field.type === 'geolocation' && item.field.attributes?.mode === 'manual'">
                              <AddressAutocomplete
                                :initialValue="
                                  typeof formData[item.field.id] === 'object' && formData[item.field.id] !== null
                                    ? formData[item.field.id]
                                    : null
                                "
                                mode="create"
                                :addressError="
                                  triedSubmit && item.field.is_required && !isGeolocationManualValid(formData[item.field.id])
                                    ? 'Este campo es requerido'
                                    : ''
                                "
                                @update:parsedAddress="(val) => handleGeolocationManual(item.field.id, val)"
                              />
                              <div
                                v-if="triedSubmit && item.field.is_required && !isGeolocationManualValid(formData[item.field.id])"
                                class="text-caption text-red mt-1"
                              >
                                Este campo es requerido
                              </div>
                            </div>

                            <!-- Otros campos -->
                            <component
                              v-else-if="
                                item.field.type !== 'image' &&
                                item.field.type !== 'document' &&
                                item.field.type !== 'checkbox' &&
                                item.field.type !== 'radio' &&
                                item.field.type !== 'signature' &&
                                item.field.type !== 'geolocation'
                              "
                              :is="FIELD_TYPES(item.field)"
                              v-bind="{ ...getFieldProps(item.field), label: undefined }"
                              v-model="formData[item.field.id]"
                              :rules="item.field.is_required ? [(v) => !!v || 'Este campo es requerido'] : []"
                            />
                          </template>
                        </v-card-text>
                      </v-col>
                    </v-row>
                  </v-card>
                </template>

                <!-- Grupo de preguntas -->
                <template v-else-if="item.type === 'group'">
                  <v-card class="rounded-lg elevation-1 pa-0 card-business-unit">
                    <v-row no-gutters>
                      <v-col
                        cols="auto"
                        class="d-flex align-center justify-center question-number-desktop"
                        style="min-width: 48px; max-width: 64px; background: #f5f5f5"
                      >
                        <div class="text-h6 font-weight-bold" style="width: 100%; text-align: center">{{ idx + 1 }}</div>
                      </v-col>
                      <v-col>
                        <!-- Número del grupo en mobile -->
                        <div class="question-number-mobile mb-2">
                          <div class="text-h6 font-weight-bold question-number-mobile-inner">
                            {{ idx + 1 }}
                          </div>
                        </div>
                        <v-card-title
                          class="d-flex align-center justify-space-between group-title-row"
                          style="cursor: pointer; padding-bottom: 0; align-items: center"
                          @click="toggleGroup(item.group.id)"
                        >
                          <v-label class="mb-2 field-label" style="margin-bottom: 0">
                            {{ item.group.name }}
                          </v-label>
                          <v-icon
                            :icon="expandedGroups[item.group.id] ? mdiChevronUp : mdiChevronDown"
                            size="20"
                            style="margin-left: 8px; vertical-align: middle"
                          />
                        </v-card-title>
                        <v-expand-transition>
                          <div v-show="expandedGroups[item.group.id]" class="pa-2">
                            <div
                              v-for="(field, gIdx) in item.group.fields.filter(
                                (f) => !(f.type === 'geolocation' && f.attributes?.mode === 'scope')
                              )"
                              :key="field.id"
                              class="mb-2"
                            >
                              <v-card class="rounded-lg elevation-0 pa-0">
                                <v-row no-gutters>
                                  <v-col
                                    cols="auto"
                                    class="d-flex align-center justify-center question-number-desktop"
                                    style="min-width: 36px; max-width: 48px; background: #f5f5f5"
                                  >
                                    <div class="text-h6 font-weight-bold" style="width: 100%; text-align: center">
                                      {{ idx + 1 }}.{{ gIdx + 1 }}
                                    </div>
                                  </v-col>
                                  <v-col>
                                    <div class="question-number-mobile mb-2">
                                      <div class="text-h6 font-weight-bold question-number-mobile-inner">{{ idx + 1 }}.{{ gIdx + 1 }}</div>
                                    </div>
                                    <v-card-text>
                                      <v-label class="mb-2 field-label">
                                        {{ field.label }}
                                        <span v-if="field.is_required" class="required-asterisk">*</span>
                                        <span v-if="field.score" class="ml-2 text-caption text-grey" style="font-weight: normal">
                                          ({{ field.score }} pts)
                                        </span>
                                      </v-label>

                                      <div
                                        v-if="field.description"
                                        class="field-description mb-2"
                                        style="white-space: pre-line; display: flex; align-items: flex-start; gap: 0.4em"
                                      >
                                        <v-icon size="16" color="grey" class="mr-1">mdi-information-outline</v-icon>
                                        {{ field.description }}
                                      </div>

                                      <!-- Si el campo requiere evidencia -->
                                      <template v-if="field.has_evidence">
                                        <div class="mb-4">
                                          <component
                                            :key="'field-' + field.id"
                                            v-if="
                                              field.type !== 'image' &&
                                              field.type !== 'document' &&
                                              field.type !== 'signature' &&
                                              field.type !== 'geolocation'
                                            "
                                            :is="FIELD_TYPES(field)"
                                            v-bind="{ ...getFieldProps(field), label: undefined }"
                                            v-model="formData[field.id]"
                                            :rules="field.is_required ? [(v) => !!v || 'Este campo es requerido'] : []"
                                          />
                                        </div>
                                        <div>
                                          <v-file-input
                                            :key="fileVersion[`evidence-${field.id}`] || 0"
                                            :model-value="evidenceData[field.id] || []"
                                            accept="image/jpeg,image/png,image/jpg"
                                            capture
                                            multiple
                                            :counter="true"
                                            :show-size="true"
                                            :rules="[(v) => !field.is_required || evidenceData[field.id]?.length >= 1 || '']"
                                            variant="outlined"
                                            :chips="true"
                                            :clearable="true"
                                            label="Adjuntar evidencia"
                                            :messages="['Solo se permiten imágenes: JPG, JPEG, PNG. Máx total: 20MB.']"
                                            @change="onEvidenceSelected(field.id, $event)"
                                            @click:clear="
                                              () => {
                                                evidenceData[field.id] = [];
                                                bumpVersion(`evidence-${field.id}`);
                                              }
                                            "
                                          />
                                          <!-- Previsualización de imágenes de evidencia -->
                                          <div v-if="filteredEvidence(field.id).length" class="d-flex flex-wrap mt-3 image-preview-row">
                                            <div
                                              v-for="(file, idx) in filteredEvidence(field.id)"
                                              :key="'evidence-' + (file?.name || file?.url || file) + idx"
                                              style="position: relative; width: 120px; height: 120px; cursor: pointer"
                                              @click="openImageModal(getImagePreview(file))"
                                            >
                                              <img
                                                :src="getImagePreview(file)"
                                                :alt="file.name || 'evidencia'"
                                                style="
                                                  width: 100%;
                                                  height: 100%;
                                                  object-fit: cover;
                                                  border-radius: 8px;
                                                  border: 1px solid #eee;
                                                "
                                              />
                                            </div>
                                          </div>
                                          <!-- Imágenes del campo imagen (cuando el campo es tipo imagen y tiene evidencia) -->
                                          <div
                                            v-if="field.type === 'image' && fileData[field.id]?.length"
                                            class="d-flex flex-wrap mt-3 image-preview-row"
                                          >
                                            <div
                                              v-for="(file, idx) in fileData[field.id]"
                                              :key="'file-' + (file?.name || idx)"
                                              style="position: relative; width: 120px; height: 120px; cursor: pointer"
                                              @click="openImageModal(getImagePreview(file))"
                                            >
                                              <img
                                                :src="getImagePreview(file)"
                                                :alt="file?.name"
                                                style="
                                                  width: 100%;
                                                  height: 100%;
                                                  object-fit: cover;
                                                  border-radius: 8px;
                                                  border: 1px solid #eee;
                                                "
                                              />
                                              <v-btn
                                                icon
                                                size="x-small"
                                                color="red"
                                                style="position: absolute; top: 2px; right: 2px; z-index: 2; background: #fff"
                                                @click.stop="removeFile(field.id, idx)"
                                              >
                                                <v-icon size="16">mdi-close</v-icon>
                                              </v-btn>
                                            </div>
                                          </div>
                                        </div>
                                      </template>

                                      <!-- Si no tiene evidencia, render normal -->
                                      <template v-else>
                                        <!-- Campo Imagen -->
                                        <div v-if="field.type === 'image'" class="d-flex flex-column align-stretch">
                                          <v-file-input
                                            :key="fileVersion[field.id] || 0"
                                            :model-value="fileData[field.id] || []"
                                            accept="image/*"
                                            multiple
                                            :counter="true"
                                            :show-size="true"
                                            :rules="[
                                              (v) =>
                                                !field.is_required ||
                                                (fileData[field.id]?.length >= 1 &&
                                                  fileData[field.id]?.length <= (field.attributes?.max_files || 4)) ||
                                                ''
                                            ]"
                                            @change="onFilesSelected(field.id, $event)"
                                            variant="outlined"
                                            :chips="true"
                                            :clearable="true"
                                            class="w-100"
                                            @click:clear="clearFiles(field.id)"
                                          />

                                          <v-btn
                                            v-if="isMobile"
                                            variant="outlined"
                                            color="primary"
                                            :prepend-icon="mdiCamera"
                                            @click="triggerCamera(field.id)"
                                            class="w-100 mt-2"
                                          >
                                            Tomar foto con cámara
                                          </v-btn>

                                          <input
                                            v-if="isMobile"
                                            :id="`camera-${field.id}`"
                                            type="file"
                                            accept="image/*"
                                            capture="environment"
                                            style="display: none"
                                            @change="onCameraCapture(field.id, $event)"
                                          />

                                          <div v-if="fileData[field.id]?.length" class="d-flex flex-wrap mt-3 image-preview-row">
                                            <div
                                              v-for="(file, idx) in fileData[field.id]"
                                              :key="file?.name + idx"
                                              style="position: relative; width: 120px; height: 120px; cursor: pointer"
                                              @click="openImageModal(getImagePreview(file))"
                                            >
                                              <img
                                                :src="getImagePreview(file)"
                                                :alt="file?.name"
                                                style="
                                                  width: 100%;
                                                  height: 100%;
                                                  object-fit: cover;
                                                  border-radius: 8px;
                                                  border: 1px solid #eee;
                                                "
                                              />
                                            </div>
                                          </div>
                                        </div>

                                        <!-- Campo Documento -->
                                        <div v-else-if="field.type === 'document'" class="d-flex align-center">
                                          <v-file-input
                                            :key="fileVersion[field.id] || 0"
                                            :model-value="fileData[field.id] || []"
                                            accept="application/pdf,.doc,.docx,.xls,.xlsx,.ppt,.pptx"
                                            multiple
                                            :counter="true"
                                            :show-size="true"
                                            :rules="[
                                              (v) =>
                                                !field.is_required ||
                                                (fileData[field.id]?.length >= 1 &&
                                                  fileData[field.id]?.length <= (field.attributes?.max_files || 2)) ||
                                                ''
                                            ]"
                                            @change="onFilesSelected(field.id, $event)"
                                            variant="outlined"
                                            :chips="true"
                                            :clearable="true"
                                            class="flex-grow-1"
                                            @click:clear="clearFiles(field.id)"
                                          />
                                        </div>
                                        <!-- Campo ID -->
                                        <div v-else-if="field.type === 'id'" class="d-flex flex-column align-stretch">
                                          <template v-if="cameraOpenFieldId !== field.id">
                                            <v-btn
                                              variant="outlined"
                                              color="primary"
                                              :prepend-icon="mdiCamera"
                                              @click="openDocumentCamera(field.id)"
                                            >
                                              Tomar foto ID
                                            </v-btn>
                                          </template>
                                          <template v-else>
                                            <CameraCrop
                                              :ref="
                                                (el) => {
                                                  documentCameraRef = el;
                                                }
                                              "
                                              :resolution="{ width: 1200, height: 1200 }"
                                              autoplay
                                              :show-download-button="false"
                                              :show-preview="false"
                                              :show-upload-button="false"
                                              :show-delete-button="false"
                                              class="flex-grow-1"
                                            >
                                              <div class="id-frame-overlay">
                                                <div class="id-frame-inner" />
                                                <p class="id-frame-text">Coloca tu documento dentro del marco</p>
                                                <v-btn color="primary" class="id-frame-capture-btn" @click="captureDocumentPhoto()">
                                                  Capturar
                                                </v-btn>
                                              </div></CameraCrop
                                            >
                                            <v-btn variant="outlined" size="small" class="mt-2" @click="closeDocumentCamera()">
                                              Cerrar cámara
                                            </v-btn>
                                          </template>
                                        </div>

                                        <div v-else-if="field.type === 'barcode'" class="d-flex flex-column align-stretch">
                                          <template v-if="barcodeOpenFieldId !== field.id">
                                            <v-btn variant="outlined" color="primary" @click="openBarcodeScanner(field.id)">
                                              Escanear
                                            </v-btn>
                                            <div v-if="formData[field.id]" class="mt-2">
                                              <v-chip color="success" size="small">{{ formData[field.id] }}</v-chip>
                                            </div>
                                          </template>
                                          <template v-else>
                                            <BarCodeRead
                                              :field-id="field.id"
                                              @scanned="(val) => handleBarcodeScanned(field.id, val)"
                                              @close="closeBarcodeScanner()"
                                            />
                                          </template>
                                        </div>
                                        <div v-else-if="field.type === 'idscan'" class="d-flex flex-column align-stretch">
                                          <template v-if="idScanOpenFieldId === field.id">
                                            <PassportRead
                                              :field-id="field.id"
                                              @scanned="(val) => handleIdScanned(field.id, val)"
                                              @close="closeIdScanner()"
                                            />
                                          </template>
                                          <template v-else>
                                            <v-btn variant="outlined" color="primary" @click="openIdScanner(field.id)"> Escanear </v-btn>

                                            <template v-if="getIdScanData(field.id)">
                                              <v-card variant="outlined" class="mt-3 pa-3">
                                                <div class="text-caption text-medium-emphasis mb-2">
                                                  Revisa y edita los datos del documento
                                                </div>
                                                <v-text-field
                                                  v-model="formData[field.id].documentNumber"
                                                  label="Número de documento"
                                                  density="compact"
                                                  variant="outlined"
                                                  class="mb-2"
                                                  hide-details
                                                />
                                                <v-text-field
                                                  v-model="formData[field.id].lastName"
                                                  label="Apellidos"
                                                  density="compact"
                                                  variant="outlined"
                                                  class="mb-2"
                                                  hide-details
                                                />
                                                <v-text-field
                                                  v-model="formData[field.id].firstName"
                                                  label="Nombre(s)"
                                                  density="compact"
                                                  variant="outlined"
                                                  class="mb-2"
                                                  hide-details
                                                />
                                                <v-text-field
                                                  v-model="formData[field.id].birthDate"
                                                  label="Fecha de nacimiento (YYMMDD)"
                                                  density="compact"
                                                  variant="outlined"
                                                  placeholder="990326"
                                                  class="mb-2"
                                                  hide-details
                                                />
                                                <v-text-field
                                                  v-model="formData[field.id].expirationDate"
                                                  label="Fecha de vencimiento (YYMMDD)"
                                                  density="compact"
                                                  variant="outlined"
                                                  placeholder="271231"
                                                  class="mb-2"
                                                  hide-details
                                                />
                                                <v-text-field
                                                  v-model="formData[field.id].nationality"
                                                  label="Nacionalidad"
                                                  density="compact"
                                                  variant="outlined"
                                                  class="mb-2"
                                                  hide-details
                                                />
                                                <v-btn
                                                  variant="text"
                                                  size="small"
                                                  color="primary"
                                                  class="mt-1"
                                                  @click="openIdScanner(field.id)"
                                                >
                                                  Escanear de nuevo
                                                </v-btn>
                                              </v-card>
                                            </template>
                                          </template>
                                        </div>

                                        <!-- Campo Checkbox -->
                                        <div v-else-if="field.type === 'checkbox'">
                                          <div class="checkbox-group">
                                            <v-checkbox
                                              v-for="option in field.options"
                                              :key="option"
                                              :label="option"
                                              :value="option"
                                              v-model="formData[field.id]"
                                              class="ml-4"
                                            />
                                          </div>
                                          <div
                                            v-if="triedSubmit && field.is_required && !formData[field.id]?.length"
                                            class="text-caption text-red mt-1"
                                          >
                                            Este campo es requerido
                                          </div>
                                        </div>

                                        <!-- Campo Firma -->
                                        <div v-else-if="field.type === 'signature'">
                                          <div class="signature-container">
                                            <SignaturePad
                                              :ref="
                                                (el) => {
                                                  if (el) signatureRefs[field.id] = [el];
                                                }
                                              "
                                              @signature-changed="(signatureBlob) => handleSignature(field.id, signatureBlob)"
                                            />
                                            <div class="d-flex flex-wrap justify-end mt-2" v-if="signatureData[field.id]?.length">
                                              <div class="mr-2 mb-2" v-for="(file, idx) in signatureData[field.id]" :key="idx">
                                                <v-chip
                                                  color="green"
                                                  class="mr-1"
                                                  size="small"
                                                  closable
                                                  :ripple="false"
                                                  @mousedown.stop.prevent
                                                  @click:close.stop="handleClearSignature(field.id, idx)"
                                                >
                                                  <span>{{ file.name }}</span>
                                                </v-chip>
                                              </div>
                                            </div>
                                          </div>
                                          <div
                                            v-if="
                                              triedSubmit &&
                                              field.is_required &&
                                              (!signatureData[field.id] || signatureData[field.id].length < 1)
                                            "
                                            class="text-caption text-red mt-1"
                                          >
                                            Adjunta al menos una firma
                                          </div>
                                        </div>

                                        <!-- Campo Semáforo (chips, texto centrado) -->
                                        <div v-else-if="isSemaforo(field)">
                                          <div class="semaforo-chips-row">
                                            <v-chip
                                              v-for="option in field.options"
                                              :key="option"
                                              pill
                                              variant="flat"
                                              class="semaforo-chip"
                                              :style="chipStyleFilled(option, formData[field.id] === option)"
                                              @click="formData[field.id] = option"
                                              :ripple="false"
                                            >
                                              <span class="semaforo-chip-grid">
                                                <span class="semaforo-check-left"></span>
                                                <span class="semaforo-text">{{ option }}</span>
                                                <span class="semaforo-check-right">
                                                  <v-icon v-if="formData[field.id] === option" size="16">mdi-check</v-icon>
                                                </span>
                                              </span>
                                            </v-chip>
                                          </div>
                                          <div
                                            v-if="triedSubmit && field.is_required && !formData[field.id]"
                                            class="text-caption text-red mt-1"
                                          >
                                            Este campo es requerido
                                          </div>
                                        </div>

                                        <!-- Campo Radio (cuando NO es semáforo) -->
                                        <div v-else-if="field.type === 'radio'">
                                          <v-radio-group
                                            v-model="formData[field.id]"
                                            :rules="field.is_required ? [(v) => !!v || 'Este campo es requerido'] : []"
                                          >
                                            <v-radio
                                              v-for="option in field.options"
                                              :key="option"
                                              :label="option"
                                              :value="option"
                                              class="ml-4"
                                            />
                                          </v-radio-group>
                                        </div>

                                        <!-- Campo geolocalización manual -->
                                        <div v-else-if="field.type === 'geolocation' && field.attributes?.mode === 'manual'">
                                          <AddressAutocomplete
                                            :initialValue="
                                              typeof formData[field.id] === 'object' && formData[field.id] !== null
                                                ? formData[field.id]
                                                : null
                                            "
                                            mode="create"
                                            :addressError="
                                              triedSubmit && field.is_required && !isGeolocationManualValid(formData[field.id])
                                                ? 'Este campo es requerido'
                                                : ''
                                            "
                                            @update:parsedAddress="(val) => handleGeolocationManual(field.id, val)"
                                          />
                                          <div
                                            v-if="triedSubmit && field.is_required && !isGeolocationManualValid(formData[field.id])"
                                            class="text-caption text-red mt-1"
                                          >
                                            Este campo es requerido
                                          </div>
                                        </div>

                                        <!-- Otros campos -->
                                        <component
                                          v-else-if="
                                            field.type !== 'image' &&
                                            field.type !== 'document' &&
                                            field.type !== 'checkbox' &&
                                            field.type !== 'radio' &&
                                            field.type !== 'signature' &&
                                            field.type !== 'geolocation'
                                          "
                                          :is="FIELD_TYPES(field)"
                                          v-bind="{ ...getFieldProps(field), label: undefined }"
                                          v-model="formData[field.id]"
                                          :rules="field.is_required ? [(v) => !!v || 'Este campo es requerido'] : []"
                                        />
                                      </template>
                                    </v-card-text>
                                  </v-col>
                                </v-row>
                              </v-card>
                            </div>
                          </div>
                        </v-expand-transition>
                      </v-col>
                    </v-row>
                  </v-card>
                </template>
              </div>
            </div>

            <div v-else class="text-center pa-8">
              <p class="text-grey">No se pudo cargar el formulario o no tiene campos</p>
            </div>
          </v-form>
        </v-col>
      </v-row>

      <!-- Modal para ver imagen grande -->
      <v-dialog v-model="imageModal.open" max-width="600px">
        <v-card>
          <v-img :src="imageModal.src" max-width="100%" max-height="80vh" style="object-fit: contain" />
          <v-card-actions class="justify-end">
            <v-btn color="primary" text @click="imageModal.open = false">Cerrar</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>

      <!-- Fila separada: botón enviar en la esquina derecha -->
      <v-row>
        <v-col cols="12" class="d-flex justify-end">
          <v-btn color="primary" class="mt-6" :loading="submitting" :disabled="isLoading" @click="submitForm">
            <template #loader>
              <v-progress-circular indeterminate color="white" size="20" />
            </template>
            <v-icon :icon="mdiCheck" class="mr-2" />
            {{ submitting ? 'Enviando...' : 'Enviar Formulario' }}
          </v-btn>
        </v-col>
      </v-row>
    </template>
  </v-container>
</template>

<script setup>
import axiosInstance from '@/utils/axios';
import { ref, onMounted, computed, reactive, watch, onUnmounted } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { mdiArrowLeft, mdiCheck, mdiMapMarkerOff, mdiCamera, mdiChevronUp, mdiChevronDown } from '@mdi/js';
import { FIELD_TYPES, getFieldProps } from '@/constants/constants';
import { useToast } from 'vue-toastification';
import { convertoToString } from '@/utils/helpers/formHelper';
import SignaturePad from '@/styles/SignaturePad.vue';
import AddressAutocomplete from '@/utils/helpers/google/AddressAutocomplete.vue';
import imageCompression from 'browser-image-compression';
import BarCodeRead from '@/utils/helpers/BarCodeRead.vue';
import PassportRead from '@/utils/helpers/PassportRead.vue';

const toast = useToast();
const router = useRouter();
const route = useRoute();
const formId = ref(route.params.id);

const formData = reactive({});
const fileData = reactive({});
const signatureData = reactive({});
const evidenceData = reactive({});

const isLoading = ref(false);
const submitting = ref(false);
const form = ref(null);
const formRef = ref(null);
const signatureRefs = ref({});

const organization = ref(null);
const business = ref(null);
const businessUnit = ref(null);
const businessUnitGroup = ref(null);

const triedSubmit = ref(false);

const userLocation = ref(null);
const geoCheckError = ref(null);

const formStartTime = ref(null);

const cameraOpenFieldId = ref(null);
const documentCameraRef = ref(null);

async function captureDocumentPhoto() {
  const fieldId = cameraOpenFieldId.value;
  if (!fieldId || !documentCameraRef.value) return;
  try {
    const blob = await documentCameraRef.value?.snapshot();
    if (!blob) return;
    const field = form.value?.fields?.find((f) => f.id == fieldId);
    const fileName = `id_${field?.label ?? fieldId}.jpg`;
    const file = new File([blob], fileName, { type: 'image/jpeg' });
    fileData[fieldId] = [file];
    formData[fieldId] = [fileName];
    bumpVersion(fieldId);
    toast.success('Foto capturada correctamente');
    closeDocumentCamera();
  } catch (error) {
    console.error('Error al capturar foto del documento:', error);
    toast.error('Error al capturar foto del documento');
  }
}

function openDocumentCamera(fieldId) {
  cameraOpenFieldId.value = fieldId;
}

function closeDocumentCamera() {
  cameraOpenFieldId.value = null;
}

const barcodeOpenFieldId = ref(null);
const idScanOpenFieldId = ref(null);

function openBarcodeScanner(fieldId) {
  barcodeOpenFieldId.value = fieldId;
}

function closeBarcodeScanner() {
  barcodeOpenFieldId.value = null;
}

function openIdScanner(fieldId) {
  idScanOpenFieldId.value = fieldId;
}

function closeIdScanner() {
  idScanOpenFieldId.value = null;
}

function handleBarcodeScanned(fieldId, value) {
  const current = Array.isArray(formData[fieldId]) ? formData[fieldId] : [];
  formData[fieldId] = [...current, value]; // array
  bumpVersion(fieldId);
  toast.success('Código escaneado correctamente');
  closeBarcodeScanner();
}

function handleIdScanned(fieldId, value) {
  formData[fieldId] = {
    documentNumber: value.documentNumber,
    format: value.format ?? 'N/A',
    lastName: value.lastName ?? 'N/A',
    firstName: value.firstName ?? 'N/A',
    birthDate: value.birthDate ?? 'N/A',
    expirationDate: value.expirationDate ?? 'N/A',
    nationality: value.nationality ?? 'N/A'
  };
  bumpVersion(fieldId);
  toast.success('Documento escaneado. Puedes editar los campos si es necesario');
  closeIdScanner();
}

function getIdScanData(fieldId) {
  const data = formData[fieldId];
  return Array.isArray(data) ? data[0] : data;
}

// --- INTEGRACIÓN: Dirección de geolocalización para campos scope ---
const geoAddress = ref('');
const fetchAddressFromCoords = async (lat, lng) => {
  try {
    const url = `https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lng}`;
    const res = await fetch(url, { headers: { 'Accept-Language': 'es' } });
    const data = await res.json();
    return data.display_name || '';
  } catch (e) {
    return '';
  }
};
// ---------------------------------------------------------------

const fileVersion = ref({});
const bumpVersion = (key) => {
  fileVersion.value[key] = (fileVersion.value[key] || 0) + 1;
};

const isMobile = computed(() => {
  return /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent) || window.innerWidth <= 768;
});

const SEMAFORO_COLORS = { Alto: '#e53935', Medio: '#ffd600', Bajo: '#43a047' };
const SEMAFORO_PASTEL = { Alto: '#ffebee', Medio: '#fffde7', Bajo: '#e8f5e9' };
const OPTION_KEYS = {
  alto: 'Alto',
  alta: 'Alto',
  high: 'Alto',
  rojo: 'Alto',
  medio: 'Medio',
  media: 'Medio',
  amarillo: 'Medio',
  medium: 'Medio',
  bajo: 'Bajo',
  baja: 'Bajo',
  low: 'Bajo',
  verde: 'Bajo'
};
const mapOptionKey = (opt) => {
  const k = String(opt || '')
    .trim()
    .toLowerCase();
  return OPTION_KEYS[k] || (['alto', 'medio', 'bajo'].includes(k) ? k[0].toUpperCase() + k.slice(1) : opt);
};
const isSemaforo = (field) => {
  const type = String(field?.type || '').toLowerCase();
  if (type === 'semaforo') return true;
  if (field?.attributes?.kind === 'semaforo' || field?.attributes?.display === 'semaforo') return true;
  const opts = (field?.options || []).map((o) => String(o).trim().toLowerCase());
  const hasAlto = opts.some((o) => ['alto', 'alta', 'rojo', 'high'].includes(o));
  const hasMedio = opts.some((o) => ['medio', 'media', 'amarillo', 'medium'].includes(o));
  const hasBajo = opts.some((o) => ['bajo', 'baja', 'verde', 'low'].includes(o));
  return type === 'radio' && hasAlto && hasMedio && hasBajo;
};
const chipStyleFilled = (option, isSelected) => {
  const key = mapOptionKey(option);
  const base = SEMAFORO_COLORS[key] || '#9e9e9e';
  const pastel = SEMAFORO_PASTEL[key] || '#f5f5f5';
  const text = key === 'Medio' ? '#111' : '#fff';
  return {
    background: isSelected ? base : pastel,
    color: isSelected ? text : '#222',
    border: 'none',
    padding: '4px 10px',
    fontWeight: isSelected ? '700' : '600',
    boxShadow: isSelected ? '0 0 0 2px rgba(0,0,0,0.06), 0 2px 8px rgba(0,0,0,0.12)' : 'none',
    transform: isSelected ? 'translateY(-1px)' : 'none',
    transition: 'all .15s ease-in-out',
    cursor: 'pointer',
    userSelect: 'none'
  };
};

const visibleFields = computed(
  () => form.value?.fields?.filter((f) => !(f.type === 'geolocation' && f.attributes?.mode === 'scope')) || []
);

const triggerCamera = (fieldId) => {
  const cameraInput = document.getElementById(`camera-${fieldId}`);
  if (cameraInput) {
    cameraInput.click();
  }
};

const onCameraCapture = async (fieldId, evt) => {
  let capturedFiles = normalizeEventFiles(evt);
  if (capturedFiles.length === 0) return;

  const field = form.value.fields.find((f) => f.id == fieldId);
  const maxFiles = field?.attributes?.max_files || 4;

  const current = Array.isArray(fileData[fieldId]) ? fileData[fieldId] : [];

  // Comprimir fotos capturadas
  capturedFiles = await Promise.all(
    capturedFiles.map((file) => {
      if (file && file.type && file.type.startsWith('image/')) {
        return compressImage(file);
      }
      return file;
    })
  );

  let combined = [...current, ...capturedFiles];

  combined = combined.filter((f, idx, arr) => f && arr.findIndex((x) => x && x.name === f.name && x.size === f.size) === idx);

  combined = combined.slice(0, maxFiles);

  fileData[fieldId] = [...combined];
  formData[fieldId] = combined.map((f) => f?.name);

  bumpVersion(fieldId);
};

const filteredEvidence = (fieldId) => {
  return (evidenceData[fieldId] || []).filter(
    (file) =>
      file &&
      ((file.type && file.type.startsWith('image/')) ||
        (typeof file === 'object' && file.url && /\.(jpg|jpeg|png)$/i.test(file.url)) ||
        (typeof file === 'string' && /\.(jpg|jpeg|png)$/i.test(file)))
  );
};

const getUserLocation = () =>
  new Promise((resolve, reject) => {
    const lat = localStorage.getItem('geo_lat');
    const lng = localStorage.getItem('geo_lng');
    if (lat !== null && lng !== null && lat !== '' && lng !== '' && !isNaN(parseFloat(lat)) && !isNaN(parseFloat(lng))) {
      // Obtener dirección
      fetchAddressFromCoords(parseFloat(lat), parseFloat(lng))
        .then((address) => {
          geoAddress.value = address;
          resolve({ lat: parseFloat(lat), lng: parseFloat(lng) });
        })
        .catch((err) => {
          reject(err);
        });
      resolve({ lat: parseFloat(lat), lng: parseFloat(lng) });
    } else {
      reject('No se encontró la ubicación del usuario. Por favor, inicia sesión de nuevo.');
    }
  });

const showForm = async () => {
  isLoading.value = true;
  geoCheckError.value = null;
  try {
    let res = await axiosInstance.get(`/forms/${formId.value}`);
    form.value = res.data.form || res.data.forms?.[0] || res.data.data;

    organization.value = form.value?.organization || null;
    business.value = form.value?.business || null;
    businessUnit.value = form.value?.business_unit || null;
    businessUnitGroup.value = form.value?.business_unit_group || null;

    (form.value?.fields || []).forEach((field) => {
      if (formData[field.id] === undefined) {
        if (field.type === 'checkbox') formData[field.id] = [];
        else formData[field.id] = '';
      }
      if (evidenceData[field.id] === undefined) {
        evidenceData[field.id] = [];
      }
    });

    const geoField = form.value?.fields?.find((f) => f.type === 'geolocation' && f.attributes?.mode === 'manual');
    if (geoField) {
      try {
        userLocation.value = await getUserLocation();
        if (
          typeof userLocation.value.lat !== 'number' ||
          typeof userLocation.value.lng !== 'number' ||
          isNaN(userLocation.value.lat) ||
          isNaN(userLocation.value.lng)
        ) {
          geoCheckError.value = 'Ubicación inválida. Por favor, vuelve a iniciar sesión.';
          isLoading.value = false;
          return;
        }

        // Obtener dirección y rellenar campos de geolocalización manual
        const address = await fetchAddressFromCoords(userLocation.value.lat, userLocation.value.lng);
        geoAddress.value = address;

        // Rellenar automáticamente los campos de geolocalización manual
        form.value.fields.forEach((field) => {
          if (field.type === 'geolocation' && field.attributes?.mode === 'manual') {
            handleGeolocationManual(field.id, {
              latitude: userLocation.value.lat,
              longitude: userLocation.value.lng,
              address
            });
          }
        });
      } catch (err) {
        geoCheckError.value = err || 'No se pudo obtener tu ubicación';
        isLoading.value = false;
        return;
      }
    }
  } catch (err) {
    geoCheckError.value = 'No se pudo cargar el formulario';
  } finally {
    isLoading.value = false;
  }
};

let geoWatchId = null;

onMounted(() => {
  showForm();

  watch(
    form,
    (newForm) => {
      const geoField = newForm?.fields?.find((f) => f.type === 'geolocation' && f.attributes?.mode === 'scope');
      if (geoField && navigator.geolocation) {
        if (geoWatchId !== null) return;
        geoWatchId = navigator.geolocation.watchPosition(
          async (pos) => {
            userLocation.value = {
              lat: pos.coords.latitude,
              lng: pos.coords.longitude
            };
            geoAddress.value = await fetchAddressFromCoords(pos.coords.latitude, pos.coords.longitude);
            try {
              await axiosInstance.post('/user/location', {
                latitude: pos.coords.latitude,
                longitude: pos.coords.longitude,
                accuracy: pos.coords.accuracy,
                recorded_at: new Date().toISOString()
              });
            } catch (e) {
              console.error('No se pudo guardar la ubicación del usuario:', e);
            }
          },
          (err) => {
            geoCheckError.value = 'No se pudo obtener tu ubicación en tiempo real.';
          },
          { enableHighAccuracy: true }
        );
      }
    },
    { immediate: true }
  );
});

onUnmounted(() => {
  if (geoWatchId !== null) {
    navigator.geolocation.clearWatch(geoWatchId);
    geoWatchId = null;
  }
});

const goBack = () => {
  router.push('/mis-formularios');
};

const normalizeEventFiles = (evt) => {
  if (Array.isArray(evt)) return evt;
  if (evt instanceof FileList) return Array.from(evt);
  if (evt && evt.target && evt.target.files) return Array.from(evt.target.files);
  return [];
};

const getImagePreview = (file) => {
  if (!file) {
    return '';
  }

  if (file.previewUrl) {
    return file.previewUrl;
  }

  if (file instanceof File) {
    const url = URL.createObjectURL(file);

    Object.defineProperty(file, 'previewUrl', {
      value: url,
      writable: true,
      configurable: true
    });

    return url;
  }

  if (typeof file === 'object' && file.url) {
    return file.url;
  }
  if (typeof file === 'string') {
    return file;
  }
  return '';
};

const onFilesSelected = async (fieldId, evt) => {
  let selected = normalizeEventFiles(evt);
  const field = form.value.fields.find((f) => f.id == fieldId);
  const maxFiles = field?.attributes?.max_files || (field.type === 'document' ? 2 : 4);

  const current = Array.isArray(fileData[fieldId]) ? fileData[fieldId] : [];

  if (field.type === 'image' || field.type === 'file') {
    selected = await Promise.all(
      selected.map((file) => {
        if (file && file.type && file.type.startsWith('image/')) {
          return compressImage(file);
        }
        return file;
      })
    );
  }

  let combined = [...current, ...selected];

  combined = combined.filter((f, idx, arr) => f && arr.findIndex((x) => x && x.name === f.name && x.size === f.size) === idx);

  combined = combined.slice(0, maxFiles);

  fileData[fieldId] = [...combined];
  formData[fieldId] = combined.map((f) => f?.name);

  bumpVersion(fieldId);
};

const removeFile = (fieldId, idx) => {
  const current = Array.isArray(fileData[fieldId]) ? [...fileData[fieldId]] : [];
  if (!current.length) return;
  current.splice(idx, 1);
  fileData[fieldId] = current;
  formData[fieldId] = current.map((f) => f?.name);

  bumpVersion(fieldId);
};

const clearFiles = (fieldId) => {
  fileData[fieldId] = [];
  formData[fieldId] = [];
  bumpVersion(fieldId);
};

const onEvidenceSelected = async (fieldId, evt) => {
  let selected = normalizeEventFiles(evt);
  const maxFiles = 4;

  selected = selected
    .filter((file) => {
      if (!file?.name) return false;
      const ext = file.name.split('.').pop().toLowerCase();
      return ['jpg', 'jpeg', 'png'].includes(ext);
    })
    .map((file) => {
      if (!file?.type && file?.name) {
        const ext = file.name.split('.').pop().toLowerCase();
        if (['jpg', 'jpeg'].includes(ext)) file.type = 'image/jpeg';
        else if (ext === 'png') file.type = 'image/png';
      }
      return file;
    });

  selected = await Promise.all(
    selected.map((file) => {
      if (file && file.type && file.type.startsWith('image/')) {
        return compressImage(file);
      }
      return file;
    })
  );

  const current = Array.isArray(evidenceData[fieldId]) ? evidenceData[fieldId] : [];
  let combined = [...current, ...selected];

  combined = combined.filter((f, idx, arr) => f && arr.findIndex((x) => x && x.name === f.name && x.size === f.size) === idx);

  combined = combined.slice(0, maxFiles);
  evidenceData[fieldId] = [...combined];

  bumpVersion(`evidence-${fieldId}`);
};

const removeEvidence = (fieldId, idx) => {
  const current = Array.isArray(evidenceData[fieldId]) ? [...evidenceData[fieldId]] : [];
  if (!current.length) return;
  current.splice(idx, 1);
  evidenceData[fieldId] = current;

  bumpVersion(`evidence-${fieldId}`);
};

const imageModal = reactive({
  open: false,
  src: ''
});
const openImageModal = (src) => {
  imageModal.src = src;
  imageModal.open = true;
};

const compressImage = async (file) => {
  const options = {
    maxSizeMB: 0.5,
    maxWidthOrHeight: 1200,
    useWebWorker: true,
    fileType: file.type
  };

  try {
    const compressedBlob = await imageCompression(file, options);

    const originalName = file.name;
    const mimeType = file.type;

    const finalFile = new File([compressedBlob], originalName, {
      type: mimeType,
      lastModified: Date.now()
    });

    if (!finalFile.previewUrl) {
      finalFile.previewUrl = URL.createObjectURL(finalFile);
    }

    return finalFile;
  } catch (error) {
    console.error('Error al comprimir imagen:', error);
    return file;
  }
};

const handleSignature = (fieldId, signatureBlob) => {
  if (!signatureBlob) {
    signatureData[fieldId] = [];
    formData[fieldId] = [];
    return;
  }

  if (signatureBlob instanceof Blob) {
    const field = form.value.fields.find((f) => f.id == fieldId);
    const maxFiles = field?.attributes?.max_files || 1;

    const fileName = `firma_${field?.label || fieldId}.png`;
    const signatureFile = new File([signatureBlob], fileName, {
      type: 'image/png',
      lastModified: Date.now()
    });

    signatureData[fieldId] = [signatureFile].slice(0, maxFiles);
    formData[fieldId] = [fileName].slice(0, maxFiles);
  }
};

const handleClearSignature = (fieldId, idx = null) => {
  if (idx !== null && Array.isArray(signatureData[fieldId])) {
    const arr = [...signatureData[fieldId]];
    arr.splice(idx, 1);
    signatureData[fieldId] = arr;
    formData[fieldId] = arr.map((f) => f?.name);
  } else {
    signatureData[fieldId] = [];
    formData[fieldId] = [];
  }
};

const handleGeolocationManual = (fieldId, val) => {
  formData[fieldId] = val;
};
const isGeolocationManualValid = (val) => !!val && !!val.latitude && !!val.longitude;

const submitForm = async () => {
  triedSubmit.value = true;
  if (!form.value) return;

  const allFields = [];
  if (form.value?.fields && form.value?.field_groups) {
    const groupMap = {};
    form.value.field_groups.forEach((g) => {
      groupMap[g.id] = { ...g, fields: [] };
    });
    form.value.fields.forEach((f) => {
      if (f.form_field_group_id && groupMap[f.form_field_group_id]) {
        groupMap[f.form_field_group_id].fields.push(f);
      } else {
        allFields.push(f);
      }
    });
    form.value.field_groups.forEach((g) => {
      if (groupMap[g.id]?.fields?.length) {
        groupMap[g.id].fields.forEach((f) => allFields.push(f));
      }
    });
  } else {
    allFields.push(...(form.value?.fields || []));
  }

  const requiredFields = allFields.filter((f) => f.is_required);
  const missingFields = requiredFields.filter((field) => {
    if (field.has_evidence && (!evidenceData[field.id] || evidenceData[field.id].length < 1)) return true;
    if (field.type === 'checkbox') return !formData[field.id] || formData[field.id].length === 0;
    if (field.type === 'image' || field.type === 'document') return !fileData[field.id] || fileData[field.id].length < 1;
    if (field.type === 'signature') return !signatureData[field.id] || signatureData[field.id].length < 1;
    if (field.type === 'geolocation' && field.attributes?.mode === 'manual') return !isGeolocationManualValid(formData[field.id]);
    if (field.type === 'geolocation' && field.attributes?.mode === 'scope') {
      // Validar que userLocation esté presente y tenga lat/lng
      return !userLocation.value || typeof userLocation.value.lat !== 'number' || typeof userLocation.value.lng !== 'number';
    }
    return !formData[field.id];
  });

  const tooManyFiles = allFields.some(
    (field) =>
      ((field.type === 'image' || field.type === 'document') &&
        fileData[field.id] &&
        fileData[field.id].length > (field.attributes?.max_files || (field.type === 'document' ? 2 : 4))) ||
      (field.type === 'signature' && signatureData[field.id] && signatureData[field.id].length > (field.attributes?.max_files || 1))
  );

  const geoField = form.value?.fields?.find((f) => f.type === 'geolocation' && f.attributes?.mode === 'scope');
  if (geoField) {
    if (
      !userLocation.value ||
      typeof userLocation.value.lat !== 'number' ||
      typeof userLocation.value.lng !== 'number' ||
      isNaN(userLocation.value.lat) ||
      isNaN(userLocation.value.lng)
    ) {
      toast.error('No se pudo obtener tu ubicación. Por favor, recarga la página o inicia sesión de nuevo.');
      return;
    }
  }

  if (missingFields.length > 0) {
    toast.error('Por favor completa todos los campos requeridos');
    return;
  }
  if (tooManyFiles) {
    toast.error('Solo puedes subir hasta el máximo de archivos permitido por campo');
    return;
  }

  submitting.value = true;
  try {
    const dataToSend = new FormData();

    const answers = allFields.map((field) => {
      let value;
      if (field.type === 'barcode') {
        const list = Array.isArray(formData[field.id]) ? formData[field.id] : [];
        value = JSON.stringify(list);
      } else if (field.type === 'image' || field.type === 'document' || field.type === 'signature' || field.type === 'id') {
        value = (formData[field.id] || []).join(',');
      } else if (field.type === 'geolocation' && field.attributes?.mode === 'manual') {
        value = JSON.stringify(formData[field.id] || {});
      } else if (field.type === 'geolocation' && field.attributes?.mode === 'scope') {
        value = userLocation.value
          ? JSON.stringify({
              lat: userLocation.value.lat,
              lng: userLocation.value.lng,
              address: geoAddress.value || ''
            })
          : '';
      } else {
        value = convertoToString(formData[field.id]);
      }
      const ans = { form_field_id: field.id, value };
      if (field.type === 'image' || field.type === 'document' || field.type === 'signature' || field.type === 'id') ans.is_file = true;
      return ans;
    });

    // --- DEBUG: Verifica que la geolocalización se envía correctamente ---
    const geoFieldAnswer = answers.find((a) => {
      const field = allFields.find((f) => f.id === a.form_field_id);
      return field && field.type === 'geolocation' && field.attributes?.mode === 'scope';
    });

    if (formStartTime.value) {
      dataToSend.append('started_at', formStartTime.value);
    }

    dataToSend.append('answers', JSON.stringify(answers));

    Object.keys(fileData).forEach((fieldId) => (fileData[fieldId] || []).forEach((f) => f && dataToSend.append(`file_${fieldId}[]`, f)));
    Object.keys(signatureData).forEach((fieldId) =>
      (signatureData[fieldId] || []).forEach((f) => f && dataToSend.append(`file_${fieldId}[]`, f))
    );
    Object.keys(evidenceData).forEach((fieldId) =>
      (evidenceData[fieldId] || []).forEach((f) => f && dataToSend.append(`evidence_${fieldId}[]`, f))
    );

    if (geoField && userLocation.value) {
      dataToSend.append('location.lat', userLocation.value.lat);
      dataToSend.append('location.lng', userLocation.value.lng);
      if (geoAddress.value) {
        dataToSend.append('location.address', geoAddress.value);
      }
    }

    await axiosInstance.post(`/forms/${formId.value}/responses`, dataToSend, {
      headers: { 'Content-Type': 'multipart/form-data' }
    });

    toast.success('Formulario enviado correctamente');
    router.push('/mis-formularios');
  } catch (err) {
    console.error('Error al enviar el formulario:', err);
    if (err.response?.data?.message) toast.error(err.response.data.message);
    else toast.error('Error al enviar el formulario');
  } finally {
    submitting.value = false;
  }
};

const formLogo = computed(
  () => form.value?.logo_url || form.value?.logo || organization.value?.logo_url || organization.value?.logo || null
);

const expandedGroups = reactive({});
const toggleGroup = (groupId) => {
  expandedGroups[groupId] = !expandedGroups[groupId];
};

// --- INTEGRACIÓN: Ocultar campos geolocalización scope y mantener numeración ---
const orderedFieldsAndGroups = computed(() => {
  if (!form.value) return [];
  const fields = (form.value.fields || []).slice();
  const groups = (form.value.field_groups || []).slice();

  groups.sort((a, b) => (a.order ?? 0) - (b.order ?? 0));
  fields.sort((a, b) => (a.order ?? 0) - (b.order ?? 0));

  const groupMap = {};
  groups.forEach((g) => {
    groupMap[g.id] = { ...g, fields: [] };
  });

  // Filtrar campos geolocalización scope
  fields.forEach((f) => {
    if (f.type === 'geolocation' && f.attributes?.mode === 'scope') {
      // No agregar a ningún lado
      return;
    }
    if (f.form_field_group_id && groupMap[f.form_field_group_id]) {
      groupMap[f.form_field_group_id].fields.push(f);
    } else {
      groupMap['__ungrouped'] = groupMap['__ungrouped'] || { fields: [] };
      groupMap['__ungrouped'].fields.push(f);
    }
  });

  Object.values(groupMap).forEach((g) => {
    g.fields.sort((a, b) => (a.order ?? 0) - (b.order ?? 0));
  });

  const elements = [];
  // Campos sueltos (no agrupados)
  (groupMap['__ungrouped']?.fields || []).forEach((f) => {
    elements.push({ type: 'field', order: f.order ?? 0, field: f, key: `field-${f.id}` });
  });
  // Grupos
  groups.forEach((g) => {
    elements.push({ type: 'group', order: g.order ?? 0, group: groupMap[g.id], key: `group-${g.id}` });
    if (expandedGroups[g.id] === undefined) expandedGroups[g.id] = false;
  });

  elements.sort((a, b) => (a.order ?? 0) - (b.order ?? 0));

  return elements;
});
// --- FIN INTEGRACIÓN ---
</script>

<style scoped src="@/styles/answer_form.css"></style>
