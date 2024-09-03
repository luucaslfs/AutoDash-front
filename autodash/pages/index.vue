<template>
  <div>
    <h1>Welcome to AutoDash</h1>
    <p v-if="isAuthenticated">
      Hello, {{ user?.name || "User" }}! You're logged in.
      <NuxtLink to="/generate-dashboard" class="text-blue-500 hover:underline"
        >Generate Dashboard</NuxtLink
      >
    </p>
    <p v-else>Please log in to use AutoDash.</p>
  </div>
</template>
