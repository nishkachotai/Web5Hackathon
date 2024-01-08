<script setup>
import { onBeforeMount, ref } from 'vue';
import { PlusIcon as PlusIconMini, TrashIcon } from '@heroicons/vue/outline';
import { Web5 } from '@web5/api';
import VueQuillEditor from 'vue-quill-editor';
import { Sketch } from 'vue-color';

let web5;
let myDid = ref('');
const entries = ref([]);
const diaryBackgroundColor = ref('#ffffff'); // Default background color

const newEntryText = ref('');
const newEntryScore = ref('');
const selectedMood = ref('');

// Quill editor options
const quillOptions = {
  modules: {
    toolbar: [
      [{ 'header': [1, 2, 3, 4, 5, 6, false] }],
      ['bold', 'italic', 'underline', 'strike'],
      [{ 'list': 'ordered' }, { 'list': 'bullet' }],
      ['link', 'image'],
      [{ 'color': [] }, { 'background': [] }],
      ['clean'],
    ],
  },
  placeholder: 'Write your thoughts here',
  theme: 'snow',
};

onBeforeMount(async () => {
  const web5Result = await Web5.connect();
  web5 = web5Result.web5;
  myDid.value = web5Result.did;

  // Populate entries from DWN
  const { records } = await web5.dwn.records.query({
    message: {
      filter: {
        schema: 'http://some-schema-registry.org/serenity-scribe-entry',
      },
      dateSort: 'createdAscending',
    },
  });

  // Add entry to entries array
  for (let record of records) {
    const data = await record.data.json();
    const entry = { record, data, id: record.id };
    entries.value.push(entry);
  }
});

async function addEntry() {
  const entryData = {
    text: newEntryText.value,
    score: parseInt(newEntryScore.value, 10) || null,
    mood: selectedMood.value,
    timestamp: Math.floor(Date.now() / 1000),
  };

  newEntryText.value = '';
  newEntryScore.value = '';
  selectedMood.value = '';

  // Create the record.
  const { record } = await web5.dwn.records.create({
    data: entryData,
    message: {
      schema: 'http://some-schema-registry.org/serenity-scribe-entry',
      dataFormat: 'application/json',
    },
  });

  // add DWeb message recordId as a way to reference the message for further operations
  // e.g. updating it or overwriting it
  const data = await record.data.json();
  const entry = { record, data, id: record.id };
  entries.value.push(entry);
}

async function deleteEntry(entryItem) {
  let deletedEntry;
  let index = 0;

  for (let entry of entries.value) {
    if (entryItem.id === entry.id) {
      deletedEntry = entry;
      break;
    }
    index++;
  }

  entries.value.splice(index, 1);

  // Delete the record.
  await web5.dwn.records.delete({
    message: {
      recordId: deletedEntry.id,
    },
  });
}
</script>

<template>
  <div class="flex flex-col items-center justify-center min-h-full px-8 py-12 sm:px-6" :style="{ backgroundColor: diaryBackgroundColor }">
    <!-- Title -->
    <div class="w-full text-center mb-8">
      <h2 class="font-bold text-3xl tracking-tight">Serenity Scribe</h2>
      <div class="flex items-center justify-center">
        <div v-if="!myDid" id="mydid-container">Connecting to web5 . . .</div>
        <div v-else id="mydid-container">Web5 Connected</div>
      </div>
    </div>

    <!-- Background Color Picker -->
    <div class="mb-4">
      <label for="background-color" class="mr-2">Choose Background Color:</label>
      <sketch v-model="diaryBackgroundColor" />
    </div>

    <!-- Add Entry Form -->
    <div class="w-full mb-8">
      <!-- The form will only be displayed if myDid is truthy -->
      <form v-if="myDid" class="flex space-x-4 items-center w-full" @submit.prevent="addEntry">
        <div class="border-b border-gray-200 flex-grow mr-4">
          <label for="add-entry" class="sr-only">Add an entry</label>
          <!-- Use the Quill editor instead of textarea -->
          <vue-quill-editor
            v-model="newEntryText"
            @keydown.enter.exact.prevent="addEntry"
            :options="quillOptions"
          />
        </div>
        <div class="border-b border-gray-200 w-1/4 mr-4">
          <label for="add-score" class="sr-only">Score</label>
          <input
            type="number"
            name="add-score"
            id="add-score"
            v-model="newEntryScore"
            min="1"
            max="10"
            class="block border-0 border-transparent focus:ring-0 p-0 pb-2 resize-none sm:text-sm w-full"
            placeholder="Score (1-10)"
          />
        </div>
        <div class="border-b border-gray-200 w-1/4 mr-4">
          <label for="add-mood" class="sr-only">Mood</label>
          <select
            name="add-mood"
            id="add-mood"
            v-model="selectedMood"
            class="block border-0 border-transparent focus:ring-0 p-0 pb-2 resize-none sm:text-sm w-full"
          >
            <option value="" disabled>Select Mood</option>
            <option value="😊">😊 Happy</option>
            <option value="😐">😐 Neutral</option>
            <option value="😢">😢 Sad</option>
            <option value="😠">😠 Frustrated</option>
            <option value="😃">😃 Excited</option>
            <option value="😰">😰 Anxious</option>
            <option value="🙏">🙏 Grateful</option>
            <!-- Add more mood options as needed -->
          </select>
        </div>
        <div class="w-1/4">
          <button
            type="submit"
            class="bg-indigo-600 border border-transparent focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2 hover:bg-indigo-700 inline-flex items-center p-1 rounded-full shadow-sm text-white"
          >
            <!-- Make sure PlusIconMini is a registered component or a known element -->
            <PlusIconMini class="h-5 w-5" aria-hidden="true" />
          </button>
        </div>
      </form>
    </div>

    <!-- Entries -->
    <div
      v-if="entries.length > 0"
      class="w-full border-gray-200 border-t border-x rounded-lg shadow-md sm:max-w-xl sm:mx-auto sm:w-full"
    >
      <div
        v-for="entry in entries"
        :key="entry"
        class="border-b border-gray-200 flex flex-col p-4"
      >
        <div class="text-gray-500 text-xs mb-2">
          {{ entry.data.timestamp ? new Date(entry.data.timestamp * 1000).toLocaleString() : '' }}
        </div>
        <div class="font-light text-gray-500 text-xl">
          {{ entry.data.text }}
        </div>
        <div class="text-right text-gray-500 text-sm mt-2">
          Score: {{ entry.data.score || 'N/A' }} | Mood: {{ entry.data.mood || 'N/A' }}
        </div>
        <!-- Delete Entry -->
        <div class="mt-2 ml-auto">
          <div @click="deleteEntry(entry)" class="cursor-pointer">
            <TrashIcon class="h-8 text-gray-200 w-8" :class="'text-red-500'" />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
