<script setup lang="ts">
import { keysApi } from "@/api/keys";
import type { Group, GroupModelsResponse, SubGroupInfo } from "@/types/models";
import { getGroupDisplayName } from "@/utils/display";
import { Add, Close } from "@vicons/ionicons5";
import {
  NButton,
  NCard,
  NForm,
  NFormItem,
  NIcon,
  NInput,
  NModal,
  NSelect,
  useMessage,
  type FormRules,
} from "naive-ui";
import { computed, reactive, ref, watch } from "vue";
import { useI18n } from "vue-i18n";

interface Props {
  show: boolean;
  aggregateGroup: Group | null;
  subGroups?: SubGroupInfo[];
}

interface Emits {
  (e: "update:show", value: boolean): void;
  (e: "success"): void;
}

interface MappingItem {
  model: string;
  target_group: string | null;
}

interface SubGroupAutoItem {
  regex: string;
  loading: boolean;
  lastError: string;
}

const props = withDefaults(defineProps<Props>(), {
  subGroups: () => [],
});
const emit = defineEmits<Emits>();

const { t } = useI18n();
const message = useMessage();
const loading = ref(false);
const formRef = ref();

const formData = reactive<{ mappings: MappingItem[] }>({
  mappings: [{ model: "", target_group: null }],
});

const subGroupAutoState = reactive<Record<number, SubGroupAutoItem>>({});

const enabledSubGroups = computed<SubGroupInfo[]>(() => {
  return props.subGroups.filter(
    subGroup => !!subGroup.group?.id && !!subGroup.group?.name && subGroup.weight > 0
  );
});

const sortedSubGroups = computed<SubGroupInfo[]>(() => {
  return [...enabledSubGroups.value].sort((a, b) => b.weight - a.weight);
});

const targetGroupOptions = computed(() => {
  return enabledSubGroups.value.map(subGroup => ({
    label: `${getGroupDisplayName(subGroup)} (#${subGroup.group.name})`,
    value: subGroup.group.name,
  }));
});

const enabledTargetGroupNameSet = computed(() => {
  return new Set(enabledSubGroups.value.map(subGroup => subGroup.group.name));
});

function hasValidTargetGroup(targetGroup: string | null): targetGroup is string {
  return typeof targetGroup === "string" && enabledTargetGroupNameSet.value.has(targetGroup);
}

const rules: FormRules = {
  mappings: {
    type: "array",
    required: true,
    validator: (_rule, value: MappingItem[]) => {
      const validItems = value.filter(item => item.model.trim() && item.target_group);
      if (validItems.length === 0) {
        return new Error(t("aggregateMappings.atLeastOneMapping"));
      }

      const modelSet = new Set<string>();
      for (const item of validItems) {
        const model = item.model.trim();
        if (!model) {
          return new Error(t("keys.modelRedirectEmptyModel"));
        }
        if (!item.target_group) {
          return new Error(t("aggregateMappings.selectTargetGroup"));
        }
        if (!hasValidTargetGroup(item.target_group)) {
          return new Error(t("aggregateMappings.selectTargetGroup"));
        }
        if (modelSet.has(model)) {
          return new Error(t("aggregateMappings.duplicateModel"));
        }
        modelSet.add(model);
      }

      return true;
    },
    trigger: ["blur", "change"],
  },
};

watch(
  () => props.show,
  show => {
    if (show) {
      loadMappings();
      resetAutoState();
      ensureAutoStateForSubGroups();
    }
  }
);

watch(
  sortedSubGroups,
  () => {
    ensureAutoStateForSubGroups();
  },
  { immediate: true }
);

function getSubGroupId(subGroup: SubGroupInfo): number {
  return subGroup.group.id as number;
}

function getSubGroupName(subGroup: SubGroupInfo): string {
  return getGroupDisplayName(subGroup) || subGroup.group.name;
}

function ensureAutoStateForSubGroups() {
  const currentIds = new Set<number>();

  sortedSubGroups.value.forEach(subGroup => {
    const subGroupId = getSubGroupId(subGroup);
    currentIds.add(subGroupId);

    if (!subGroupAutoState[subGroupId]) {
      subGroupAutoState[subGroupId] = {
        regex: "",
        loading: false,
        lastError: "",
      };
    }
  });

  Object.keys(subGroupAutoState).forEach(key => {
    const subGroupId = Number(key);
    if (!currentIds.has(subGroupId)) {
      delete subGroupAutoState[subGroupId];
    }
  });
}

function getSubGroupAutoState(subGroupId: number): SubGroupAutoItem {
  if (!subGroupAutoState[subGroupId]) {
    subGroupAutoState[subGroupId] = {
      regex: "",
      loading: false,
      lastError: "",
    };
  }
  return subGroupAutoState[subGroupId];
}

