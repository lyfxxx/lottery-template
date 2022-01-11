<!--
 * @Author: your name
 * @Date: 2021-12-27 18:22:02
 * @LastEditTime: 2022-01-06 10:57:55
 * @LastEditors: Please set LastEditors
 * @Description: 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
 * @FilePath: \lottery\src\App.vue
-->
<template>
  <lottery msg="Welcome to Your Vue.js App" />
</template>

<script>
import lottery from "./components/lottery.vue";

export default {
  name: "App",
  components: {
    lottery,
  },
};
</script>

<style>
body,
html {
  margin: 0;
  padding: 0;
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin: 0;
  padding: 0;
}
</style>
