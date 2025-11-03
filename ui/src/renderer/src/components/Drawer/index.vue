<template>
  <n-drawer
    v-model:show="props.active"
    placement="right"
    to="#drawer-target"
    :width="350"
    :show-mask="false"
    :trap-focus="false"
    :block-scroll="false"
    :mask-closable="false"
    class="!rounded-[unset] bg-primary border-l-primary border-l"
  >
    <n-drawer-content footer-style="border: unset;" body-content-style="padding: 15px 5px 40px">
      <template #header>
        <n-flex align="center" justify="space-between">
          <n-text depth="1">{{ t('Setting.Default') }}</n-text>
          <n-flex>
            <n-icon
              size="20"
              :component="ArrowBarRight"
              class="icon-hover rounded-[5px]"
              @click="closeDrawer"
            />
          </n-flex>
        </n-flex>
      </template>

      <n-scrollbar>
        <template #default>
          <n-flex class="mx-[15px]">
            <n-card
              v-for="item of currentOption"
              :key="item"
              :bordered="false"
              size="small"
              header-style="font-size: 13px;"
              class="rounded-[10px] !bg-secondary"
            >
              <n-collapse :trigger-areas="['extra']" :default-expanded-names="enabledItems">
                <n-collapse-item :name="item.display_name" :title="item.display_name">
                  <template #header-extra>
                    <n-switch
                      v-model:value="item.is_set"
                      size="small"
                      @update:value="handleChangeSwitchValue(item)"
                    >
                      <template #checked>{{ t('Setting.Enabled') }}</template>
                      <template #unchecked> {{ t('Setting.NotEnabled') }}</template>
                    </n-switch>
                  </template>
                  <n-form :model="item">
                    <n-form-item
                      :label="t('Setting.ApplicationPath')"
                      label-style="font-size: 13px"
                    >
                      <n-input-group>
                        <n-input
                          v-model:value="item.path"
                          size="small"
                          :disabled="item.is_internal || platform === 'macos'"
                        />
                        <input
                          :id="item.name"
                          type="file"
                          name="filename"
                          style="display: none"
                          @change="changeFile(item)"
                        />
                        <n-button
                          type="primary"
                          ghost
                          size="small"
                          :disabled="item.is_internal || platform === 'macos'"
                          @click="openFile(item)"
                        >
                          <n-icon :component="Folder28Regular" size="14" />
                        </n-button>
                      </n-input-group>
                    </n-form-item>
                    <n-form-item :label="t('Setting.Protocol')" label-style="font-size: 13px">
                      <n-select
                        v-model:value="item.match_first"
                        multiple
                        :options="
                          item.protocol.map((value: any) => ({ label: value, value: value }))
                        "
                        @update:value="handleItemChange(item)"
                      />
                    </n-form-item>
                    <n-form-item
                      :label="t('Setting.Comment')"
                      label-style="font-size: 13px"
                      label-placement="left"
                    >
                      <n-popover>
                        <template #trigger>
                          <span class="truncate">
                            {{ item.comment.en ? item.comment.en : t('Setting.NoDescription') }}
                          </span>
                        </template>
                        {{ item.comment.en ? item.comment.en : t('Setting.NoDescription') }}
                      </n-popover>
                    </n-form-item>
                    <n-form-item
                      :label="t('Setting.DownloadUrl')"
                      label-style="font-size: 13px"
                      feedback-style="min-height: unset"
                      label-placement="left"
                    >
                      <n-popover>
                        <template #trigger>
                          <n-a
                            href="#"
                            class="truncate"
                            @click.prevent="copyToClipboard(item.download_url)"
                          >
                            {{ item.download_url ? item.download_url : t('Common.None') }}
                          </n-a>
                        </template>
                        {{ item.download_url ? item.download_url : t('Common.None') }}
                      </n-popover>
                    </n-form-item>
                  </n-form>
                </n-collapse-item>
              </n-collapse>
            </n-card>

            <!-- Add Custom RDP Client Button (Windows only) -->
            <n-card
              v-if="route.name === 'Windows'"
              :bordered="false"
              size="small"
              header-style="font-size: 13px;"
              class="rounded-[10px] !bg-secondary"
            >
              <n-button
                type="primary"
                ghost
                block
                size="small"
                @click="showAddCustomClientModal = true"
              >
                <template #icon>
                  <n-icon :component="Add20Regular" size="16" />
                </template>
                {{ t('Setting.AddCustomRDPClient') }}
              </n-button>
            </n-card>

            <n-card
              :bordered="false"
              size="small"
              header-style="font-size: 13px;"
              class="rounded-[10px] !bg-secondary"
            >
              <n-collapse :trigger-areas="['main']">
                <n-collapse-item :title="t('Setting.Advanced')" name="advanced">
                  <n-form>
                    <n-form-item :label="t('Setting.Charset')" label-style="font-size: 13px">
                      <n-select v-model:value="charset" size="small" :options="charsetOptions" />
                    </n-form-item>
                    <n-form-item
                      :label="t('Setting.BackspaceAsCtrlH')"
                      label-style="font-size: 13px"
                    >
                      <n-select
                        v-model:value="is_backspace_as_ctrl_h"
                        size="small"
                        :options="boolOptions"
                      />
                    </n-form-item>
                    <n-form-item :label="t('Setting.Resolution')" label-style="font-size: 13px">
                      <n-select
                        v-model:value="rdp_resolution"
                        size="small"
                        :options="resolutionsOptions"
                      />
                    </n-form-item>
                  </n-form>
                </n-collapse-item>
              </n-collapse>
            </n-card>
          </n-flex>
        </template>
      </n-scrollbar>
    </n-drawer-content>
  </n-drawer>

  <!-- Add Custom RDP Client Modal -->
  <n-modal v-model:show="showAddCustomClientModal" preset="card" title="Add Custom RDP Client" style="width: 500px">
    <n-form :model="newClient" label-placement="left" label-width="120px">
      <n-form-item label="Display Name" required>
        <n-input v-model:value="newClient.display_name" placeholder="e.g., My Custom RDP Client" />
      </n-form-item>
      <n-form-item label="Application Path" required>
        <n-input-group>
          <n-input v-model:value="newClient.path" placeholder="Full path to .exe file" />
          <input
            id="custom-rdp-client-file"
            type="file"
            name="filename"
            accept=".exe"
            style="display: none"
            @change="changeCustomClientFile"
          />
          <n-button type="primary" ghost @click="openCustomClientFile">
            <n-icon :component="Folder28Regular" size="14" />
          </n-button>
        </n-input-group>
      </n-form-item>
      <n-form-item label="Argument Format">
        <n-input
          v-model:value="newClient.arg_format"
          placeholder="e.g., {file} or /v:{host}:{port}"
        />
        <template #feedback>
          <n-text depth="3" style="font-size: 11px">
            Available variables: {file}, {host}, {port}, {username}, {value}
          </n-text>
        </template>
      </n-form-item>
      <n-form-item label="Comment">
        <n-input v-model:value="newClient.comment.en" type="textarea" :rows="2" />
      </n-form-item>
    </n-form>
    <template #footer>
      <n-space justify="end">
        <n-button @click="showAddCustomClientModal = false">Cancel</n-button>
        <n-button type="primary" @click="addCustomRDPClient">Add Client</n-button>
      </n-space>
    </template>
  </n-modal>
