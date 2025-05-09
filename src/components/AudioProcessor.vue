<script setup lang="ts">
import { ref, onMounted, computed } from "vue";

const API_KEY_STORAGE_KEY = "assemblyai_api_key";
const apiKey = ref("");
const audioFile = ref<File | null>(null);
const outputType = ref("transcript");
const characterLength = ref(32); // Default character length for subtitles
const isLoading = ref(false);
const errorMessage = ref("");
const successMessage = ref("");
const processingProgress = ref(0);
const processingStage = ref("");

// UI states
const apiKeyVisible = ref(false);
const dragActive = ref(false);

// Load API key from localStorage on component mount
onMounted(() => {
  const savedApiKey = localStorage.getItem(API_KEY_STORAGE_KEY);
  if (savedApiKey) {
    apiKey.value = savedApiKey;
  }
});

// Save API key to localStorage whenever it changes
const saveApiKey = () => {
  localStorage.setItem(API_KEY_STORAGE_KEY, apiKey.value);
};

// Computed property for file size display
const fileSize = computed(() => {
  if (!audioFile.value) return "";

  const fileSizeBytes = audioFile.value.size;
  if (fileSizeBytes < 1024 * 1024) {
    return `${(fileSizeBytes / 1024).toFixed(2)} KB`;
  } else {
    return `${(fileSizeBytes / (1024 * 1024)).toFixed(2)} MB`;
  }
});

// Handle file dropping
const handleDragEnter = (event: DragEvent) => {
  event.preventDefault();
  dragActive.value = true;
};

const handleDragLeave = (event: DragEvent) => {
  event.preventDefault();
  dragActive.value = false;
};

const handleDragOver = (event: DragEvent) => {
  event.preventDefault();
  dragActive.value = true;
};

const handleDrop = (event: DragEvent) => {
  event.preventDefault();
  dragActive.value = false;

  if (event.dataTransfer?.files && event.dataTransfer.files.length > 0) {
    const file = event.dataTransfer.files[0];
    if (file.type.startsWith("audio/")) {
      audioFile.value = file;
      errorMessage.value = "";
    } else {
      errorMessage.value = "Please select an audio file (MP3, WAV, etc.)";
    }
  }
};

const handleFileChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  if (target.files && target.files.length > 0) {
    audioFile.value = target.files[0];
    errorMessage.value = "";
  }
};

const clearFile = () => {
  audioFile.value = null;
};

const toggleApiKeyVisibility = () => {
  apiKeyVisible.value = !apiKeyVisible.value;
};

