<template>
  <main>
    <Camera v-if="cameraOn" @emitUp="stop" />
    <div v-else class="fixed bottom-5 text-center w-full">
      <button
        class="
          p-2
          bg-gray-200
          text-gray-800
          font-bold
          rounded-lg
          border-2 border-gray-800
          shadow-lg
          mb-2
        "
        @click="start"
      >
        Start Camera
      </button>
    </div>
    <div
      class="
        mt-10
        text-left
        font-bold
        p-3
        mx-2
        bg-red-200 bg-opacity-90
        rounded-lg
        shadow-lg
      "
    >
      <h1 class="p-2 text-2xl underline">JANコード</h1>
      <h2 class="p-2 text-lg bg-gray-100 text-gray-600 rounded-xl my-3">
        {{ code || "カメラで読み込んでください！" }}
      </h2>
    </div>
  </main>
</template>

<script lang="ts">
import { defineComponent, ref } from "vue";
import Camera from "./components/Camera.vue";

export default defineComponent({
  name: "App",
  components: {
    Camera,
  },
  setup() {
    const cameraOn = ref<boolean>(false);
    const code = ref<string | null>(null);
    const start = () => {
      cameraOn.value = true;
    };
    const stop = (data: string | null) => {
      cameraOn.value = false;
      if (data) code.value = data;
    };
    return {
      cameraOn,
      code,
      start,
      stop,
    };
  },
});
</script>
