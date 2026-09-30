<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_disk_mover.title') }}</h2>

    <v-alert type="warning" variant="tonal" class="mb-4">
      {{ $t('plugin_disk_mover.safe_notice') }}
    </v-alert>

    <v-card class="mb-4 pa-0">
      <v-card-title class="d-flex align-center">
        <v-icon class="mr-2">mdi-swap-horizontal-bold</v-icon>
        <span>{{ $t('plugin_disk_mover.status') }}</span>
      </v-card-title>
      <v-card-text>
        <v-chip :color="status.running ? 'warning' : 'success'" variant="tonal">
          {{ status.running ? $t('plugin_disk_mover.running') : $t('plugin_disk_mover.idle') }}
        </v-chip>
      </v-card-text>
    </v-card>

    <v-card class="mb-4 pa-0">
      <v-card-title>Physical disks</v-card-title>
      <v-card-text>
        <v-select
          v-model="settings.last_source"
          :items="diskItems"
          item-title="title"
          item-value="value"
          :label="$t('plugin_disk_mover.source')"
          class="mb-2"
          @update:model-value="onSourceDiskChanged"
        />

        <v-text-field
          :model-value="displayFolder(settings.source_folder)"
          :label="$t('plugin_disk_mover.source_folder')"
          :hint="$t('plugin_disk_mover.folder_root_hint')"
          persistent-hint
          readonly
          class="mb-4"
          :disabled="!settings.last_source"
          @click="openFolderDialog('source')"
        >
          <template #append-inner>
            <v-btn
              size="small"
              icon="mdi-folder"
              variant="text"
              :disabled="!settings.last_source"
              @click.stop="openFolderDialog('source')"
            />
          </template>
        </v-text-field>

        <v-select
          v-model="settings.last_destination"
          :items="diskItems"
          item-title="title"
          item-value="value"
          :label="$t('plugin_disk_mover.destination')"
          class="mb-2"
          @update:model-value="onDestinationDiskChanged"
        />

        <v-text-field
          :model-value="displayFolder(settings.destination_base)"
          :label="$t('plugin_disk_mover.destination_base')"
          :hint="$t('plugin_disk_mover.destination_base_hint')"
          persistent-hint
          readonly
          class="mb-4"
          :disabled="!settings.last_destination"
          @click="openFolderDialog('destination')"
        >
          <template #append-inner>
            <v-btn
              size="small"
              icon="mdi-folder"
              variant="text"
              :disabled="!settings.last_destination"
              @click.stop="openFolderDialog('destination')"
            />
          </template>
        </v-text-field>

        <v-alert v-if="!loadingDisks && diskItems.length === 0" type="info" variant="tonal" class="mb-4">
          {{ $t('plugin_disk_mover.no_disks') }}
        </v-alert>

        <div class="d-flex ga-2 flex-wrap mb-4">
          <v-btn variant="tonal" color="secondary" :loading="loadingDisks" @click="fetchDisks">
            <v-icon start>mdi-refresh</v-icon>
            {{ $t('plugin_disk_mover.refresh') }}
          </v-btn>

          <v-btn
            variant="tonal"
            color="primary"
            :disabled="!settings.last_source || !settings.last_destination"
            :loading="validating"
            @click="validateSelection"
          >
            <v-icon start>mdi-shield-check</v-icon>
            {{ $t('plugin_disk_mover.validate') }}
          </v-btn>

          <v-btn
            color="secondary"
            :disabled="!validationOk"
            :loading="simulating"
            @click="simulateTransfer"
          >
            <v-icon start>mdi-flask-outline</v-icon>
            {{ $t('plugin_disk_mover.simulate') }}
          </v-btn>
        </div>

        <v-alert v-if="validationMessage || validationOk" :type="validationOk ? 'success' : 'error'" variant="tonal" class="mb-4">
          {{ validationOk ? $t('plugin_disk_mover.validation_ok') : validationMessage }}
        </v-alert>

        <v-card v-if="simulationResult" variant="tonal" class="mb-4">
          <v-card-title class="d-flex align-center">
            <v-icon class="mr-2">mdi-flask-outline</v-icon>
            {{ $t('plugin_disk_mover.simulation') }}
          </v-card-title>

          <v-card-text>
            <v-alert type="success" variant="tonal" class="mb-4">
              {{ $t('plugin_disk_mover.simulation_safe') }}
            </v-alert>

            <div class="text-body-2 mb-4">
              <div><strong>Source:</strong> {{ simulationResult.source_path }}</div>
              <div><strong>Destination base:</strong> {{ simulationResult.destination_base_path }}</div>
              <div><strong>{{ $t('plugin_disk_mover.resolved_destination') }}:</strong> {{ simulationResult.resolved_destination }}</div>
            </div>

            <div class="mb-4">
              <strong>{{ $t('plugin_disk_mover.directories_to_create') }}:</strong>
              <template v-if="simulationResult.directories_to_create?.length">
                <div
                  v-for="path in simulationResult.directories_to_create"
                  :key="path"
                  class="text-caption"
                >
                  {{ path }}
                </div>
              </template>
              <span v-else class="text-caption">{{ $t('plugin_disk_mover.none') }}</span>
            </div>

            <v-row>
              <v-col cols="12" md="4">
                <strong>{{ $t('plugin_disk_mover.files_total') }}:</strong>
                {{ formatNumber(simulationResult.files_total) }}
              </v-col>
              <v-col cols="12" md="4">
                <strong>{{ $t('plugin_disk_mover.files_created') }}:</strong>
                {{ formatNumber(simulationResult.files_created) }}
              </v-col>
              <v-col cols="12" md="4">
                <strong>{{ $t('plugin_disk_mover.files_transfer') }}:</strong>
                {{ formatNumber(simulationResult.files_to_transfer) }}
              </v-col>

              <v-col cols="12" md="4">
                <strong>{{ $t('plugin_disk_mover.total_size') }}:</strong>
                {{ formatBytes(simulationResult.total_size) }}
              </v-col>
              <v-col cols="12" md="4">
                <strong>{{ $t('plugin_disk_mover.transfer_size') }}:</strong>
                {{ formatBytes(simulationResult.transfer_size) }}
              </v-col>
              <v-col cols="12" md="4">
                <strong>{{ $t('plugin_disk_mover.destination_free') }}:</strong>
                {{ formatBytes(simulationResult.destination_free) }}
              </v-col>
            </v-row>

            <v-alert
              :type="simulationResult.enough_space ? 'success' : 'error'"
              variant="tonal"
              class="mt-4"
            >
              {{ simulationResult.enough_space
                ? $t('plugin_disk_mover.space_ok')
                : $t('plugin_disk_mover.space_low') }}
            </v-alert>

            <div class="text-caption mt-3">
              {{ $t('plugin_disk_mover.duration') }}:
              {{ simulationResult.duration_seconds }} {{ $t('plugin_disk_mover.seconds') }}
            </div>
          </v-card-text>
        </v-card>

        <v-alert v-if="simulationError" type="error" variant="tonal" class="mb-4">
          {{ simulationError }}
        </v-alert>

        <v-divider class="mb-4" />

        <v-switch v-model="settings.verify" :label="$t('plugin_disk_mover.verify')" inset color="green" />
        <v-switch v-model="settings.preserve_xattrs" :label="$t('plugin_disk_mover.preserve_xattrs')" inset color="green" />
        <v-switch v-model="settings.preserve_acl" :label="$t('plugin_disk_mover.preserve_acl')" inset color="green" />

        <div class="d-flex ga-2 flex-wrap">
          <v-btn color="secondary" variant="tonal" :loading="saving" @click="saveSettings">
            <v-icon start>mdi-content-save</v-icon>
            {{ $t('plugin_disk_mover.save') }}
          </v-btn>

          <v-btn color="primary" disabled>
            <v-icon start>mdi-content-copy</v-icon>
            {{ $t('plugin_disk_mover.copy') }}
          </v-btn>

          <v-btn color="warning" disabled>
            <v-icon start>mdi-folder-move</v-icon>
            {{ $t('plugin_disk_mover.move') }}
          </v-btn>
        </div>
      </v-card-text>
    </v-card>

    <v-dialog v-model="folderDialog" max-width="850">
      <v-card>
        <v-card-title class="d-flex align-center">
          <span>
            {{ folderMode === 'source'
              ? $t('plugin_disk_mover.select_source_folder')
              : $t('plugin_disk_mover.select_destination_base') }}
          </span>
          <v-spacer />
          <v-chip size="small" variant="tonal">
            {{ folderCurrentDisplay }}
          </v-chip>
        </v-card-title>

        <v-card-subtitle class="pb-0">
          <div class="d-flex align-center ga-2">
            <v-btn size="small" variant="text" icon="mdi-home" color="secondary" :disabled="folderLoading" @click="folderGoRoot" />
            <v-btn
              size="small"
              variant="text"
              icon="mdi-arrow-up"
              color="secondary"
              :disabled="!folderCurrentRelative || folderLoading"
              @click="folderNavigateUp"
            />
            <v-btn
              size="small"
              variant="text"
              icon="mdi-refresh"
              color="secondary"
              :disabled="folderLoading"
              @click="fetchFolders(folderCurrentRelative)"
            />
            <v-spacer />
            <v-progress-circular v-if="folderLoading" indeterminate size="20" color="secondary" />
          </div>
        </v-card-subtitle>
        <div
          v-if="folderBreadcrumbs.length"
          class="px-4 pt-2"
          style="overflow-x: auto; white-space: nowrap"
        >
          <div class="d-flex align-center ga-1">
            <v-btn
              size="x-small"
              variant="text"
              color="secondary"
              :disabled="folderLoading"
              @click="folderNavigateBreadcrumb('')"
            >
              /
            </v-btn>

            <template
              v-for="(crumb, index) in folderBreadcrumbs"
              :key="crumb.relative"
            >
              <v-icon size="14">mdi-chevron-right</v-icon>

              <v-btn
                size="x-small"
                variant="text"
                color="secondary"
                :disabled="folderLoading || index === folderBreadcrumbs.length - 1"
                @click="folderNavigateBreadcrumb(crumb.relative)"
              >
                {{ crumb.name }}
              </v-btn>
            </template>
          </div>
        </div>
        <v-card-text class="pt-2" style="min-height: 300px; max-height: 60vh; overflow-y: auto">
          <v-table density="compact">
            <thead>
              <tr>
                <th>{{ $t('plugin_disk_mover.name') }}</th>
                <th>{{ $t('plugin_disk_mover.path') }}</th>
                <th style="width: 60px" class="text-center">{{ $t('plugin_disk_mover.action') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="!folderLoading && folderItems.length === 0">
                <td colspan="3" class="text-center text-medium-emphasis">
                  {{ $t('plugin_disk_mover.no_folders') }}
                </td>
              </tr>

              <tr
                v-for="item in folderItems"
                :key="item.relative"
                class="cursor-pointer"
                @click.stop.prevent="folderNavigateInto(item)"
              >
                <td>
                  <div class="d-flex align-center ga-2">
                    <v-icon size="18">mdi-folder</v-icon>
                    <span>{{ item.name }}</span>
                  </div>
                </td>
                <td><span class="text-caption">{{ item.display }}</span></td>
                <td class="text-center">
                  <v-btn
                    size="small"
                    icon="mdi-folder-open"
                    variant="text"
                    :disabled="folderLoading"
                    @click.stop="folderNavigateInto(item)"
                  />
                </td>
              </tr>
            </tbody>
          </v-table>
        </v-card-text>

        <v-divider />

        <v-card-actions>
          <div class="text-caption">
            <strong>{{ $t('plugin_disk_mover.current_folder') }}:</strong>
            {{ folderCurrentDisplay }}
          </div>
          <v-spacer />
          <v-btn variant="text" @click="folderDialog = false">
            {{ $t('plugin_disk_mover.cancel') }}
          </v-btn>
          <v-btn color="primary" :disabled="folderLoading" @click="selectCurrentFolder">
            <v-icon start>mdi-check</v-icon>
            {{ $t('plugin_disk_mover.select_current') }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted, computed } from 'vue';

const PLUGIN_NAME = 'disk-mover';

const diskItems = ref([]);
const loadingDisks = ref(false);
const validating = ref(false);
const saving = ref(false);
const simulating = ref(false);
const validationMessage = ref('');
const validationOk = ref(false);
const simulationResult = ref(null);
const simulationError = ref('');

const folderDialog = ref(false);
const folderMode = ref('source');
const folderDisk = ref('');
const folderCurrentRelative = ref('');
const folderCurrentDisplay = ref('/');
const folderParentRelative = ref('');
const folderItems = ref([]);
const folderLoading = ref(false);

const folderBreadcrumbs = computed(() => {
  const relative = folderCurrentRelative.value || '';
  if (!relative) return [];

  const parts = relative.split('/').filter(Boolean);

  return parts.map((name, index) => ({
    name,
    relative: parts.slice(0, index + 1).join('/'),
  }));
});

const status = reactive({ running: false, state: 'idle', percent: 0 });

const settings = reactive({
  verify: true,
  preserve_xattrs: true,
  preserve_acl: true,
  last_source: '',
  source_folder: '',
  last_destination: '',
  destination_base: '',
});

let statusInterval = null;

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const pluginQuery = async (args, timeout = 10) => {
  const res = await fetch('/api/v1/mos/plugins/query', {
    method: 'POST',
    headers: {
      ...getAuthHeaders(),
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      command: PLUGIN_NAME,
      args,
      timeout,
      parse_json: true,
    }),
  });

  const data = await res.json().catch(() => ({}));

  if (!res.ok || data?.success === false) {
    const backendError =
      data?.output?.error ||
      data?.error ||
      `MOS plugin query failed: ${res.status}`;
    throw new Error(backendError);
  }

  return data;
};

const encodePathArg = (value) => {
  if (!value) return '';

  const bytes = new TextEncoder().encode(value);
  let hex = '';

  for (const byte of bytes) {
    hex += byte.toString(16).padStart(2, '0');
  }

  return `xhex_${hex}`;
};

const displayFolder = (relative) =>
  relative ? `/${relative}` : '/';

const formatBytes = (value) => {
  const bytes = Number(value || 0);
  if (!Number.isFinite(bytes) || bytes <= 0) return '0 B';

  const units = ['B', 'KB', 'MB', 'GB', 'TB', 'PB'];
  const index = Math.min(
    Math.floor(Math.log(bytes) / Math.log(1024)),
    units.length - 1
  );
  const amount = bytes / Math.pow(1024, index);

  return `${amount.toLocaleString(undefined, {
    maximumFractionDigits: index === 0 ? 0 : 2,
  })} ${units[index]}`;
};

const formatNumber = (value) =>
  Number(value || 0).toLocaleString();

const clearValidation = () => {
  validationOk.value = false;
  validationMessage.value = '';
  simulationResult.value = null;
  simulationError.value = '';
};

const onSourceDiskChanged = () => {
  settings.source_folder = '';
  clearValidation();
};

const onDestinationDiskChanged = () => {
  settings.destination_base = '';
  clearValidation();
};

const fetchDisks = async () => {
  loadingDisks.value = true;
  clearValidation();
  try {
    const data = await pluginQuery(['disks']);
    diskItems.value = Array.isArray(data?.output?.items) ? data.output.items : [];
  } catch (e) {
    console.error('Failed to discover disks:', e);
    diskItems.value = [];
  } finally {
    loadingDisks.value = false;
  }
};

const openFolderDialog = async (mode) => {
  const disk = mode === 'source'
    ? settings.last_source
    : settings.last_destination;

  if (!disk) return;

  folderMode.value = mode;
  folderDisk.value = disk;
  folderCurrentRelative.value = mode === 'source'
    ? settings.source_folder
    : settings.destination_base;

  folderDialog.value = true;
  await fetchFolders(folderCurrentRelative.value);
};

const fetchFolders = async (relative = '') => {
  if (!folderDisk.value) return;

  folderLoading.value = true;
  try {
    const data = await pluginQuery([
      'folders',
      encodePathArg(folderDisk.value),
      encodePathArg(relative || ''),
    ], 30);

    const output = data?.output || {};
    folderCurrentRelative.value = output.current_relative || '';
    folderCurrentDisplay.value = output.current_display || '/';
    folderParentRelative.value = output.parent_relative || '';
    folderItems.value = Array.isArray(output.items) ? output.items : [];
  } catch (e) {
    console.error('Failed to browse folders:', e);
    folderItems.value = [];
  } finally {
    folderLoading.value = false;
  }
};

const folderNavigateInto = (item) => {
  if (!item?.relative) return;
  fetchFolders(item.relative);
};

const folderNavigateUp = () => {
  fetchFolders(folderParentRelative.value || '');
};

const folderGoRoot = () => {
  fetchFolders('');
};

const folderNavigateBreadcrumb = (relative) => {
  fetchFolders(relative || '');
};

const selectCurrentFolder = () => {
  if (folderMode.value === 'source') {
    settings.source_folder = folderCurrentRelative.value || '';
  } else {
    settings.destination_base = folderCurrentRelative.value || '';
  }

  folderDialog.value = false;
  clearValidation();
};

const validateSelection = async () => {
  validating.value = true;
  validationMessage.value = '';
  validationOk.value = false;
  simulationResult.value = null;
  simulationError.value = '';

  try {
    const data = await pluginQuery([
      'validate',
      encodePathArg(settings.last_source),
      encodePathArg(settings.source_folder || ''),
      encodePathArg(settings.last_destination),
      encodePathArg(settings.destination_base || ''),
    ], 30);

    validationOk.value = data?.output?.valid === true;

    if (!validationOk.value) {
      validationMessage.value =
        data?.output?.error || 'Selection is not valid.';
    }
  } catch (e) {
    validationMessage.value =
      e.message || 'Selection is not valid.';
  } finally {
    validating.value = false;
  }
};

const simulateTransfer = async () => {
  if (!validationOk.value) return;

  simulating.value = true;
  simulationResult.value = null;
  simulationError.value = '';

  try {
    const data = await pluginQuery([
      'dry-run',
      encodePathArg(settings.last_source),
      encodePathArg(settings.source_folder || ''),
      encodePathArg(settings.last_destination),
      encodePathArg(settings.destination_base || ''),
      settings.preserve_xattrs ? '1' : '0',
      settings.preserve_acl ? '1' : '0',
    ], 1800);

    if (data?.output?.success === true && data?.output?.dry_run === true) {
      simulationResult.value = data.output;
    } else {
      simulationError.value =
        data?.output?.error || 'Transfer simulation failed.';
    }
  } catch (e) {
    simulationError.value =
      e.message || 'Transfer simulation failed.';
  } finally {
    simulating.value = false;
  }
};

const fetchSettings = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });

    if (!res.ok) return;
    const data = await res.json();

    if (data.verify !== undefined) settings.verify = data.verify;
    if (data.preserve_xattrs !== undefined) settings.preserve_xattrs = data.preserve_xattrs;
    if (data.preserve_acl !== undefined) settings.preserve_acl = data.preserve_acl;
    if (data.last_source !== undefined) settings.last_source = data.last_source;
    if (data.source_folder !== undefined) settings.source_folder = data.source_folder;
    if (data.last_destination !== undefined) settings.last_destination = data.last_destination;
    if (data.destination_base !== undefined) settings.destination_base = data.destination_base;
  } catch (e) {
    console.error('Failed to load settings:', e);
  }
};

const saveSettings = async () => {
  saving.value = true;
  try {
    await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(settings),
    });
  } catch (e) {
    console.error('Failed to save settings:', e);
  } finally {
    saving.value = false;
  }
};

const checkStatus = async () => {
  try {
    const data = await pluginQuery(['status'], 5);
    if (data?.output) Object.assign(status, data.output);
  } catch (e) {
    status.running = false;
    status.state = 'error';
  }
};

onMounted(async () => {
  await fetchSettings();
  await fetchDisks();
  await checkStatus();
  statusInterval = setInterval(checkStatus, 5000);
});

onUnmounted(() => {
  if (statusInterval) clearInterval(statusInterval);
});
</script>