const processAudio = async () => {
  if (!apiKey.value) {
    errorMessage.value = "Please enter your AssemblyAI API key";
    return;
  }

  // Save API key to localStorage
  saveApiKey();

  if (!audioFile.value) {
    errorMessage.value = "Please select an audio file";
    return;
  }

  try {
    isLoading.value = true;
    errorMessage.value = "";
    successMessage.value = "";
    processingProgress.value = 10;
    processingStage.value = "Uploading audio file...";

    // Step 1: Upload the file to AssemblyAI
    const uploadResponse = await fetch("https://api.assemblyai.com/v2/upload", {
      method: "POST",
      headers: {
        Authorization: apiKey.value,
        "Content-Type": "application/octet-stream",
      },
      body: audioFile.value,
    });

    if (!uploadResponse.ok) {
      throw new Error("Failed to upload audio file");
    }

    const uploadResult = await uploadResponse.json();
    const audioUrl = uploadResult.upload_url;

    processingProgress.value = 30;
    processingStage.value = "Submitting transcription request...";

    // Step 2: Submit the transcription request
    const transcriptResponse = await fetch(
      "https://api.assemblyai.com/v2/transcript",
      {
        method: "POST",
        headers: {
          Authorization: apiKey.value,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          audio_url: audioUrl,
          language_code: "en_us",
          speech_model: "slam-1",
        }),
      }
    );

    if (!transcriptResponse.ok) {
      throw new Error("Failed to submit transcription request");
    }

    const transcriptResult = await transcriptResponse.json();
    const transcriptId = transcriptResult.id;

    processingProgress.value = 50;
    processingStage.value = "Processing audio...";

    // Step 3: Poll for the result
    let result;
    let status = "processing";
    let pollCount = 0;

    while (status === "processing" || status === "queued") {
      await new Promise((resolve) => setTimeout(resolve, 1000));

      const pollingResponse = await fetch(
        `https://api.assemblyai.com/v2/transcript/${transcriptId}`,
        {
          method: "GET",
          headers: {
            Authorization: apiKey.value,
          },
        }
      );

      if (!pollingResponse.ok) {
        throw new Error("Failed to check transcription status");
      }

      result = await pollingResponse.json();
      status = result.status;

      // Update progress based on polling
      pollCount++;
      if (status === "processing") {
        processingProgress.value = Math.min(80, 50 + pollCount * 2);
        processingStage.value = `Processing audio... (${processingProgress.value}%)`;
      }
    }

    if (status === "completed") {
      processingProgress.value = 90;
      processingStage.value = "Generating output file...";

      let output: string;
      let filename: string;

      // Get the appropriate output format
      if (outputType.value === "subtitles") {
        processingStage.value = "Generating subtitles...";
        processingProgress.value = 90;

        // Fetch SRT content using the correct endpoint with the existing transcript ID
        const srtResponse = await fetch(
          `https://api.assemblyai.com/v2/transcript/${transcriptId}/srt?chars_per_caption=${characterLength.value}`,
          {
            method: "GET",
            headers: {
              Authorization: apiKey.value,
            },
          }
        );

        if (!srtResponse.ok) {
          throw new Error("Failed to get SRT file");
        }

        output = await srtResponse.text();
        filename = `${audioFile.value.name.split(".")[0]}.srt`;
      } else {
        // Use the text transcript
        output = result.text;
        filename = `${audioFile.value.name.split(".")[0]}.txt`;
      }

      processingProgress.value = 100;
      processingStage.value = "Complete!";

      // Download the result
      downloadFile(output, filename);
      successMessage.value = `Your ${outputType.value} has been successfully processed and downloaded!`;
    } else {
      throw new Error(`Transcription failed with status: ${status}`);
    }
  } catch (error) {
    errorMessage.value =
      error instanceof Error ? error.message : "An unknown error occurred";
  } finally {
    isLoading.value = false;
  }
};

