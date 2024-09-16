<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Generate Dashboard</h1>

    <div class="flex justify-center mb-4">
      <label
        for="file-upload"
        class="cursor-pointer bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded"
      >
        Upload File
        <input
          id="file-upload"
          type="file"
          class="hidden"
          @change="handleFileUpload"
          accept=".csv,.xlsx"
        />
      </label>
    </div>

    <div v-if="filePreview" class="mb-4">
      <h2 class="text-xl font-semibold mb-2">Data Preview</h2>
      <pre class="bg-gray-100 p-4 rounded overflow-x-auto">{{
        filePreview
      }}</pre>
    </div>

    <div v-if="filePreview" class="mb-4">
      <label class="block mb-2 font-semibold">Select AI Model:</label>
      <select v-model="selectedModel" class="w-full p-2 border border-gray-300 rounded">
        <option value="claude">Claude</option>
        <option value="openai">OpenAI</option>
      </select>
    </div>

    <button
      v-if="filePreview"
      @click="generateDashboard"
      :disabled="isGenerating"
      class="bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded mb-4"
    >
      {{ isGenerating ? "Generating..." : "Generate Dashboard" }}
    </button>

    <div v-if="generatedCode" class="mb-4">
      <h2 class="text-xl font-semibold mb-2">
        Generated Streamlit Dashboard Code
      </h2>
      <div class="relative">
        <pre class="bg-gray-100 p-4 rounded overflow-x-auto">{{
          generatedCode
        }}</pre>
        <button
          @click="copyCode"
          class="absolute top-2 right-2 bg-blue-500 hover:bg-blue-600 text-white font-bold py-1 px-2 rounded text-sm"
        >
          Copy
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import { useRuntimeConfig } from "#app";
import Papa from "papaparse";

const config = useRuntimeConfig();

// Estado reativo
const filePreview = ref("");
const generatedCode = ref("");
const isGenerating = ref(false);
const previewData = ref(null);
const selectedModel = ref("claude"); // Valor padrão

// Handler de upload de arquivo
const handleFileUpload = async (event) => {
  const file = event.target.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = (e) => {
      const content = e.target.result;

      // Use Papa Parse para analisar corretamente o CSV
      Papa.parse(content, {
        complete: (results) => {
          const headers = results.data[0];
          const data = results.data
            .slice(1)
            .filter((row) => row.some((cell) => cell.trim() !== "")); // Remove linhas vazias

          filePreview.value = results.data
            .slice(0, 6)
            .map((row) => row.join(","))
            .join("\n");
          previewData.value = { columns: headers, data: data };

          console.log("Parsed data:", previewData.value);
        },
        error: (error) => {
          console.error("Error parsing CSV:", error);
          alert("Error parsing CSV file. Please check the file format.");
        },
      });
    };
    reader.readAsText(file);
  }
};

// Função para gerar o dashboard
const generateDashboard = async () => {
  if (!previewData.value) return;

  isGenerating.value = true;
  try {
    console.log("Sending data:", previewData.value, "Model:", selectedModel.value);

    const requestBody = {
      table_data: previewData.value,
      model: selectedModel.value,
    };

    const response = await fetch(
      `${config.public.apiBase}/api/v1/generate-dashboard`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify(requestBody),
      }
    );

    if (!response.ok) {
      const errorText = await response.text();
      console.error("Error response:", errorText);
      throw new Error(`Failed to generate dashboard: ${errorText}`);
    }

    const result = await response.json();
    generatedCode.value = result.dashboard_code;
  } catch (error) {
    console.error("Error generating dashboard:", error);
    alert(`Failed to generate dashboard: ${error.message}`);
  } finally {
    isGenerating.value = false;
  }
};

// Função para copiar o código gerado
const copyCode = () => {
  navigator.clipboard.writeText(generatedCode.value);
  alert("Code copied to clipboard!");
};
</script>

<style scoped>
/* Adicione estilos personalizados aqui, se necessário */
</style>
