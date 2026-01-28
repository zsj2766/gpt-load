<script setup lang="ts">
import type { Group } from "@/types/models";
import { getGroupDisplayName } from "@/utils/display";
import { Add, LinkOutline, Search } from "@vicons/ionicons5";
import { NButton, NCard, NEmpty, NInput, NSpin, NTag } from "naive-ui";
import { computed, ref, watch } from "vue";
import { useI18n } from "vue-i18n";
import AggregateGroupModal from "./AggregateGroupModal.vue";
import GroupFormModal from "./GroupFormModal.vue";

const { t } = useI18n();

interface Props {
  groups: Group[];
  selectedGroup: Group | null;
  loading?: boolean;
}

interface Emits {
  (e: "group-select", group: Group): void;
  (e: "refresh"): void;
  (e: "refresh-and-select", groupId: number): void;
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
});

const emit = defineEmits<Emits>();

const searchText = ref("");
const showGroupModal = ref(false);
// 存储分组项 DOM 元素的引用
const groupItemRefs = ref(new Map());
const showAggregateGroupModal = ref(false);

const filteredGroups = computed(() => {
  if (!searchText.value.trim()) {
    return props.groups;
  }
  const search = searchText.value.toLowerCase().trim();
  return props.groups.filter(
    group =>
      group.name.toLowerCase().includes(search) ||
      group.display_name?.toLowerCase().includes(search)
  );
});

const isSearching = computed(() => Boolean(searchText.value.trim()));

const channelConfigs = [
  { key: "openai", title: "OpenAI" },
  { key: "anthropic", title: "Claude" },
  { key: "gemini", title: "Gemini" },
] as const;

type ChannelKey = (typeof channelConfigs)[number]["key"];
type ColumnKey = "standard" | "aggregate";

interface ColumnConfig {
  key: ColumnKey;
  title: string;
  createLabel: string;
  createType: "success" | "info";
  icon: typeof Add | typeof LinkOutline;
  onCreate: () => void;
}

const channelKeySet = new Set<ChannelKey>(channelConfigs.map(channel => channel.key));

const groupBuckets = computed(() => {
  const buckets: Record<ColumnKey, Record<ChannelKey, Group[]>> = {
    standard: { openai: [], anthropic: [], gemini: [] },
    aggregate: { openai: [], anthropic: [], gemini: [] },
  };

  for (const group of filteredGroups.value) {
    const columnKey: ColumnKey = group.group_type === "aggregate" ? "aggregate" : "standard";
    const channelKey: ChannelKey = channelKeySet.has(group.channel_type as ChannelKey)
      ? (group.channel_type as ChannelKey)
      : "openai";

    buckets[columnKey][channelKey].push(group);
  }

  return buckets;
});

// 监听选中项 ID 的变化，并自动滚动到该项
watch(
  () => props.selectedGroup?.id,
  id => {
    if (!id || props.groups.length === 0) {
      return;
    }

    const element = groupItemRefs.value.get(id);
    if (element) {
      element.scrollIntoView({
        behavior: "smooth", // 平滑滚动
        block: "nearest", // 将元素滚动到最近的边缘
      });
    }
  },
  {
    flush: "post", // 确保在 DOM 更新后执行回调
    immediate: true, // 立即执行一次以处理初始加载
  }
);

function handleGroupClick(group: Group) {
  emit("group-select", group);
}

// 获取渠道类型的标签颜色
function getChannelTagType(channelType: string) {
  switch (channelType) {
    case "openai":
      return "success";
    case "gemini":
      return "info";
    case "anthropic":
      return "warning";
    default:
      return "default";
  }
}

function getChannelLabel(channelType: string) {
  const match = channelConfigs.find(channel => channel.key === channelType);
  return match?.title || channelType;
}

function openCreateGroupModal() {
  showGroupModal.value = true;
}

function openCreateAggregateGroupModal() {
  showAggregateGroupModal.value = true;
}

function handleGroupCreated(group: Group) {
  showGroupModal.value = false;
  showAggregateGroupModal.value = false;
  if (group?.id) {
    emit("refresh-and-select", group.id);
  }
}