const downloadFile = (content: string, filename: string) => {
  const blob = new Blob([content], { type: "text/plain" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
};
</script>

<template>
  <div>
    <!-- Main Card -->
    <div
      class="bg-white overflow-hidden shadow-md rounded-xl border border-gray-200"
    >
      <!-- Card Header -->
      <div class="bg-indigo-700 px-6 py-4">
        <div class="flex items-center">
          <div class="flex-shrink-0">
            <svg
              class="h-8 w-8 text-white"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M9 19V6l12-3v13M9 19c0 1.105-1.343 2-3 2s-3-.895-3-2 1.343-2 3-2 3 .895 3 2zm12-3c0 1.105-1.343 2-3 2s-3-.895-3-2 1.343-2 3-2 3 .895 3 2zM9 10l12-3"
              />
            </svg>
          </div>
          <div class="ml-3">
            <h2 class="text-xl font-bold text-white">
              Audio Transcription Tool
            </h2>
            <p class="text-indigo-200 text-sm">
              Convert audio files to text with AI-powered transcription
            </p>
          </div>
        </div>
      </div>

      <!-- Card Body -->
      <div class="px-6 py-6 sm:p-8">
        <!-- API Key Field -->
        <div class="mb-8 bg-gray-50 p-4 rounded-lg border border-gray-200">
          <div
            class="flex flex-col sm:flex-row sm:items-center sm:justify-between"
          >
            <label
              for="api-key"
              class="block text-sm font-medium text-gray-700 mb-2 sm:mb-0"
            >
              AssemblyAI API Key
            </label>
            <div
              class="text-xs text-indigo-600 cursor-pointer hover:text-indigo-800"
              @click="toggleApiKeyVisibility"
            >
              {{ apiKeyVisible ? "Hide Key" : "Show Key" }}
            </div>
          </div>

          <div class="relative rounded-md shadow-sm mt-2">
            <input
              id="api-key"
              v-model="apiKey"
              :type="apiKeyVisible ? 'text' : 'password'"
              class="block w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-indigo-500 focus:border-indigo-500"
              placeholder="Enter your AssemblyAI API Key"
              autocomplete="off"
            />
            <div
              class="absolute inset-y-0 right-0 pr-3 flex items-center text-gray-400"
            >
              <svg
                class="h-5 w-5"
                fill="none"
                viewBox="0 0 24 24"
                stroke="currentColor"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M15 7a2 2 0 012 2m4 0a6 6 0 01-7.743 5.743L11 17H9v2H7v2H4a1 1 0 01-1-1v-2.586a1 1 0 01.293-.707l5.964-5.964A6 6 0 1121 9z"
                />
              </svg>
            </div>
          </div>
          <p class="mt-2 text-xs text-gray-500">
            Your API key is stored locally in your browser and never sent to our
            servers.
          </p>
        </div>

        <!-- File Upload Field -->
        <div class="mb-8">
          <div class="flex justify-between items-center mb-2">
            <label class="block text-sm font-medium text-gray-700">
              Upload Audio File
            </label>
            <div class="text-xs text-gray-500">Max size: 500MB</div>
          </div>

          <!-- Drag & Drop Zone -->
          <div
            @dragenter="handleDragEnter"
            @dragleave="handleDragLeave"
            @dragover="handleDragOver"
            @drop="handleDrop"
            class="mt-1 relative border-2 rounded-lg transition-all duration-200 cursor-pointer"
            :class="[
              dragActive
                ? 'border-indigo-400 bg-indigo-50'
                : audioFile
                  ? 'border-green-100 bg-green-50 border-solid'
                  : 'border-indigo-100 border-dashed hover:border-indigo-300 hover:bg-gray-50',
            ]"
          >
            <div class="p-6 text-center" v-if="!audioFile">
              <svg
                class="mx-auto h-12 w-12"
                :class="dragActive ? 'text-indigo-500' : 'text-indigo-400'"
                stroke="currentColor"
                fill="none"
                viewBox="0 0 48 48"
                aria-hidden="true"
              >
                <path
                  d="M28 8H12a4 4 0 00-4 4v20m32-12v8m0 0v8a4 4 0 01-4 4H12a4 4 0 01-4-4v-4m32-4l-3.172-3.172a4 4 0 00-5.656 0L28 28M8 32l9.172-9.172a4 4 0 015.656 0L28 28m0 0l4 4m4-24h8m-4-4v8m-12 4h.02"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                />
              </svg>
              <div class="flex flex-col items-center mt-4 space-y-1">
                <div class="flex text-sm text-gray-600">
                  <label
                    for="audio-file"
                    class="relative cursor-pointer rounded-md font-medium text-indigo-600 hover:text-indigo-500 focus-within:outline-none"
                  >
                    <span>Upload a file</span>
                    <input
                      id="audio-file"
                      type="file"
                      @change="handleFileChange"
                      accept="audio/*"
                      class="sr-only"
                    />
                  </label>
                  <p class="pl-1">or drag and drop</p>
                </div>
                <p class="text-xs text-gray-500">
                  MP3, WAV, M4A, and other audio formats
                </p>
              </div>
            </div>

            <!-- File Selected View -->
            <div v-else class="p-6">
              <div class="flex items-center justify-between">
                <div class="flex items-center space-x-3">
                  <div class="flex-shrink-0">
                    <div
                      class="flex items-center justify-center w-10 h-10 rounded-full bg-green-100"
                    >
                      <svg
                        class="h-6 w-6 text-green-600"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                      >
                        <path
                          stroke-linecap="round"
                          stroke-linejoin="round"
                          stroke-width="2"
                          d="M9 19V6l12-3v13M9 19c0 1.105-1.343 2-3 2s-3-.895-3-2 1.343-2 3-2 3 .895 3 2zm12-3c0 1.105-1.343 2-3 2s-3-.895-3-2 1.343-2 3-2 3 .895 3 2zM9 10l12-3"
                        />
                      </svg>
                    </div>
                  </div>
                  <div class="min-w-0 flex-1">
                    <div class="text-sm font-medium text-gray-900 truncate">
                      {{ audioFile.name }}
                    </div>
                    <div class="flex mt-1 items-center text-xs text-gray-500">
                      <span>{{ fileSize }}</span>
                      <span class="mx-1">•</span>
                      <span>{{ audioFile.type || "audio file" }}</span>
                    </div>
                  </div>
                </div>

                <button
                  type="button"
                  class="flex-shrink-0 ml-4 p-1 text-gray-400 rounded-full hover:text-gray-500 focus:outline-none focus:ring-2 focus:ring-indigo-500"
                  @click="clearFile"
                >
                  <svg class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                    <path
                      fill-rule="evenodd"
                      d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z"
                      clip-rule="evenodd"
                    />
                  </svg>
                </button>
              </div>
            </div>
          </div>
        </div>

        <!-- Output Format Selection -->
        <div class="mb-8">
          <label class="block text-sm font-medium text-gray-700 mb-3">
            Output Format
          </label>
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div
              class="relative border rounded-lg p-4 cursor-pointer transition-all duration-200 hover:bg-gray-50"
              :class="
                outputType === 'transcript'
                  ? 'border-indigo-500 ring-2 ring-indigo-200 bg-indigo-50'
                  : 'border-gray-200'
              "
              @click="outputType = 'transcript'"
            >
              <input
                id="transcript"
                v-model="outputType"
                type="radio"
                value="transcript"
                class="sr-only"
              />
              <label for="transcript" class="flex items-center cursor-pointer">
                <div class="flex-shrink-0 text-indigo-500 mr-3">
                  <svg
                    class="h-6 w-6"
                    fill="none"
                    viewBox="0 0 24 24"
                    stroke="currentColor"
                  >
                    <path
                      stroke-linecap="round"
                      stroke-linejoin="round"
                      stroke-width="2"
                      d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"
                    />
                  </svg>
                </div>
                <div>
                  <p class="font-medium text-gray-900">Transcript</p>
                  <p class="text-xs text-gray-500">
                    Plain text format without timestamps (.txt)
                  </p>
                </div>
              </label>
            </div>

            <div
              class="relative border rounded-lg p-4 cursor-pointer transition-all duration-200 hover:bg-gray-50"
              :class="
                outputType === 'subtitles'
                  ? 'border-indigo-500 ring-2 ring-indigo-200 bg-indigo-50'
                  : 'border-gray-200'
              "
              @click="outputType = 'subtitles'"
            >
              <input
                id="subtitles"
                v-model="outputType"
                type="radio"
                value="subtitles"
                class="sr-only"
              />
              <label for="subtitles" class="flex items-center cursor-pointer">
                <div class="flex-shrink-0 text-indigo-500 mr-3">
                  <svg
                    class="h-6 w-6"
                    fill="none"
                    viewBox="0 0 24 24"
                    stroke="currentColor"
                  >
                    <path
                      stroke-linecap="round"
                      stroke-linejoin="round"
                      stroke-width="2"
                      d="M7 8h10M7 12h4m1 8l-4-4H5a2 2 0 01-2-2V6a2 2 0 012-2h14a2 2 0 012 2v8a2 2 0 01-2 2h-3l-4 4z"
                    />
                  </svg>
                </div>
                <div>
                  <p class="font-medium text-gray-900">Subtitles</p>
                  <p class="text-xs text-gray-500">
                    Timed subtitles format (.srt)
                  </p>
                </div>
              </label>
            </div>
          </div>

          <!-- Character Length Input (only shown when subtitles is selected) -->
          <div
            v-if="outputType === 'subtitles'"
            class="mt-4 p-4 bg-indigo-50 rounded-lg border border-indigo-100"
          >
            <label
              for="character-length"
              class="block text-sm font-medium text-gray-700 mb-2"
            >
              Subtitle Character Length
            </label>
            <div class="flex items-center">
              <input
                id="character-length"
                v-model="characterLength"
                type="number"
                min="1"
                max="100"
                class="block w-24 px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"
              />
              <span class="ml-2 text-sm text-gray-500"
                >characters per line</span
              >
            </div>
            <p class="mt-1 text-xs text-gray-500">
              Maximum number of characters per subtitle line (default: 32)
            </p>
          </div>
        </div>

        <!-- Processing Progress (shows during loading) -->
        <div v-if="isLoading" class="mb-8">
          <div class="flex items-center justify-between mb-2">
            <div class="text-sm font-medium text-gray-700">
              {{ processingStage }}
            </div>
            <div class="text-sm text-indigo-600">{{ processingProgress }}%</div>
          </div>
          <div class="w-full bg-gray-200 rounded-full h-2.5">
            <div
              class="bg-indigo-600 h-2.5 rounded-full transition-all duration-300 ease-in-out"
              :style="`width: ${processingProgress}%`"
            ></div>
          </div>
        </div>

        <!-- Submit Button -->
        <div class="mt-8">
          <button
            @click="processAudio"
            :disabled="isLoading"
            class="w-full inline-flex justify-center items-center px-6 py-3 border border-transparent text-base font-medium rounded-lg shadow-sm text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 disabled:opacity-50 disabled:cursor-not-allowed transition-colors duration-200"
          >
            <span v-if="isLoading" class="mr-3">
              <svg
                class="animate-spin h-5 w-5 text-white"
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
                  d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
                ></path>
              </svg>
            </span>
            {{ isLoading ? "Processing..." : "Process Audio" }}
          </button>
          <p v-if="!isLoading" class="mt-2 text-xs text-center text-gray-500">
            Processing time depends on audio length and may take several minutes
          </p>
        </div>

        <!-- Status Messages -->
        <div class="mt-6 space-y-4">
          <!-- Error Message -->
          <div
            v-if="errorMessage"
            class="rounded-lg bg-red-50 p-4 border border-red-100"
          >
            <div class="flex">
              <div class="flex-shrink-0">
                <svg
                  class="h-5 w-5 text-red-400"
                  xmlns="http://www.w3.org/2000/svg"
                  viewBox="0 0 20 20"
                  fill="currentColor"
                  aria-hidden="true"
                >
                  <path
                    fill-rule="evenodd"
                    d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z"
                    clip-rule="evenodd"
                  />
                </svg>
              </div>
              <div class="ml-3">
                <p class="text-sm font-medium text-red-800">
                  {{ errorMessage }}
                </p>
              </div>
            </div>
          </div>

          <!-- Success Message -->
          <div
            v-if="successMessage"
            class="rounded-lg bg-green-50 p-4 border border-green-100"
          >
            <div class="flex">
              <div class="flex-shrink-0">
                <svg
                  class="h-5 w-5 text-green-400"
                  xmlns="http://www.w3.org/2000/svg"
                  viewBox="0 0 20 20"
                  fill="currentColor"
                  aria-hidden="true"
                >
                  <path
                    fill-rule="evenodd"
                    d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z"
                    clip-rule="evenodd"
                  />
                </svg>
              </div>
              <div class="ml-3">
                <p class="text-sm font-medium text-green-800">
                  {{ successMessage }}
                </p>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
