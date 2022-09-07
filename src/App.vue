<script setup lang="ts">
import { ref, computed, watch, onMounted } from "vue";
import Beep from "@/assets/beep.mp3";
import QrScanner from "qr-scanner";

type Camera = Awaited<ReturnType<typeof QrScanner.listCameras>>[number];
const player = ref<HTMLVideoElement>();
const cameras = ref<Camera[]>([]);
const selectedCamera = ref<Camera>();
const oldResults = localStorage.getItem("results");
const startingResults = oldResults ? JSON.parse(oldResults) : [];
const results = ref<{ id: number; data: string }[]>(startingResults);
let lastId = Math.max(...[...startingResults.map((d: any) => d.id), 0]);
const cooldown = ref(false);
const resultsString = computed(() =>
  results.value.map((result) => result.data)
);

const reversedResults = computed(() => results.value.slice().reverse());

function hasRepeat(link: string) {
  return resultsString.value.filter((result) => result === link).length > 1;
}

function handler(result: QrScanner.ScanResult) {
  console.log("scanned");
  if (!cooldown.value) {
    cooldown.value = true;
    results.value = [...results.value, { id: ++lastId, data: result.data }];
    new Audio(Beep).play();
    setTimeout(() => {
      cooldown.value = false;
    }, 1000);
  }
}

function removeResults(id: number) {
  results.value = results.value.filter((result) => result.id !== id);
}

watch(results, (newVal) => {
  localStorage.setItem("results", JSON.stringify(newVal));
});

onMounted(() => {
  QrScanner.listCameras(true).then((response) => {
    cameras.value = response;
  });

  const scanner = new QrScanner(player.value!, handler, {
    maxScansPerSecond: 1,
    highlightScanRegion: true,
    highlightCodeOutline: true,
  });

  scanner.start();

  watch(selectedCamera, (newVal) => {
    scanner.setCamera(newVal!.id);
  });
});
</script>

<template>
  <div class="container mx-auto">
    <div class="grid grid-cols-3 gap-4">
      <div class="col-span-2">
        <div>
          <div>
            <label for="cam-select">Camera</label>
          </div>

          <select
            id="cam-select"
            class="px-4 py-2 bg-gray-700"
            v-model="selectedCamera"
          >
            <option v-for="camera in cameras" :value="camera" :key="camera.id">
              {{ camera.label }}
            </option>
          </select>
        </div>

        <div>
          <video class="mt-4 w-full" ref="player"></video>
        </div>
      </div>

      <main class="col-span-1">
        <div v-if="results.length > 0">Results: {{ results.length }}</div>
        <ul
          v-if="results.length > 0"
          class="space-y-2 overflow-y-auto"
          style="height: 90vh"
        >
          <li
            v-for="code in reversedResults"
            :key="code.id"
            class="flex items-center text-left px-2 py-0.5 rounded"
            :class="hasRepeat(code.data) ? 'bg-red-500 text-white' : ''"
          >
            <a :href="code.data" target="_blank">{{ code.data }}</a>
            <div class="flex-grow ml-3"></div>
            <button class="p-0.5" @click="removeResults(code.id)">✖</button>
          </li>
        </ul>

        <div v-else>No results available</div>
      </main>
    </div>
  </div>
</template>