</template>

<script setup lang="ts">
import mittBus from '@renderer/eventBus';
import { Folder28Regular, Add20Regular } from '@vicons/fluent';
import { ArrowBarRight } from '@vicons/tabler';

import { computed, nextTick, onBeforeUnmount, onMounted, Ref, ref, toRaw, watch } from 'vue';
import { useMessage } from 'naive-ui';
import { useRoute } from 'vue-router';
import { useSettingStore } from '@renderer/store/module/settingStore';

import { Conf } from 'electron-conf/renderer';
import {
  boolOptions,
  charsetOptions,
  databaseOptions,
  IClient,
  linuxOptions,
  resolutionsOptions,
  windowsOptions
} from './config/index';
import { useI18n } from 'vue-i18n';

const props = withDefaults(defineProps<{ active: boolean }>(), {
  active: false
});

const { t } = useI18n();
const route = useRoute();
const message = useMessage();
const settingStore = useSettingStore();

const charset = ref(settingStore.charset);
const rdp_resolution = ref(settingStore.rdp_resolution);
const is_backspace_as_ctrl_h = ref(settingStore.is_backspace_as_ctrl_h);

const currentOption: Ref<IClient[] | null> = ref(null);

const platform = ref('');

const conf = new Conf();

// Custom RDP Client Modal State
const showAddCustomClientModal = ref(false);
const newClient = ref<Partial<IClient>>({
  name: '',
  display_name: '',
  path: '',
  protocol: ['rdp'],
  arg_format: '{file}',
  type: 'remotedesktop',
  match_first: ['rdp'],
  is_internal: false,
  is_default: false,
  is_set: false,
  comment: { en: '', zh: '' },
  download_url: ''
});