function resetAutoState() {
  Object.values(subGroupAutoState).forEach(state => {
    state.regex = "";
    state.loading = false;
    state.lastError = "";
  });
}

function loadMappings() {
  const rulesMap = props.aggregateGroup?.aggregate_model_rules || {};
  const entries = Object.entries(rulesMap).map(([model, target]) => ({
    model,
    target_group: target,
  }));
  formData.mappings = entries.length > 0 ? entries : [{ model: "", target_group: null }];
}

function addMappingItem() {
  formData.mappings.push({ model: "", target_group: null });
}

function removeMappingItem(index: number) {
  if (formData.mappings.length > 1) {
    formData.mappings.splice(index, 1);
  }
}

function handleClose() {
  emit("update:show", false);
}

function parseRegexInput(input: string): RegExp | null {
  const trimmed = input.trim();
  if (!trimmed) {
    return null;
  }

  const slashWrapped = trimmed.match(/^\/(.*)\/([a-z]*)$/i);
  if (slashWrapped) {
    return new RegExp(slashWrapped[1], slashWrapped[2]);
  }

  return new RegExp(trimmed);
}

function isRegexMatch(model: string, regex: RegExp | null): boolean {
  if (!regex) {
    return true;
  }
  regex.lastIndex = 0;
  return regex.test(model);
}

function pushModelName(items: string[], model: unknown) {
  if (typeof model !== "string") {
    return;
  }
  const trimmed = model.trim();
  if (trimmed) {
    items.push(trimmed);
  }
}

function extractModelNames(response: GroupModelsResponse): string[] {
  const items: string[] = [];

  if (Array.isArray(response?.data)) {
    response.data.forEach(item => {
      pushModelName(items, item?.id);
      if (typeof item?.id !== "string" || !item.id.trim()) {
        pushModelName(items, item?.name);
      }
    });
  }

  if (Array.isArray(response?.models)) {
    response.models.forEach(item => {
      pushModelName(items, item?.name);
      if (typeof item?.name !== "string" || !item.name.trim()) {
        pushModelName(items, item?.displayName);
      }
      if (
        (typeof item?.name !== "string" || !item.name.trim()) &&
        (typeof item?.displayName !== "string" || !item.displayName.trim())
      ) {
        pushModelName(items, item?.id);
      }
    });
  }

  return Array.from(new Set(items));
}

function clearPlaceholderRowIfNeeded() {
  const hasOnlyPlaceholder =
    formData.mappings.length === 1 &&
    !formData.mappings[0].model.trim() &&
    !formData.mappings[0].target_group;

  if (hasOnlyPlaceholder) {
    formData.mappings = [];
  }
}

async function handleAutoAdd(subGroup: SubGroupInfo) {
  const subGroupId = getSubGroupId(subGroup);
  const targetGroup = subGroup.group.name;
  const subGroupName = getSubGroupName(subGroup);
  const autoState = getSubGroupAutoState(subGroupId);

  if (autoState.loading || !targetGroup) {
    return;
  }

  let regex: RegExp | null = null;
  try {
    regex = parseRegexInput(autoState.regex);
  } catch {
    autoState.lastError = t("aggregateMappings.autoAddInvalidRegex", { name: subGroupName });
    message.error(autoState.lastError);
    return;
  }

  autoState.loading = true;
  autoState.lastError = "";

  try {
    const response = await keysApi.getGroupModels(subGroupId, 0);
    const normalizedModels = extractModelNames(response);
    const matchedModels = normalizedModels.filter(model => isRegexMatch(model, regex));

    if (matchedModels.length === 0) {
      message.info(t("aggregateMappings.autoAddNoMatch", { name: subGroupName }));
      return;
    }

    clearPlaceholderRowIfNeeded();

    const existingModels = new Set(
      formData.mappings.map(item => item.model.trim()).filter(model => model.length > 0)
    );

    let added = 0;
    let skipped = 0;

    matchedModels.forEach(model => {
      if (existingModels.has(model)) {
        skipped += 1;
        return;
      }

      formData.mappings.push({ model, target_group: targetGroup });
      existingModels.add(model);
      added += 1;
    });

    const summaryText = t("aggregateMappings.autoAddSummary", {
      name: subGroupName,
      matched: matchedModels.length,
      added,
      skipped,
    });

    if (added > 0) {
      message.success(summaryText);
    } else {
      message.info(summaryText);
    }
  } catch (error) {
    console.error(error);
    autoState.lastError = t("aggregateMappings.autoAddFetchFailed", { name: subGroupName });
    message.error(autoState.lastError);
  } finally {
    autoState.loading = false;
  }
}

