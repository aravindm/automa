<template>
  <div class="container pt-8 pb-4">
    <div class="flex items-center mb-8">
      <input
        :value="collection.name"
        placeholder="Collection name"
        class="
          text-2xl
          hover:ring-2 hover:ring-accent
          font-semibold
          bg-transparent
        "
        @blur="updateCollection({ name: $event.target.value || 'Unnamed' })"
      />
      <div class="flex-grow"></div>
      <ui-button variant="accent" class="mr-4" @click="executeCollection">
        Execute
      </ui-button>
      <ui-button class="text-red-500" @click="deleteCollection">
        Delete
      </ui-button>
    </div>
    <div class="flex items-start">
      <div
        class="w-80 border-r sticky top-11 pr-6 mr-6 p-1 scroll overflow-auto"
        style="max-height: calc(100vh - 8rem)"
      >
        <ui-input
          v-model="state.query"
          placeholder="Search workflows"
          class="w-full space-x-1 mb-3"
          prepend-icon="riSearch2Line"
        />
        <ui-tabs v-model="state.sidebarTab" fill class="w-full mb-4">
          <ui-tab value="workflows">Workflows</ui-tab>
          <ui-tab value="blocks">Blocks</ui-tab>
        </ui-tabs>
        <draggable
          :list="state.sidebarTab === 'workflows' ? workflows : blocksArr"
          :group="{ name: 'collection', pull: 'clone', put: false }"
          :sort="false"
          item-key="id"
        >
          <template #item="{ element }">
            <ui-card
              v-bind="{
                title: element.description ? element.description : element.name,
              }"
              class="mb-2 cursor-move flex items-center"
            >
              <v-remixicon :name="element.icon" class="mr-2" />
              <p class="flex-1 text-overflow">{{ element.name }}</p>
            </ui-card>
          </template>
        </draggable>
      </div>
      <div class="flex-1 relative">
        <div class="flex items-center mb-4">
          <div class="px-1 inline-block rounded-lg bg-white">
            <ui-tabs
              v-model="state.activeTab"
              class="border-none h-full space-x-1"
            >
              <ui-tab value="flow">Flow</ui-tab>
              <ui-tab value="logs">Logs</ui-tab>
              <ui-tab value="running">
                Running
                <span
                  v-if="runningCollection.length > 0"
                  class="
                    ml-2
                    p-1
                    text-center
                    inline-block
                    text-xs
                    rounded-full
                    bg-black
                    text-white
                  "
                  style="min-width: 25px"
                >
                  {{ runningCollection.length }}
                </span>
              </ui-tab>
              <ui-tab value="options">Options</ui-tab>
            </ui-tabs>
          </div>
          <div class="flex-grow"></div>
          <ui-button
            v-tooltip="'Global data'"
            icon
            @click="state.showGlobalData = !state.showGlobalData"
          >
            <v-remixicon name="riDatabase2Line" />
          </ui-button>
        </div>
        <ui-tab-panels v-model="state.activeTab">
          <ui-tab-panel class="relative" value="flow">
            <div
              v-if="collection.flow.length === 0"
              class="
                border
                text-gray-600
                absolute
                top-0
                w-full
                z-0
                dark:text-gray-200
                rounded-lg
                border-dashed
                text-center
                p-4
              "
            >
              Drop a workflow or block in here
            </div>
            <draggable
              :model-value="collectionFlow"
              item-key="id"
              group="collection"
              style="min-height: 200px"
              @update:modelValue="updateCollectionFlow"
            >
              <template #item="{ element, index }">
                <ui-card
                  class="
                    group
                    flex
                    cursor-move
                    mb-2
                    items-center
                    relative
                    overflow-hidden
                  "
                >
                  <span
                    :class="[
                      element.type === 'block'
                        ? 'bg-yellow-200'
                        : 'bg-green-200',
                    ]"
                    class="absolute w-1 left-0 top-0 h-full"
                  ></span>
                  <v-remixicon :name="element.icon" class="mr-4" />
                  <p class="flex-1 text-overflow">{{ element.name }}</p>
                  <router-link
                    v-if="element.type !== 'block'"
                    :to="'/workflows/' + element.id"
                    title="Open workflow"
                    class="mr-4 group group-hover:visible invisible"
                  >
                    <v-remixicon name="riExternalLinkLine" />
                  </router-link>
                  <v-remixicon
                    name="riDeleteBin7Line"
                    class="cursor-pointer"
                    @click="deleteCollectionFlow(index)"
                  />
                </ui-card>
              </template>
            </draggable>
          </ui-tab-panel>
          <ui-tab-panel value="logs">
            <div v-if="logs.length === 0" class="text-center">
              <img
                src="@/assets/svg/files-and-folder.svg"
                class="mx-auto max-w-sm"
              />
              <p class="text-xl font-semibold">No data to show</p>
            </div>
            <shared-logs-table :logs="logs" class="w-full">
              <template #item-append="{ log }">
                <td class="text-right">
                  <v-remixicon
                    name="riDeleteBin7Line"
                    class="inline-block text-red-500 cursor-pointer"
                    @click="deleteLog(log.id)"
                  />
                </td>
              </template>
            </shared-logs-table>
          </ui-tab-panel>
          <ui-tab-panel value="running">
            <div v-if="runningCollection.length === 0" class="text-center">
              <img
                src="@/assets/svg/files-and-folder.svg"
                class="mx-auto max-w-sm"
              />
              <p class="text-xl font-semibold">No data to show</p>
            </div>
            <div class="grid grid-cols-2 gap-4">
              <shared-workflow-state
                v-for="item in runningCollection"
                :key="item.id"
                :data="item"
              />
            </div>
          </ui-tab-panel>
          <ui-tab-panel value="options">
            <ui-checkbox v-model="collectionOptions.atOnce">
              <p class="leading-tight">
                Execute all workflows in the collection at once
              </p>
              <p class="text-sm text-gray-600 leading-tight">
                Block not gonna executed when using this option
              </p>
            </ui-checkbox>
          </ui-tab-panel>
        </ui-tab-panels>
      </div>
    </div>
  </div>
  <ui-modal v-model="state.showGlobalData" content-class="max-w-xl">
    <template #header>Global data</template>
    <p class="inline-block">
      This will overwrite the global data of the workflow
    </p>
    <p class="float-right clear-both" title="Characters limit">
      {{ collection.globalData.length }}/{{ (1e4).toLocaleString() }}
    </p>
    <prism-editor
      :model-value="collection.globalData"
      :highlight="highlighter('json')"
      class="h-full scroll mt-2"
      style="height: calc(100vh - 10rem)"
      @update:modelValue="updateGlobalData"
    />
  </ui-modal>