const updateCurrentOptions = (newValue: string | null) => {
  switch (newValue) {
    case 'Linux':
      currentOption.value = linuxOptions.value;
      break;
    case 'Windows':
      currentOption.value = windowsOptions.value;
      break;
    case 'Database':
      currentOption.value = databaseOptions.value;
      break;
    default:
      currentOption.value = [];
  }
};

const enabledItems = computed(() => {
  // 获取所有启用状态的 item 标签作为展开的名称
  return currentOption.value
    ? currentOption.value.filter(item => item.is_set).map(item => item.display_name)
    : [];
});

const handleItemChange = async (item: IClient) => {
  // Rebuild the specific list for the item's type
  const updatedList =
    currentOption.value
      ?.filter(option => option.type === item.type)
      .map(option => {
        if (option.name !== item.name) {
          option.match_first = option.match_first.filter(i => !item.match_first.includes(i));
        }
        return toRaw(option);
      }) || [];

  // Persist the entire platform block to keep reads/writes consistent
  const platformData = (await conf.get(platform.value)) || {};
  platformData[item.type] = updatedList;
  await conf.set(platform.value, platformData);
};

const openFile = (item: IClient) => {
  window.document.getElementById(item.name)!.click();
};

const changeFile = (item: IClient) => {
  const fileInput = window.document.getElementById(item.name) as HTMLInputElement;
  item.path = fileInput?.files?.[0]?.path || '';
  handleItemChange(item).then(() => {
    message.success(`${t('Message.ChangeSuccess')}`);
  });
};

const copyToClipboard = (text: string) => {
  navigator.clipboard
    .writeText(text)
    .then(() => message.success(`${t('Message.CopyToClipboard')}`))
    .catch();
};

const openCustomClientFile = () => {
  window.document.getElementById('custom-rdp-client-file')!.click();
};

const changeCustomClientFile = () => {
  const fileInput = window.document.getElementById('custom-rdp-client-file') as HTMLInputElement;
  newClient.value.path = fileInput?.files?.[0]?.path || '';
};

