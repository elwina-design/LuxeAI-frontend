<template>
  <div v-if="showLimit" class="p-3 mb-4 bg-red-500/10 border border-red-300 rounded-lg shadow-md transition-all duration-300 transform scale-100">
    <div class="flex items-center space-x-3">
      <svg class="w-6 h-6 text-red-600 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.308 15c-.77 1.333.192 3 1.732 3z"></path></svg>
      
      <p class="text-sm font-medium text-red-800">
        {{ message }}
      </p>
    </div>
  </div>
</template>

<script setup>
import { defineProps } from 'vue';

defineProps({
  message: {
    type: String,
    required: true,
    default: "This chat has reached its limit."
  },
  showLimit: {
    type: Boolean,
    required: true,
    default: false
  }
});
</script>

<style scoped>
</style>