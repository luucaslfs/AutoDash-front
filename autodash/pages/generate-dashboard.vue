<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Generate Dashboard</h1>

    <!-- Upload de Arquivo -->
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

    <!-- Pré-visualização dos Dados -->
    <div v-if="filePreview" class="mb-4">
      <h2 class="text-xl font-semibold mb-2">Data Preview</h2>
      <pre class="bg-gray-100 p-4 rounded overflow-x-auto">{{ filePreview }}</pre>
    </div>

    <!-- Seleção do Modelo AI -->
    <div v-if="filePreview" class="mb-4">
      <label class="block mb-2 font-semibold">Select AI Model:</label>
      <select v-model="selectedModel" class="w-full p-2 border border-gray-300 rounded">
        <option value="claude">Claude</option>
        <option value="openai">OpenAI</option>
      </select>
    </div>

    <!-- Botão para Gerar Dashboard -->
    <button
      v-if="filePreview"
      @click="generateDashboard"
      :disabled="isGenerating"
      class="bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded mb-4"
    >
      {{ isGenerating ? "Generating..." : "Generate Dashboard" }}
    </button>

    <!-- Código Gerado e Botão de Download -->
    <div v-if="generatedCode" class="mb-4">
      <h2 class="text-xl font-semibold mb-2">
        Generated Streamlit Dashboard Code
      </h2>
      <div class="relative">
        <pre class="bg-gray-100 p-4 rounded overflow-x-auto">{{ generatedCode }}</pre>
        <button
          @click="copyCode"
          class="absolute top-2 right-2 bg-blue-500 hover:bg-blue-600 text-white font-bold py-1 px-2 rounded text-sm"
        >
          Copy
        </button>
      </div>

      <!-- Botão de Download do Projeto Estruturado -->
      <button
        @click="downloadZip"
        :disabled="!uniqueId || isDownloading"
        class="relative bg-purple-500 hover:bg-purple-600 text-white font-bold py-2 px-4 rounded mt-4 flex items-center"
      >
        <span v-if="isDownloading">Downloading...</span>
        <span v-else>Download Project</span>
        <svg
          v-if="isDownloading"
          class="animate-spin h-5 w-5 ml-2 text-white"
          xmlns="http://www.w3.org/2000/svg"
          fill="none"
          viewBox="0 0 24 24"
        >
          <circle
            class="opacity-25"
            cx="12"
            cy="12"
            r="10"
            stroke="currentColor"
            stroke-width="4"
          ></circle>
          <path
            class="opacity-75"
            fill="currentColor"
            d="M4 12a8 8 0 018-8v8H4z"
          ></path>
        </svg>
      </button>
    </div>

    <!-- Mensagem de Erro -->
    <p v-if="error" class="error">{{ error }}</p>
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
const uniqueId = ref(""); // Armazena o unique_id
const isGenerating = ref(false);
const isDownloading = ref(false); // Controla o estado de download
const previewData = ref(null);
const selectedModel = ref("claude"); // Valor padrão
const error = ref(""); // Estado para mensagens de erro

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
  error.value = "";
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
    uniqueId.value = result.unique_id; // Armazena o unique_id
  } catch (error) {
    console.error("Error generating dashboard:", error);
    error.value = `Failed to generate dashboard: ${error.message}`;
  } finally {
    isGenerating.value = false;
  }
};

// Função para copiar o código gerado
const copyCode = () => {
  navigator.clipboard.writeText(generatedCode.value);
  alert("Code copied to clipboard!");
};

// Função para baixar o projeto estruturado
const downloadZip = async () => {
  if (!uniqueId.value) {
    error.value = "Unique ID not found. Please generate the dashboard first.";
    return;
  }

  isDownloading.value = true;
  error.value = "";

  try {
    const requestBody = {
      unique_id: uniqueId.value,
    };

    const response = await fetch(
      `${config.public.apiBase}/api/v1/download-dashboard`,
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
      throw new Error(`Failed to download project: ${errorText}`);
    }

    const blob = await response.blob();
    const url = window.URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'dashboard_project.zip';
    document.body.appendChild(a);
    a.click();
    a.remove();
    window.URL.revokeObjectURL(url);

    // Limpar o estado após o download
    generatedCode.value = "";
    uniqueId.value = "";
    filePreview.value = "";
    previewData.value = null;
    selectedModel.value = "claude";
  } catch (error) {
    console.error("Error downloading project:", error);
    error.value = `Failed to download project: ${error.message}`;
  } finally {
    isDownloading.value = false;
  }
};
</script>

<style scoped>
/* Adicione estilos personalizados aqui, se necessário */
.container {
  max-width: 800px;
}
.error {
  color: red;
}
</style>