</template>
<script setup>
import { computed, shallowReactive, onMounted, watch } from 'vue';
import { nanoid } from 'nanoid';
import { useStore } from 'vuex';
import { useRoute, useRouter } from 'vue-router';
import { PrismEditor } from 'vue-prism-editor';
import Draggable from 'vuedraggable';
import { highlighter } from '@/lib/prism';
import { useDialog } from '@/composable/dialog';
import { sendMessage } from '@/utils/message';
import Log from '@/models/log';
import Workflow from '@/models/workflow';
import Collection from '@/models/collection';
import SharedLogsTable from '@/components/newtab/shared/SharedLogsTable.vue';
import SharedWorkflowState from '@/components/newtab/shared/SharedWorkflowState.vue';

const blocks = {
  'export-result': {
    type: 'block',
    id: 'export-result',
    icon: 'riDownloadLine',
    name: 'Export result',
    description: 'Export the collection result as JSON',
    data: {
      type: 'json',
    },
  },
};
const blocksArr = Object.entries(blocks).map(([id, value]) => ({
  ...value,
  id,
}));

const store = useStore();
const route = useRoute();
const router = useRouter();
const dialog = useDialog();

const state = shallowReactive({
  query: '',
  activeTab: 'flow',
  showGlobalData: false,
  sidebarTab: 'workflows',
});
const collectionOptions = shallowReactive({
  atOnce: false,
});

const runningCollection = computed(() =>
  store.state.workflowState.filter(
    ({ collectionId }) => collectionId === route.params.id
  )
);
const logs = computed(() =>
  Log.query()
    .where(
      ({ collectionId, isInCollection }) =>
        collectionId === route.params.id && !isInCollection
    )
    .orderBy('startedAt', 'desc')
    .limit(10)
    .get()
);
const workflows = computed(() =>
  Workflow.query()
    .where(({ name }) =>
      name.toLocaleLowerCase().includes(state.query.toLocaleLowerCase())
    )
    .orderBy('createdAt', 'desc')
    .get()
);
const collection = computed(() => Collection.find(route.params.id));
const collectionFlow = computed(() => {
  if (!collection.value) return [];

  return collection.value.flow.map(({ itemId, type }) => {
    if (type === 'workflow') return Workflow.find(itemId) || { type };

    return blocks[itemId];
  });
});

function deleteLog(logId) {
  Log.delete(logId).then(() => {
    store.dispatch('saveToStorage', 'logs');
  });
}
function executeCollection() {
  sendMessage('collection:execute', collection.value, 'background');
}
function updateCollection(data) {
  Collection.update({
    where: route.params.id,
    data,
  });
}
function updateGlobalData(str) {
  let value = str;

  if (value.length > 1e4) {
    value = value.slice(0, 1e4);
  }

  updateCollection({ globalData: value });
}
function updateCollectionFlow(event) {
  const flow = event.map(({ type, id, flowId, data }) => {
    const itemFlowId = flowId || nanoid();

    return type === 'block'
      ? { type, itemId: id, id: itemFlowId, data }
      : { type: 'workflow', itemId: id, id: itemFlowId };
  });

  updateCollection({ flow });
}
function deleteCollectionFlow(index) {
  const flow = [...collection.value.flow];

  flow.splice(index, 1);

  updateCollection({ flow });
}
function deleteCollection() {
  dialog.confirm({
    title: 'Delete collection',
    okVariant: 'danger',
    body: 'Are you sure you want to delete this collection?',
    onConfirm: () => {
      Collection.delete(route.params.id).then(() => {
        router.replace('/collections');
      });
    },
  });
}

watch(
  () => collectionOptions,
  (value) => {
    Collection.update({
      where: route.params.id,
      data: {
        options: value,
      },
    });
  },
  { deep: true }
);

onMounted(() => {
  Object.assign(collectionOptions, collection.value.options);

  collectionFlow.value.forEach((item, index) => {
    if (!item.itemId && item.type === 'workflow') {
      deleteCollectionFlow(index);
    }
  });
});
</script>
<style>
.ghost {
  position: relative;
  z-index: 100;
}
</style>