async function handleSubmit() {
  if (loading.value || !props.aggregateGroup?.id) {
    return;
  }

  try {
    await formRef.value?.validate();

    const invalidTargetExists = formData.mappings.some(
      item => !!item.model.trim() && !!item.target_group && !hasValidTargetGroup(item.target_group)
    );
    if (invalidTargetExists) {
      message.error(t("aggregateMappings.selectTargetGroup"));
      return;
    }

    loading.value = true;

    const aggregateModelRules = Object.fromEntries(
      formData.mappings
        .filter(item => item.model.trim() && item.target_group)
        .map(item => [item.model.trim(), item.target_group as string])
    );

    await keysApi.updateGroup(props.aggregateGroup.id, {
      aggregate_model_rules: aggregateModelRules,
    });

    emit("success");
    handleClose();
  } finally {
    loading.value = false;
  }
}
</script>

<template>
  <n-modal :show="show" @update:show="handleClose" class="add-aggregate-mapping-modal">
    <n-card
      class="add-aggregate-mapping-card"
      :title="t('aggregateMappings.title')"
      :bordered="false"
      size="huge"
      role="dialog"
      aria-modal="true"
    >
      <template #header-extra>
        <n-button quaternary circle @click="handleClose">
          <template #icon>
            <n-icon :component="Close" />
          </template>
        </n-button>
      </template>

      <n-form
        ref="formRef"
        :model="formData"
        :rules="rules"
        label-placement="left"
        label-width="100px"
      >
        <div class="form-section">
          <h4 class="section-title">
            {{ t("aggregateMappings.configTitle") }}
            <span class="section-subtitle">
              ({{ t("keys.channelType") }}: {{ aggregateGroup?.channel_type?.toUpperCase() || "-" }})
            </span>
          </h4>

          <div class="mapping-layout">
            <div class="sub-groups-panel">
              <h5 class="panel-title">{{ t("aggregateMappings.subGroupsTitle") }}</h5>

              <div v-if="sortedSubGroups.length === 0" class="sub-groups-empty">
                {{ t("subGroups.noSubGroups") }}
              </div>

              <div v-else class="sub-groups-list">
                <div v-for="subGroup in sortedSubGroups" :key="subGroup.group.id" class="sub-group-item">
                  <div class="sub-group-header">
                    <div class="sub-group-name">{{ getSubGroupName(subGroup) }}</div>
                    <div class="sub-group-meta">#{{ subGroup.group.name }}</div>
                  </div>

                  <div class="sub-group-status-row">
                    <span class="sub-group-weight">{{ t("subGroups.weight") }}: {{ subGroup.weight }}</span>
                  </div>

                  <div class="sub-group-actions">
                    <n-input
                      v-model:value="getSubGroupAutoState(getSubGroupId(subGroup)).regex"
                      :placeholder="t('aggregateMappings.regexPlaceholder')"
                      :disabled="getSubGroupAutoState(getSubGroupId(subGroup)).loading"
                      clearable
                    />
                    <n-button
                      type="primary"
                      secondary
                      @click="handleAutoAdd(subGroup)"
                      :loading="getSubGroupAutoState(getSubGroupId(subGroup)).loading"
                    >
                      {{ t("aggregateMappings.autoAdd") }}
                    </n-button>
                  </div>

                  <div v-if="getSubGroupAutoState(getSubGroupId(subGroup)).lastError" class="sub-group-error">
                    {{ getSubGroupAutoState(getSubGroupId(subGroup)).lastError }}
                  </div>
                </div>
              </div>
            </div>

            <div class="mappings-panel">
              <h5 class="panel-title">{{ t("aggregateMappings.mappings") }}</h5>

              <div class="mappings-list">
                <div v-for="(item, index) in formData.mappings" :key="index" class="mapping-item">
                  <span class="item-label">{{ t("aggregateMappings.mapping") }} {{ index + 1 }}</span>

                  <n-form-item
                    class="item-model"
                    :path="`mappings[${index}].model`"
                    :show-feedback="false"
                  >
                    <n-input
                      v-model:value="item.model"
                      :placeholder="t('aggregateMappings.modelPlaceholder')"
                      clearable
                    />
                  </n-form-item>

                  <n-form-item
                    class="item-target"
                    :path="`mappings[${index}].target_group`"
                    :show-feedback="false"
                  >
                    <n-select
                      v-model:value="item.target_group"
                      :options="targetGroupOptions"
                      :placeholder="t('aggregateMappings.targetPlaceholder')"
                      clearable
                    />
                  </n-form-item>

                  <n-button
                    @click="removeMappingItem(index)"
                    type="error"
                    quaternary
                    circle
                    size="small"
                    class="item-delete"
                    :style="{ visibility: formData.mappings.length > 1 ? 'visible' : 'hidden' }"
                  >
                    <template #icon>
                      <n-icon :component="Close" />
                    </template>
                  </n-button>
                </div>
              </div>

              <div class="add-item-section">
                <n-button @click="addMappingItem" dashed style="width: 100%">
                  <template #icon>
                    <n-icon :component="Add" />
                  </template>
                  {{ t("aggregateMappings.addMapping") }}
                </n-button>
              </div>
            </div>
          </div>
        </div>
      </n-form>

      <template #footer>
        <div style="display: flex; justify-content: flex-end; gap: 12px">
          <n-button @click="handleClose">{{ t("common.cancel") }}</n-button>
          <n-button type="primary" @click="handleSubmit" :loading="loading">
            {{ t("common.confirm") }}
          </n-button>
        </div>
      </template>
    </n-card>
  </n-modal>
