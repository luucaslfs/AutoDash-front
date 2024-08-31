<template>
  <nav>
    <NuxtLink to="/">Home</NuxtLink>
    <NuxtLink v-if="!isAuthenticated" to="/login">Login</NuxtLink>
    <a v-else href="#" @click.prevent="logout">Logout</a>
  </nav>
</template>

<script setup>
import { useAuthStore } from "~/stores/auth";
import { storeToRefs } from "pinia";

const authStore = useAuthStore();
const { isAuthenticated } = storeToRefs(authStore);

const logout = () => {
  authStore.logout();
  // Redirect to home page after logout
  navigateTo("/");
};
</script>

<style scoped>
nav {
  padding: 1rem;
  background-color: #f0f0f0;
}

nav a {
  margin-right: 1rem;
  color: #333;
  text-decoration: none;
}

nav a:hover {
  text-decoration: underline;
}
</style>
