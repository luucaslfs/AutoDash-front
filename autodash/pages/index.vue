<template>
  <div>
    <h1>Welcome to AutoDash</h1>
    <p v-if="isAuthenticated">
      Hello, {{ user?.name || "User" }}! You're logged in.
    </p>
    <p v-else>Please log in to use AutoDash.</p>
  </div>
</template>

<script setup>
import { useAuthStore } from "~/stores/auth";
import { storeToRefs } from "pinia";

const authStore = useAuthStore();
const { isAuthenticated, user } = storeToRefs(authStore);
</script>
