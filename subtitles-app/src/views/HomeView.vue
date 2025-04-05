<script setup lang="ts">
import AudioProcessor from '../components/AudioProcessor.vue'
</script>

<template>
  <div class="py-6 md:py-12">
    <div class="max-w-6xl mx-auto px-4 sm:px-6 lg:px-8">
      <div class="text-center mb-12">
        <h2 class="text-sm font-semibold text-indigo-600 uppercase tracking-wide">
          Made with ❤️  by Monetizely using AssemblyAI
        </h2>
        <p class="mt-2 text-4xl md:text-5xl font-extrabold text-gray-900 tracking-tight">
          Audio Transcription Made Simple
        </p>
        <p class="mt-5 max-w-xl mx-auto text-xl text-gray-500">
          Transform your audio files to accurate text transcripts or timed subtitles with just a few clicks.
        </p>
      </div>
      
      <div class="grid grid-cols-1 lg:grid-cols-5 gap-8">
        <!-- Features sidebar -->
        <div class="lg:col-span-2">
          <div class="bg-white rounded-xl shadow-sm border border-gray-100 p-6 md:p-8">
            <h3 class="text-lg font-semibold text-gray-900 mb-6 flex items-center">
              <svg class="w-6 h-6 text-indigo-500 mr-2" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z" />
              </svg>
              How It Works
            </h3>
            
            <ul class="space-y-5">
              <li class="flex">
                <div class="flex-shrink-0">
                  <div class="flex items-center justify-center h-8 w-8 rounded-full bg-indigo-100 text-indigo-600 font-semibold text-sm">
                    1
                  </div>
                </div>
                <div class="ml-4">
                  <h4 class="text-base font-medium text-gray-900">Enter Your API Key</h4>
                  <p class="mt-1 text-sm text-gray-500">Get your key from AssemblyAI and enter it securely</p>
                </div>
              </li>
              
              <li class="flex">
                <div class="flex-shrink-0">
                  <div class="flex items-center justify-center h-8 w-8 rounded-full bg-indigo-100 text-indigo-600 font-semibold text-sm">
                    2
                  </div>
                </div>
                <div class="ml-4">
                  <h4 class="text-base font-medium text-gray-900">Upload Your Audio</h4>
                  <p class="mt-1 text-sm text-gray-500">Select any audio file from your device</p>
                </div>
              </li>
              
              <li class="flex">
                <div class="flex-shrink-0">
                  <div class="flex items-center justify-center h-8 w-8 rounded-full bg-indigo-100 text-indigo-600 font-semibold text-sm">
                    3
                  </div>
                </div>
                <div class="ml-4">
                  <h4 class="text-base font-medium text-gray-900">Choose Output Format</h4>
                  <p class="mt-1 text-sm text-gray-500">Get plain text or timed subtitles (SRT format)</p>
                </div>
              </li>
              
              <li class="flex">
                <div class="flex-shrink-0">
                  <div class="flex items-center justify-center h-8 w-8 rounded-full bg-indigo-100 text-indigo-600 font-semibold text-sm">
                    4
                  </div>
                </div>
                <div class="ml-4">
                  <h4 class="text-base font-medium text-gray-900">Download Results</h4>
                  <p class="mt-1 text-sm text-gray-500">Your transcribed text downloads automatically</p>
                </div>
              </li>
            </ul>
            
            <div class="mt-8 p-4 bg-indigo-50 rounded-lg border border-indigo-100">
              <div class="flex items-start">
                <div class="flex-shrink-0">
                  <svg class="h-5 w-5 text-indigo-400" viewBox="0 0 20 20" fill="currentColor">
                    <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7-4a1 1 0 11-2 0 1 1 0 012 0zM9 9a1 1 0 000 2v3a1 1 0 001 1h1a1 1 0 100-2v-3a1 1 0 00-1-1H9z" clip-rule="evenodd" />
                  </svg>
                </div>
                <div class="ml-3">
                  <h4 class="text-sm font-medium text-indigo-800">Need an API key?</h4>
                  <p class="mt-1 text-xs text-indigo-700">
                    <a href="https://www.assemblyai.com/dashboard/signup" target="_blank" class="font-medium underline hover:text-indigo-600">
                      Sign up for AssemblyAI
                    </a>
                    to get your free API key with 5 hours of free transcription each month.
                  </p>
                </div>
              </div>
            </div>
          </div>
        </div>
        
        <!-- Main processor -->
        <div class="lg:col-span-3">
          <AudioProcessor />
        </div>
      </div>
    </div>
  </div>
</template>