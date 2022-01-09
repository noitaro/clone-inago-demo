<template>
  <div id="app">
    <TradingVue />
  </div>
</template>

<script>
import TradingVue from "./components/TradingVue.vue";

export default {
  name: "App",
  components: {
    TradingVue,
  },
};
</script>

<style>
* {
  margin: 0px;
  padding: 0px;
}
</style>
