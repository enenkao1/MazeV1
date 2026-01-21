<script setup lang="ts">
import { useRouter } from 'vue-router'

const router = useRouter()

const startGame = () => {
  router.push('/game')
}
</script>

<template>
  <div class="home-container">
    <button class="start-button" @click="startGame">开始游戏</button>
  </div>
</template>

<style scoped>
.home-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  width: 100vw;
  background-color: #fff;
}

.start-button {
  padding: 15px 40px;
  font-size: 20px;
  cursor: pointer;
  background-color: #fff;
  color: #000;
  border: 2px solid #000;
  font-weight: bold;
  transition: all 0.3s ease;
}

.start-button:hover {
  background-color: #000;
  color: #fff;
}
</style>

