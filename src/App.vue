<script setup>
import { Button } from "@/components/ui/button";
import Canvas from "@/components/Canvas.vue";
import Video from "@/components/videoCanvas.vue";
import { ref } from "vue";

const mode = ref(true);

const toggleMode = () => {
  mode.value = !mode.value;
};
</script>

<template>
  <Button
    @click="toggleMode"
    class="absolute z-10 top-4 left-1/2 -translate-x-1/2"
    >Switch to {{ mode ? "Video" : "Canvas" }}</Button
  >
  <main class="flex relative p-4 items-center justify-center h-[100vh]">
    <Canvas v-if="mode" />
    <Video v-else />
  </main>
</template>
