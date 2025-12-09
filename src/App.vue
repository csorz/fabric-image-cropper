<script setup lang="ts">
import { onMounted, ref } from "vue";
import HelloWorld from './components/HelloWorld.vue'
import Clipping from './components/Clipping.vue'

const clippingRef = ref(null);

onMounted(() => {
  console.log('clippingRef.value.getSvgs', clippingRef.value.getSvgs())
  // 新增
  clippingRef.value.init({
    imgBgUrl: "/public_bg2.png",
    imgBgScale: 0.9564293304994687,
    svgPath: "M18.25,15.52l1.36,7.92L12.5,19.7,5.39,23.44l1.36-7.92L1,9.92,8.95,8.76l3.55-7.2,3.55,7.2L24,9.92Z",
    download: true
  });

  // setTimeout(() => {
  //   clippingRef.value.dispose();
    
  //   setTimeout(() => {
  //     // 修改
  //     clippingRef.value.init({
  //       imgBgUrl: "/public_bg2.png",
  //       imgBgScale: 0.9564293304994687,
  //       svgPath: "M20.808,11.079C19.829,16.132,12,20.5,12,20.5s-7.829-4.368-8.808-9.421C2.227,6.1,5.066,3.5,8,3.5a4.444,4.444,0,0,1,4,2,4.444,4.444,0,0,1,4-2C18.934,3.5,21.773,6.1,20.808,11.079Z",
  //       originX: "center",
  //       originY: "center",
  //       top: 400.3340710860457,
  //       left: 1636.6731248351916,
  //       scaleX: 15.679645977631914,
  //       scaleY: 15.679645977631914,
  //       angle: 40.49110521250504,
  //       download: true
  //     });
  //   }, 2000)
  // }, 2000)


})
</script>

<template>
  <div>
    <a href="https://vite.dev" target="_blank">
      <img src="/vite.svg" class="logo" alt="Vite logo" />
    </a>
    <a href="https://vuejs.org/" target="_blank">
      <img src="./assets/vue.svg" class="logo vue" alt="Vue logo" />
    </a>
    <Clipping ref="clippingRef"></Clipping>
    <HelloWorld msg="Vite + Vue"></HelloWorld>
  </div>
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
