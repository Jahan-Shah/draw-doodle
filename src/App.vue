<script setup>
import {
  HoverCard,
  HoverCardContent,
  HoverCardTrigger,
} from "@/components/ui/hover-card";
import { Button } from "@/components/ui/button";
import Canvas from "@/components/Canvas.vue";
import Video from "@/components/videoCanvas.vue";
import { ref, computed, provide } from "vue";
import classes from "@/assets/class_names.json";
import { useWindowSize } from "@vueuse/core";
import { cn } from "@/lib/utils";

const { width } = useWindowSize();

const mode = ref(true);
const mobile = computed(() => width.value < 768);

provide("mobile", mobile);
provide("width", width);

const toggleMode = () => {
  mode.value = !mode.value;
};
</script>

<template>
  <div
    class="absolute flex items-center gap-3 z-10 top-4 left-1/2 -translate-x-1/2"
  >
    <Button @click="toggleMode"
      >Switch to {{ mode ? "Video" : "Canvas" }}</Button
    >
    <HoverCard>
      <HoverCardTrigger class="cursor-pointer">Classes</HoverCardTrigger>
      <HoverCardContent class="w-max">
        <div class="grid grid-cols-3 gap-4">
          <p v-for="cls in classes">{{ cls }}</p>
        </div>
      </HoverCardContent>
    </HoverCard>
  </div>
  <main
    class="flex relative p-4 items-center justify-center h-[100vh]"
    :class="{ 'flex-col mt-12': mobile }"
  >
    <Canvas v-if="mode" />
    <Video v-else />
  </main>
</template>