const addCustomRDPClient = async () => {
  try {
    if (!newClient.value.display_name || !newClient.value.path) {
      message.warning('Please fill in Display Name and Application Path');
      return;
    }

    const platformKey = platform.value || 'Windows';

    // Generate unique name from display_name
    const name = newClient.value.display_name
      .toLowerCase()
      .replace(/[^a-z0-9]/g, '_')
      .replace(/_+/g, '_')
      .replace(/^_|_$/g, '');
    newClient.value.name = `custom_${name}_${Date.now()}`;

    // Load whole platform block and current remotedesktop list
    const platformData = (await conf.get(platformKey)) || {};
    const currentList = platformData.remotedesktop || [];

    // Add new client to the list
    const clientToAdd: IClient = {
      ...newClient.value,
      name: newClient.value.name,
      display_name: newClient.value.display_name,
      path: newClient.value.path,
      protocol: newClient.value.protocol || ['rdp'],
      arg_format: newClient.value.arg_format || '{file}',
      type: 'remotedesktop',
      match_first: newClient.value.match_first || ['rdp'],
      is_internal: false,
      is_default: false,
      is_set: true,
      comment: newClient.value.comment || { en: '', zh: '' },
      download_url: newClient.value.download_url || ''
    } as IClient;

    currentList.push(clientToAdd);

    // Save updated list back into the platform block and persist
    platformData.remotedesktop = currentList;
    await conf.set(platformKey, platformData);

    // Refresh the UI
    windowsOptions.value = currentList;
    currentOption.value = windowsOptions.value;

    // Reset form and close modal
    newClient.value = {
      name: '',
      display_name: '',
      path: '',
      protocol: ['rdp'],
      arg_format: '{file}',
      type: 'remotedesktop',
      match_first: ['rdp'],
      is_internal: false,
      is_default: false,
      is_set: false,
      comment: { en: '', zh: '' },
      download_url: ''
    };
    showAddCustomClientModal.value = false;

    message.success('Custom RDP client added successfully');
  } catch (e) {
    console.error(e);
    message.error('Failed to add client. Please try again.');
  }
};

/**
 * @description 关闭抽屉
 */
const closeDrawer = () => {
  mittBus.emit('createDrawer');
};

/**
 * @deprecated 当只有一个处于启用状态时不允许关闭当前 Switch，只有点击启用其他 Option 时该状态关闭
 *
 * @param item
 */
const handleChangeSwitchValue = (item: IClient) => {
  nextTick(() => {
    const enabledOptions = currentOption.value?.filter(option => option.is_set);
    if (enabledOptions && enabledOptions.length === 0) {
      message.warning(`${t('Message.EnableOneOption')}`);
      item.is_set = true;
      return;
    }
    handleItemChange(item).then(() => {
      message.success(`${t('Message.ChangeSuccess')}`);
    });
  });
};

const initPlatformData = async () => {
  const platformData = await conf.get(platform.value);
  if (platformData) {
    linuxOptions.value = [...platformData.terminal, ...platformData.filetransfer];
    windowsOptions.value = platformData.remotedesktop;
    databaseOptions.value = platformData.databases;
  }
};

const checkMatch = async (protocol: string) => {
  const platformData = await conf.get(platform.value);
  const clients = [
    ...platformData.terminal,
    ...platformData.filetransfer,
    ...platformData.remotedesktop,
    ...platformData.databases
  ];
  const enabledOptions = clients.filter(
    option => option.is_set && option.match_first.includes(protocol)
  );
  if (enabledOptions && enabledOptions.length === 0) {
    message.warning(`${t('Message.NotMatched')}`);
  } else {
    message.success(`${t('Message.ConnectSuccess')}`);
  }
};

watch(
  () => route.name,
  newValue => {
    if (newValue && typeof newValue === 'string') {
      updateCurrentOptions(newValue);
    }
  },
  { immediate: true }
);

watch([charset, is_backspace_as_ctrl_h, rdp_resolution], () => {
  settingStore.charset = charset.value;
  settingStore.is_backspace_as_ctrl_h = is_backspace_as_ctrl_h.value;
  settingStore.rdp_resolution = rdp_resolution.value;
});

onMounted(() => {
  window.electron.ipcRenderer.send('get-platform');
  window.electron.ipcRenderer.on('platform-response', (_event, _platform) => {
    platform.value = _platform;
    initPlatformData();
  });
  mittBus.on('checkMatch', checkMatch);
});
onBeforeUnmount(() => {
  mittBus.off('checkMatch', checkMatch);
});
</script>

<style lang="scss" scoped>
:deep(.n-form-item-feedback-wrapper) {
  min-height: 15px;
}

:deep(.n-collapse-item-arrow) {
  display: none !important;
}
</style>
