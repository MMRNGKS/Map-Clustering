<template>
  <q-page class="flex flex-center">
    <div>
      <MyMap style="width: 1080px" />
    </div>
  </q-page>
</template>

<script setup>
import MyMap from "../components/Map.vue";

defineOptions({
  name: "IndexPage",
});
</script>
