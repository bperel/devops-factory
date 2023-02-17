<template>
  <form>
    <div>
      <label for="bps">Requests per second</label>
      <input type="number" id="bps" v-model="bps" min="0" max="10" />
    </div>
    <div>
      <label for="myRange">Playback speed</label>
      <input
        type="range"
        min="0.9"
        max="1.1"
        step="0.1"
        id="myRange"
        v-model="playbackSpeed"
      />
    </div>
  </form>
</template>

<script lang="ts" setup>
import { ref, watch } from "vue";

const emit = defineEmits<{
  (e: "change-bps", bps: number): void;
  (e: "change-playback-speed", playbackSpeed: number): void;
}>();

const bps = ref(0 as number);
const playbackSpeed = ref(1 as number);

watch(
  () => bps.value,
  (newValue) => {
    emit("change-bps", newValue);
  }
);

watch(
  () => playbackSpeed.value,
  (newValue) => {
    console.log(newValue);
    emit("change-playback-speed", newValue);
  }
);
</script>

<style scoped>
form {
  position: fixed;
  top: 0;
  right: 0;
  border: 1px solid grey;
  background: darkgray;
  padding: 10px;
}
</style>