const columnConfigs = computed<ColumnConfig[]>(() => [
  {
    key: "aggregate",
    title: t("keys.aggregateGroup"),
    createLabel: t("keys.createAggregateGroup"),
    createType: "info",
    icon: LinkOutline,
    onCreate: openCreateAggregateGroupModal,
  },
  {
    key: "standard",
    title: t("keys.standardGroup"),
    createLabel: t("keys.createGroup"),
    createType: "success",
    icon: Add,
    onCreate: openCreateGroupModal,
  },
]);
</script>

<template>
  <div class="group-list-container">
    <n-card class="group-list-card modern-card" :bordered="false" size="small">
      <!-- 搜索框 -->
      <div class="search-section">
        <n-input
          v-model:value="searchText"
          :placeholder="t('keys.searchGroupPlaceholder')"
          size="small"
          clearable
        >
          <template #prefix>
            <n-icon :component="Search" />
          </template>
        </n-input>
      </div>

      <!-- 分组列表 -->
      <div class="groups-section">
        <n-spin :show="loading" size="small">
          <div class="groups-columns">
            <div v-for="column in columnConfigs" :key="column.key" class="group-column">
              <div class="column-header">{{ column.title }}</div>
              <div class="column-body">
                <div v-for="channel in channelConfigs" :key="channel.key" class="channel-section">
                  <div class="channel-title">{{ channel.title }}</div>
                  <div
                    v-if="
                      groupBuckets[column.key][channel.key].length === 0 &&
                      isSearching &&
                      !loading
                    "
                    class="empty-container channel-empty"
                  >
                    <n-empty size="small" :description="t('keys.noMatchingGroups')" />
                  </div>
                  <div
                    v-else-if="groupBuckets[column.key][channel.key].length > 0"
                    class="groups-list"
                  >
                    <div
                      v-for="group in groupBuckets[column.key][channel.key]"
                      :key="group.id"
                      class="group-item"
                      :class="{
                        active: selectedGroup?.id === group.id,
                        aggregate: group.group_type === 'aggregate',
                      }"
                      @click="handleGroupClick(group)"
                      :ref="
                        el => {
                          if (el) groupItemRefs.set(group.id, el);
                        }
                      "
                    >
                      <div class="group-icon">
                        <span v-if="group.group_type === 'aggregate'">🔗</span>
                        <span v-else-if="group.channel_type === 'openai'">🤖</span>
                        <span v-else-if="group.channel_type === 'gemini'">💎</span>
                        <span v-else-if="group.channel_type === 'anthropic'">🧠</span>
                        <span v-else>🔧</span>
                      </div>
                      <div class="group-content">
                        <div class="group-name">{{ getGroupDisplayName(group) }}</div>
                        <div class="group-meta">
                          <n-tag size="tiny" :type="getChannelTagType(group.channel_type)">
                            {{ getChannelLabel(group.channel_type) }}
                          </n-tag>
                          <n-tag v-if="group.group_type === 'aggregate'" size="tiny" type="warning" round>
                            {{ t("keys.aggregateGroup") }}
                          </n-tag>
                          <span v-if="group.group_type !== 'aggregate'" class="group-id">
                            #{{ group.name }}
                          </span>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
              </div>
              <div class="column-footer">
                <n-button
                  class="column-create-btn"
                  :type="column.createType"
                  size="small"
                  block
                  @click="column.onCreate"
                >
                  <template #icon>
                    <n-icon :component="column.icon" />
                  </template>
                  {{ column.createLabel }}
                </n-button>
              </div>
            </div>
          </div>
        </n-spin>
      </div>
    </n-card>
    <group-form-modal v-model:show="showGroupModal" @success="handleGroupCreated" />
    <aggregate-group-modal
      v-model:show="showAggregateGroupModal"
      :groups="groups"
      @success="handleGroupCreated"
    />
  </div>
</template>

<style scoped>
:deep(.n-card__content) {
  height: 100%;
}

.groups-section::-webkit-scrollbar {
  width: 1px;
  height: 1px;
}

.group-list-container {
  height: 100%;
}

.group-list-card {
  height: 100%;
  display: flex;
  flex-direction: column;
  background: var(--card-bg-solid);
}

.group-list-card:hover {
  transform: none;
  box-shadow: var(--shadow-lg);
}

.search-section {
  height: 41px;
}

.groups-section {
  flex: 1;
  height: calc(100% - 41px);
  overflow: auto;
}

.groups-columns {
  display: grid;
  grid-template-columns: 1fr;
  gap: 12px;
  padding-right: 4px;
}

