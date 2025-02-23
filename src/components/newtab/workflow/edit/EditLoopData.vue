<template>
  <div>
    <ui-textarea
      :model-value="data.description"
      placeholder="Description"
      class="w-full"
      @change="updateData({ description: $event })"
    />
    <ui-input
      :model-value="data.loopId"
      class="w-full mb-3"
      label="Loop ID"
      placeholder="Loop ID"
      @change="updateLoopID"
    />
    <ui-select
      :model-value="data.loopThrough"
      placeholder="Loop through"
      class="w-full mb-2"
      @change="
        updateData({
          loopThrough: $event,
          loopData: $event === 'custom-data' ? data.loopData : '[]',
        })
      "
    >
      <option v-for="type in loopTypes" :key="type.id" :value="type.id">
        {{ type.name }}
      </option>
    </ui-select>
    <ui-input
      :model-value="data.maxLoop"
      class="w-full mb-4"
      min="0"
      type="number"
      label="Max data to loop (0 to disable)"
      title="Max numbers of data to loop"
      @change="updateData({ maxLoop: +$event || 0 })"
    />
    <ui-button
      v-if="data.loopThrough === 'custom-data'"
      class="w-full"
      variant="accent"
      @click="state.showDataModal = true"
    >
      Insert data
    </ui-button>
    <ui-modal
      v-model="state.showDataModal"
      title="Data"
      content-class="max-w-3xl"
    >
      <div class="flex mb-4 items-center">
        <ui-button variant="accent" @click="importFile">
          Import file
        </ui-button>
        <ui-button
          v-tooltip="'Options'"
          :class="{ 'text-primary': state.showOptions }"
          icon
          class="ml-2"
          @click="state.showOptions = !state.showOptions"
        >
          <v-remixicon name="riSettings3Line" />
        </ui-button>
        <p class="flex-1 text-overflow mx-4">{{ file.name }}</p>
        <template v-if="data.loopData.length > maxStrLength">
          <p class="mr-2">File too large to edit</p>
          <ui-button @click="updateData({ loopData: '[]' })">
            Clear data
          </ui-button>
        </template>
        <p v-else>Max file size is 1MB</p>
      </div>
      <div style="height: calc(100vh - 11rem)">
        <prism-editor
          v-show="!state.showOptions"
          v-model="state.tempLoopData"
          :highlight="highlighter('json')"
          :readonly="data.loopData.length > maxStrLength"
          class="py-4"
          @input="updateData({ loopData: $event.target.value })"
        />
        <div v-show="state.showOptions">
          <p class="font-semibold mb-2">CSV</p>
          <ui-checkbox v-model="options.header">
            Use the first row as keys
          </ui-checkbox>
        </div>
      </div>
    </ui-modal>
  </div>
</template>
<script setup>
import { onMounted, shallowReactive } from 'vue';
import { nanoid } from 'nanoid';
import { PrismEditor } from 'vue-prism-editor';
import Papa from 'papaparse';
import { highlighter } from '@/lib/prism';
import { openFilePicker } from '@/utils/helper';

const props = defineProps({
  blockId: {
    type: String,
    default: '',
  },
  data: {
    type: Object,
    default: () => ({}),
  },
});
const emit = defineEmits(['update:data']);

const maxStrLength = 5e4;
const maxFileSize = 1024 * 1024;
const loopTypes = [
  { id: 'data-columns', name: 'Data columns' },
  { id: 'custom-data', name: 'Custom data' },
];
const tempLoopData =
  props.data.loopData.length > maxStrLength
    ? props.data.loopData.slice(0, maxStrLength)
    : props.data.loopData;

const state = shallowReactive({
  tempLoopData,
  showOptions: false,
  showDataModal: false,
  workflowLoopData: {},
});
const options = shallowReactive({
  header: true,
});
const file = shallowReactive({
  size: 0,
  name: '',
  type: '',
});

function updateData(value) {
  emit('update:data', { ...props.data, ...value });
}
function updateLoopID(id) {
  let loopId = id.replace(/\s/g, '');

  if (!loopId) {
    loopId = nanoid(6);
  }

  updateData({ loopId });
}
function importFile() {
  openFilePicker(['application/json', 'text/csv', 'application/vnd.ms-excel'])
    .then(async (fileObj) => {
      if (fileObj.size > maxFileSize) {
        alert('The file size is the exceeded maximum allowed');
        return;
      }

      file.name = fileObj.name;
      file.type = fileObj.type;

      const csvTypes = ['text/csv', 'application/vnd.ms-excel'];

      const reader = new FileReader();

      reader.onload = ({ target }) => {
        let loopData;

        if (fileObj.type === 'application/json') {
          const result = JSON.parse(target.result);
          loopData = Array.isArray(result) ? result : [result];
        } else if (csvTypes.includes(fileObj.type)) {
          loopData = Papa.parse(target.result, options).data;
        }

        if (Array.isArray(loopData)) {
          const loopDataStr = JSON.stringify(loopData);

          state.tempLoopData =
            loopDataStr.length > maxStrLength
              ? loopDataStr.slice(0, maxStrLength)
              : loopDataStr;
          updateData({ loopData: loopDataStr });
        }
      };

      reader.readAsText(fileObj);
    })
    .catch((error) => {
      console.error(error);
      if (error.message.startsWith('invalid')) alert(error.message);
    });
}

onMounted(() => {
  if (!props.data.loopId) {
    updateData({ loopId: nanoid(6) });
  }
});
</script>
