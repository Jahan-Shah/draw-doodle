<script setup>
import * as tf from "@tensorflow/tfjs";
import { onMounted, ref, watchEffect, computed } from "vue";
import { useUserMedia, useDevicesList } from "@vueuse/core";
import { useDebounceFn } from "@vueuse/core";
import { useRound } from "@vueuse/math";
import { Button } from "@/components/ui/button";
import classes from "@/assets/class_names.json";

const canvas = ref(null);
const predictions = ref([]);

let model;

const loadModel = async () => {
  model = await tf.loadLayersModel("/model/model.json");
};

const currentCamera = ref("");
const { videoInputs: cameras } = useDevicesList({
  requestPermissions: true,
  onUpdated() {
    if (!cameras.value.find((i) => i.deviceId === currentCamera.value))
      currentCamera.value = cameras.value[0]?.deviceId;
  },
});

const video = ref();
const { stream, enabled } = useUserMedia({
  constraints: { video: { deviceId: currentCamera } },
});

const predict = useDebounceFn(async () => {
  if (!canvas.value) return;
  const image = new Image();
  image.src = canvas.value.toDataURL();

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
    .slice(0, 10);

  return topValues.map(({ value, index }) => ({
    label: classes[index],
    accuracy: useRound(value * 100),
  }));
});

watchEffect(() => {
  if (video.value) {
    video.value.srcObject = stream.value;
    video.value.play(); // Make sure the video starts playing

    const context = canvas.value.getContext("2d");

    const render = () => {
      if (!video.value) return;
      if (video.value.readyState === video.value.HAVE_ENOUGH_DATA) {
        context.drawImage(
          video.value,
          0,
          0,
          canvas.value.width,
          canvas.value.height
        );
        let imageData = context.getImageData(
          0,
          0,
          canvas.value.width,
          canvas.value.height
        );
        let data = imageData.data;

        for (let i = 0; i < data.length; i += 4) {
          let avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
          let threshold = 0.5 * 255;
          let binaryColor = avg > threshold ? 255 : 0;
          data[i] = binaryColor; // red
          data[i + 1] = binaryColor; // green
          data[i + 2] = binaryColor; // blue
        }

        context.putImageData(imageData, 0, 0);
      }

      requestAnimationFrame(render);
    };

    render();
  }
});

onMounted(async () => {
  await loadModel();
  predict();
});
</script>
<template>
  <video
    ref="video"
    muted
    autoplay
    controls
    class="h-[500px] hidden w-[500px]"
  />
  <canvas
    ref="canvas"
    class="border -scale-x-100 rounded-md"
    height="560"
    width="560"
  ></canvas>
  <div class="p-6 h-full flex flex-col items-center justify-center">
    <p v-for="item in guess">{{ item.label }}: {{ item.accuracy }}%</p>
  </div>
  <div class="absolute bottom-4 flex gap-2">
    <Button @click="enabled = !enabled"
      >{{ enabled ? "Stop" : "Start" }} Camera</Button
    >
  </div>
</template>