.group-column {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.column-header {
  font-weight: 600;
  font-size: 13px;
  color: var(--text-secondary);
  padding: 4px 2px 0;
  flex-shrink: 0;
}

.column-body {
  display: flex;
  flex-direction: column;
  gap: 12px;
  flex: 1;
  overflow-y: auto;
  min-height: 0;
}

.column-footer {
  flex-shrink: 0;
  padding-top: 8px;
}

.column-create-btn {
  margin-top: 4px;
}

.channel-section {
  display: flex;
  flex-direction: column;
  gap: 6px;
  flex-shrink: 0;
}

.channel-title {
  font-size: 12px;
  font-weight: 600;
  color: var(--text-secondary);
}

.empty-container {
  padding: 12px 0;
}

.channel-empty :deep(.n-empty) {
  padding: 0;
}

.groups-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
  max-height: 100%;
  overflow-y: auto;
  width: 100%;
}

.group-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s ease;
  border: 1px solid var(--border-color);
  font-size: 12px;
  color: var(--text-primary);
  background: transparent;
  box-sizing: border-box;
  position: relative;
}

/* 聚合分组样式 */
.group-item.aggregate {
  border-style: dashed;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.02) 0%, rgba(102, 126, 234, 0.05) 100%);
}

:root.dark .group-item.aggregate {
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.05) 0%, rgba(102, 126, 234, 0.1) 100%);
  border-color: rgba(102, 126, 234, 0.2);
}

.group-item:hover,
.group-item.aggregate:hover {
  background: var(--bg-tertiary);
  border-color: var(--primary-color);
}

.group-item.aggregate:hover {
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.05) 0%, rgba(102, 126, 234, 0.1) 100%);
  border-style: dashed;
}

:root.dark .group-item:hover {
  background: rgba(102, 126, 234, 0.1);
  border-color: rgba(102, 126, 234, 0.3);
}

:root.dark .group-item.aggregate:hover {
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.1) 0%, rgba(102, 126, 234, 0.15) 100%);
  border-color: rgba(102, 126, 234, 0.4);
}

.group-item.aggregate.active {
  background: var(--primary-gradient);
  border-style: solid;
}

.group-item.active,
:root.dark .group-item.active,
:root.dark .group-item.aggregate.active {
  background: var(--primary-gradient);
  color: white;
  border-color: transparent;
  box-shadow: var(--shadow-md);
  border-style: solid;
}

.group-icon {
  font-size: 16px;
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--bg-secondary);
  border-radius: 6px;
  flex-shrink: 0;
  box-sizing: border-box;
}

.group-item.active .group-icon {
  background: rgba(255, 255, 255, 0.2);
}

.group-content {
  flex: 1;
  min-width: 0;
}

.group-name {
  font-weight: 600;
  font-size: 14px;
  line-height: 1.2;
  margin-bottom: 4px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.group-meta {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 10px;
  flex-wrap: wrap;
}

.group-id {
  opacity: 0.8;
  color: var(--text-secondary);
}

.group-item.active .group-id {
  opacity: 0.9;
  color: white;
}

/* 滚动条样式 */
.groups-list::-webkit-scrollbar {
  width: 4px;
}

.groups-list::-webkit-scrollbar-track {
  background: transparent;
}

.groups-list::-webkit-scrollbar-thumb {
  background: var(--scrollbar-bg);
  border-radius: 2px;
}

.groups-list::-webkit-scrollbar-thumb:hover {
  background: var(--border-color);
}

@media (min-width: 768px) {
  .groups-columns {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

/* 暗黑模式特殊样式 */
:root.dark .group-item {
  border-color: rgba(255, 255, 255, 0.05);
}

:root.dark .group-icon {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.08);
}

:root.dark .search-section :deep(.n-input) {
  --n-border: 1px solid rgba(255, 255, 255, 0.08);
  --n-border-hover: 1px solid rgba(102, 126, 234, 0.4);
  --n-border-focus: 1px solid var(--primary-color);
  background: rgba(255, 255, 255, 0.03);
}

/* 标签样式优化 */
:root.dark .group-meta :deep(.n-tag) {
  background: rgba(102, 126, 234, 0.15);
  border: 1px solid rgba(102, 126, 234, 0.3);
}

:root.dark .group-item.active .group-meta :deep(.n-tag) {
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.3);
}
</style>
