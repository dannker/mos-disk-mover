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
        />

        <v-select
          v-model="settings.last_destination"
          :items="diskItems"
          item-title="title"
          item-value="value"
          :label="$t('plugin_disk_mover.destination')"
          class="mb-2"
        />

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
        </div>

        <v-alert v-if="validationMessage" :type="validationOk ? 'success' : 'error'" variant="tonal" class="mb-4">
          {{ validationMessage }}
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
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from 'vue';

const PLUGIN_NAME = 'disk-mover';

const diskItems = ref([]);
const loadingDisks = ref(false);
const validating = ref(false);
const saving = ref(false);
const validationMessage = ref('');
const validationOk = ref(false);
const status = reactive({ running: false, state: 'idle', percent: 0 });
const settings = reactive({
  verify: true,
  preserve_xattrs: true,
  preserve_acl: true,
  last_source: '',
  last_destination: '',
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

  if (!res.ok) {
    throw new Error(`MOS plugin query failed: ${res.status}`);
  }

  return res.json();
};

const fetchDisks = async () => {
  loadingDisks.value = true;
  validationMessage.value = '';
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

const validateSelection = async () => {
  validating.value = true;
  validationMessage.value = '';
  validationOk.value = false;
  try {
    const data = await pluginQuery([
      'validate',
      settings.last_source,
      settings.last_destination,
    ]);

    validationOk.value = data?.output?.valid === true;
    validationMessage.value = validationOk.value
      ? 'Selection is valid.'
      : (data?.output?.error || 'Selection is not valid.');
  } catch (e) {
    validationMessage.value = e.message || 'Selection is not valid.';
  } finally {
    validating.value = false;
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
    if (data.last_destination !== undefined) settings.last_destination = data.last_destination;
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
