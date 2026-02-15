<script setup lang="ts">
import { keysApi } from "@/api/keys";
import type { Group, SubGroupInfo } from "@/types/models";
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
  type FormRules,
} from "naive-ui";
import { computed, reactive, ref, watch } from "vue";
import { useI18n } from "vue-i18n";

interface Props {
  show: boolean;
  aggregateGroup: Group | null;
  groups: Group[];
}

interface Emits {
  (e: "update:show", value: boolean): void;
  (e: "success"): void;
}

interface MappingItem {
  model: string;
  target_group: string | null;
}

const props = defineProps<Props>();
const emit = defineEmits<Emits>();

const { t } = useI18n();
const loading = ref(false);
const formRef = ref();

const formData = reactive<{ mappings: MappingItem[] }>({
  mappings: [{ model: "", target_group: null }],
});

const existingSubGroups = computed<SubGroupInfo[]>(() => {
  if (!props.aggregateGroup?.id) {
    return [];
  }
  const aggregateGroup = props.groups.find(group => group.id === props.aggregateGroup?.id);
  return aggregateGroup?.sub_groups || [];
});

const targetGroupOptions = computed(() => {
  if (!props.aggregateGroup) {
    return [];
  }
  const existingIds = new Set(
    existingSubGroups.value.map(subGroup => subGroup.group?.id).filter((id): id is number => !!id)
  );

  return props.groups
    .filter(group => {
      if (group.group_type === "aggregate") {
        return false;
      }
      if (group.channel_type !== props.aggregateGroup.channel_type) {
        return false;
      }
      if (!group.id || !group.name) {
        return false;
      }
      return existingIds.has(group.id);
    })
    .map(group => ({
      label: group.display_name || group.name,
      value: group.name,
    }));
});

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
    }
  }
);

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

async function handleSubmit() {
  if (loading.value || !props.aggregateGroup?.id) {
    return;
  }

  try {
    await formRef.value?.validate();
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
              ({{ t("keys.channelType") }}: {{ aggregateGroup.channel_type.toUpperCase() }})
            </span>
          </h4>

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
  width: 760px;
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

.mappings-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
  margin-bottom: 20px;
}

.mapping-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: var(--bg-secondary);
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
  margin-top: 16px;
}

@media (max-width: 900px) {
  .add-aggregate-mapping-modal {
    width: 94vw;
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
}
</style>