</template>

<style scoped>
.add-aggregate-mapping-modal {
  width: 1100px;
}

.form-section {
  margin-top: 0;
}

.section-title {
  font-size: 1rem;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 16px;
  padding-bottom: 8px;
  border-bottom: 1px solid var(--border-color);
}

.section-subtitle {
  font-size: 0.85rem;
  font-weight: 400;
  color: var(--text-secondary);
  margin-left: 8px;
}

.mapping-layout {
  display: grid;
  grid-template-columns: 340px minmax(0, 1fr);
  gap: 16px;
}

.sub-groups-panel,
.mappings-panel {
  border: 1px solid var(--border-color);
  border-radius: var(--border-radius-md);
  padding: 12px;
  background: var(--bg-secondary);
}

.panel-title {
  margin: 0 0 12px;
  font-size: 0.95rem;
  font-weight: 600;
  color: var(--text-primary);
}

.sub-groups-empty {
  text-align: center;
  color: var(--text-tertiary);
  padding: 20px 12px;
  border: 1px dashed var(--border-color);
  border-radius: var(--border-radius-sm);
  background: var(--card-bg-solid);
}

.sub-groups-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-height: 58vh;
  overflow-y: auto;
  padding-right: 4px;
}

.sub-group-item {
  padding: 10px;
  border-radius: var(--border-radius-sm);
  border: 1px solid var(--border-color);
  background: var(--card-bg-solid);
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.sub-group-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}

.sub-group-name {
  font-weight: 600;
  color: var(--text-primary);
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.sub-group-meta {
  font-size: 12px;
  font-family: "SFMono-Regular", Consolas, "Liberation Mono", Menlo, Courier, monospace;
  color: var(--primary-color);
  background: var(--primary-color-suppl);
  padding: 2px 6px;
  border-radius: 4px;
  white-space: nowrap;
}

.sub-group-status-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
  font-size: 12px;
}

.sub-group-weight {
  color: var(--text-secondary);
}

.sub-group-actions {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  gap: 8px;
  align-items: center;
}

.sub-group-error {
  color: var(--error-color);
  font-size: 12px;
}

.mappings-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 16px;
  max-height: 58vh;
  overflow-y: auto;
  padding-right: 4px;
}

.mapping-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px;
  background: var(--card-bg-solid);
  border: 1px solid var(--border-color);
  border-radius: var(--border-radius-md);
}

.item-label {
  flex-shrink: 0;
  min-width: 90px;
  font-weight: 500;
  color: var(--text-primary);
  font-size: 0.9rem;
}

.item-model {
  flex: 1;
  min-width: 200px;
}

.item-target {
  flex: 1;
  min-width: 220px;
}

.item-delete {
  flex-shrink: 0;
}

.add-item-section {
  margin-top: 8px;
}

@media (max-width: 1200px) {
  .add-aggregate-mapping-modal {
    width: 96vw;
  }
}

@media (max-width: 900px) {
  .mapping-layout {
    grid-template-columns: minmax(0, 1fr);
  }

  .sub-groups-list,
  .mappings-list {
    max-height: 42vh;
  }

  .mapping-item {
    flex-direction: column;
    align-items: stretch;
    gap: 8px;
  }

  .item-label {
    min-width: auto;
    text-align: center;
  }

  .item-model,
  .item-target {
    width: 100%;
    min-width: auto;
  }

  .item-delete {
    align-self: center;
  }

  .sub-group-actions {
    grid-template-columns: minmax(0, 1fr);
  }
}
</style>
