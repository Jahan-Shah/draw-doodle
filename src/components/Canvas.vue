<script setup>
import * as tf from "@tensorflow/tfjs";
import { Button } from "@/components/ui/button";
import { ref, onMounted, computed, inject } from "vue";
import { useDebounceFn } from "@vueuse/core";
import { useRound } from "@vueuse/math";
import vueDrawingCanvas from "vue-drawing-canvas";
import classes from "@/assets/class_names.json";

const VueCanvasDrawing = ref(null);
const canvasImage = ref();
const predictions = ref([]);
const mobile = inject("mobile");
const width = inject("width");
let model;

const canvasSize = computed(() => (mobile.value ? width.value - 16 : 560));

const loadModel = async () => {
  model = await tf.loadLayersModel("/model/model.json");
};

const predict = useDebounceFn(async () => {
  const image = new Image();
  image.src = canvasImage.value;

  await new Promise((resolve) => {
    image.onload = resolve;
  });

  const tensor = tf.browser.fromPixels(image, 1);
  const resized = tf.image.resizeBilinear(tensor, [28, 28]).toFloat();
  const normalized = tf.scalar(1.0).sub(resized.div(tf.scalar(255.0)));
  const batched = normalized.expandDims(0);

  if (model) {
    const predictionsData = await model.predict(batched).data();
    predictions.value = Array.from(predictionsData);
  }
}, 1500);

const guess = computed(() => {
  predict();
  const topValues = [...predictions.value]
    .map((value, index) => ({ value, index }))
    .sort((a, b) => b.value - a.value)
    .slice(0, mobile.value ? 2 : 10);

  return topValues.map(({ value, index }) => ({
    label: classes[index],
    accuracy: useRound(value * 100),
  }));
});

onMounted(async () => {
  await loadModel();
  predict();
});
</script>

<template>
  <vue-drawing-canvas
    class="border-2 rounded-md"
    v-model:image="canvasImage"
    ref="VueCanvasDrawing"
    :width="canvasSize"
    :height="canvasSize"
    lineJoin="round"
    :lineWidth="16"
  />
  <div
    class="p-6 h-full flex flex-col items-center justify-center"
    :class="{ 'h-auto': mobile }"
  >
    <p v-for="item in guess">{{ item.label }}: {{ item.accuracy }}%</p>
  </div>
  <div class="absolute bottom-4 flex gap-2">
    <Button @click.prevent="VueCanvasDrawing.reset()">Erase</Button>
    <Button variant="secondary" @click="VueCanvasDrawing.undo()">Undo</Button>
  </div>
</template>
