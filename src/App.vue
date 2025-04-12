<script setup>
import OrderBook from "./components/OrderBook.vue";
</script>

<template>
  <div class="app">
    <OrderBook />
  </div>
</template>

<style>
body {
  margin: 0;
  padding: 0;
  background-color: #131b29;
}

.app {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>
