<script lang="ts">
  import { writable } from "svelte/store";
  import { GoogleGenAI } from "@google/genai";
  import { encode } from "@stablelib/base64";
  import { set, get, del, createStore } from "idb-keyval";
  // Import Iconify (not actually needed as we're using web components)

  // Type definitions
  interface HistoryItem {
    id: number;
    text: string;
    date: string;
    audioURL: string;
  }

  interface TranscriptionResponse {
    transcript: AsyncIterable<string>;
  }

  // Stores
  const apiKey = writable(localStorage.getItem("geminiApiKey") || "");
  const recording = writable(false);
  const isProcessing = writable(false);
  const transcript = writable("");
  const history = writable<HistoryItem[]>([]);
  const showHistoryModal = writable(false);
  const showSettingsModal = writable(false);
  const audioURL = writable("");
  const isStartingRecording = writable(false); // Add state for recording initialization

  // Load history on mount
  const loadHistory = async () => {
    try {
      const stored = (await get("transcriptHistory")) || [];
      history.set(stored);
    } catch (error) {
      console.error("Failed to load history:", error);
    }
  };

  loadHistory();

  // Clean up old recordings (older than 30 days)
  const cleanupOldRecordings = async () => {
    const stored = (await get<HistoryItem[]>("transcriptHistory")) || [];
    const thirtyDaysAgo = new Date();
    thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

    const filtered = stored.filter(
      (item) => new Date(item.date) > thirtyDaysAgo
    );
    if (filtered.length !== stored.length) {
      await set("transcriptHistory", filtered);
      history.set(filtered);
    }
  };

  cleanupOldRecordings();

  // Save API key to localStorage
  apiKey.subscribe((value) => {
    localStorage.setItem("geminiApiKey", value);
  });

  // Media recording
  let mediaRecorder: MediaRecorder | null = null;
  let audioChunks: Blob[] = [];
  let stream: MediaStream | null = null;
  let shouldProcessRecording = true; // Flag to control whether to process recording

  let currentVoiceLevel = 0;
  let durationSeconds = 0;
  let durationInterval: number | null = null;

  // Add new state for showing copy success
  let showCopySuccess = false;

  let dragOver = false; // Add state for drag over indication

  // Handle keyboard shortcuts
  const handleKeydown = (event: KeyboardEvent) => {
    // Skip if user is typing in an input or textarea
    if (event.target instanceof HTMLInputElement || event.target instanceof HTMLTextAreaElement) {
      return;
    }
    
    // Enter key - Start/stop recording
    if (event.key === 'Enter' && !$isProcessing && $apiKey) {
      event.preventDefault();
      $recording ? stopRecording() : startRecording();
    }
    
    // Escape key - Cancel recording
    if (event.key === 'Escape' && $recording) {
      event.preventDefault();
      cancelRecording();
    }
    
    // Ctrl+C or Command+C - Copy transcript
    if ((event.ctrlKey || event.metaKey) && event.key === 'c' && $transcript) {
      event.preventDefault();
      copyTranscript();
    }
  };

  // Add and remove event listener for keyboard shortcuts
  import { onMount, onDestroy } from 'svelte';
  
  onMount(() => {
    window.addEventListener('keydown', handleKeydown);
  });
  
  onDestroy(() => {
    window.removeEventListener('keydown', handleKeydown);
  });

  const calculatePeakLevel = (data: Uint8Array) => {
    let peak = 0;
    for (let i = 0; i < data.length; i++) {
      const normalizedValue = Math.abs((data[i] - 128) / 128);
      peak = Math.max(peak, normalizedValue);
    }
    return peak;
  };

  const analyseAudio = (audioStream: MediaStream) => {
    const audioContext = new AudioContext();
    const audioStreamSource = audioContext.createMediaStreamSource(audioStream);

    const analyser = audioContext.createAnalyser();
    audioStreamSource.connect(analyser);

    const timeDomainData = new Uint8Array(analyser.fftSize);

    const detectSound = () => {
      const processFrame = () => {
        if (!$recording) return;

        analyser.getByteTimeDomainData(timeDomainData);

        const peakLevel =
          1 - Math.pow(1 - calculatePeakLevel(timeDomainData), 2);
        currentVoiceLevel = peakLevel;

        window.requestAnimationFrame(processFrame);
      };

      window.requestAnimationFrame(processFrame);
    };

    detectSound();
  };

  const startRecording = async () => {
    try {
      // Check for API key first
      if (!$apiKey) {
        $showSettingsModal = true;
        return;
      }

      // Set initializing state to true immediately for UI feedback
      $isStartingRecording = true;

      shouldProcessRecording = true;
      stream = await navigator.mediaDevices.getUserMedia({
        audio: true,
      });

      mediaRecorder = new MediaRecorder(stream);
      audioChunks = [];

      // Start duration counter
      durationSeconds = 0;
      durationInterval = window.setInterval(() => {
        durationSeconds++;
      }, 1000);

      mediaRecorder.addEventListener("dataavailable", (event: BlobEvent) => {
        audioChunks.push(event.data);
      });

      mediaRecorder.addEventListener("stop", async () => {
        if (audioChunks.length > 0 && shouldProcessRecording) {
          const audioBlob = new Blob(audioChunks, { type: "audio/wav" });
          $audioURL = URL.createObjectURL(audioBlob);

          if ($apiKey) {
            $isProcessing = true;
            try {
              await processTranscription(audioBlob);
            } catch (error: unknown) {
              alert(
                "Error transcribing audio: " +
                  (error instanceof Error ? error.message : String(error))
              );
            } finally {
              $isProcessing = false;
            }
          } else {
            alert("Please set your Gemini API key in settings first");
            $showSettingsModal = true;
          }
        }
      });

      mediaRecorder.start();
      $recording = true;
      analyseAudio(stream);
    } catch (error: unknown) {
      console.error("Error accessing microphone:", error);
      alert(
        "Could not access microphone. Please ensure you have granted permission."
      );
    } finally {
      $isStartingRecording = false; // Reset initializing state
    }
  };

  const stopRecording = () => {
    if (mediaRecorder && $recording) {
      mediaRecorder.stop();
      $recording = false;

      if (stream) {
        stream.getTracks().forEach((track) => track.stop());
      }

      clearDurationCounter();
    }
  };

  const cancelRecording = () => {
    if (mediaRecorder && $recording) {
      // Set flag to prevent processing
      shouldProcessRecording = false;

      // Stop the recording
      mediaRecorder.stop();
      $recording = false;
      audioChunks = [];

      if (stream) {
        stream.getTracks().forEach((track) => track.stop());
      }

      clearDurationCounter();
    }
  };

  const clearDurationCounter = () => {
    if (durationInterval) {
      window.clearInterval(durationInterval);
      durationInterval = null;
    }
  };

  const formatDuration = (seconds: number): string => {
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = seconds % 60;
    return `${minutes}:${remainingSeconds < 10 ? "0" : ""}${remainingSeconds}`;
  };

  // Mock implementation of transcribeAudio
  const transcribeAudio = async (
    apiKey: string,
    audioBlob: Blob
  ): Promise<TranscriptionResponse> => {
    const ai = new GoogleGenAI({ apiKey });
    const response = await ai.models.generateContentStream({
      model: "gemini-1.5-pro-002",
      contents: [
        {
          inlineData: {
            mimeType: "audio/wav",
            data: encode(new Uint8Array(await audioBlob.arrayBuffer())),
          },
        },
        {
          text: 'Please transcribe any speech in this audio and respond with the transcribed text in its original language. If there is no clear speech, respond with "No speech detected".',
        },
      ],
    });
    const streamText = async function* (): AsyncGenerator<string> {
      for await (const chunk of response) {
        yield chunk.text || "";
      }
    };
    return {
      transcript: streamText(),
    };
  };

  const processTranscription = async (audioData: Blob | File) => {
    const response = await transcribeAudio($apiKey, audioData);

    let fullTranscript = "";
    $transcript = "";

    // Process streaming response
    for await (const chunk of response.transcript) {
      fullTranscript += chunk;
      $transcript = fullTranscript;
    }

    // Save to history
    const newHistoryItem: HistoryItem = {
      id: Date.now(),
      text: fullTranscript,
      date: new Date().toISOString(),
      audioURL: $audioURL,
    };

    const currentHistory =
      (await get<HistoryItem[]>("transcriptHistory")) || [];
    const updatedHistory = [newHistoryItem, ...currentHistory];
    await set("transcriptHistory", updatedHistory);
    history.set(updatedHistory);
  };

  const copyTranscript = () => {
    if (!$transcript) return;
    navigator.clipboard.writeText($transcript);
    // Replace alert with temporary checkmark
    showCopySuccess = true;
    setTimeout(() => {
      showCopySuccess = false;
    }, 2000);
  };

  const downloadRecording = () => {
    if (!$audioURL) return;
    const a = document.createElement("a");
    document.body.appendChild(a);
    a.style = "display: none";
    a.href = $audioURL;
    a.download = `recording-${new Date().toISOString()}.wav`;
    a.click();
    window.URL.revokeObjectURL($audioURL);
  };

  const resetRecording = () => {
    $transcript = "";
    $audioURL = "";
  };

  const deleteHistoryItem = async (id: number) => {
    const currentHistory =
      (await get<HistoryItem[]>("transcriptHistory")) || [];
    const updatedHistory = currentHistory.filter((item) => item.id !== id);
    await set("transcriptHistory", updatedHistory);
    history.set(updatedHistory);
  };

  const clearHistory = async () => {
    if (confirm("Are you sure you want to clear all history?")) {
      await set("transcriptHistory", []);
      history.set([]);
    }
  };

  // Close modal when clicking outside
  const handleModalOutsideClick = (
    event: MouseEvent,
    closeAction: () => void
  ) => {
    if (event.target === event.currentTarget) {
      closeAction();
    }
  };

  const processDroppedFile = async (file: File) => {
    if (!file.type.startsWith('audio/')) {
      alert('Please drop an audio file');
      return;
    }
    
    if (!$apiKey) {
      alert("Please set your Gemini API key in settings first");
      $showSettingsModal = true;
      return;
    }
    
    $audioURL = URL.createObjectURL(file);
    $isProcessing = true;
    
    try {
      await processTranscription(file);
    } catch (error: unknown) {
      alert(
        "Error transcribing audio: " +
          (error instanceof Error ? error.message : String(error))
      );
    } finally {
      $isProcessing = false;
    }
  };
  
  const handleDragOver = (event: DragEvent) => {
    event.preventDefault();
    event.stopPropagation();
    dragOver = true;
  };
  
  const handleDragLeave = (event: DragEvent) => {
    event.preventDefault();
    event.stopPropagation();
    
    // Only set dragOver to false if we're leaving the target element, not its children
    if (event.target === event.currentTarget) {
      dragOver = false;
    }
  };
  
  const handleDrop = (event: DragEvent) => {
    event.preventDefault();
    event.stopPropagation();
    dragOver = false;
    
    if (event.dataTransfer?.files.length) {
      const file = event.dataTransfer.files[0];
      processDroppedFile(file);
    }
  };
</script>

<main
  class="min-h-screen bg-[#353433] text-[#e9e8e7] flex flex-col items-center p-4"
>
  <div class="w-full max-w-3xl mx-auto md:mt-[max(0px,50vh-240px)]">
    <header class="flex justify-between items-center mb-6">
      <h1 class="text-3xl font-bold text-[#8b8685]">Voice to Text</h1>
      <div class="flex gap-2 text-[#8b8685]">
        <button
          class="p-2 bg-[#252423] rounded-full hover:bg-[#1d1c1b] flex"
          on:click={() => ($showHistoryModal = true)}
          title="View transcription history"
          aria-label="View transcription history"
        >
          <iconify-icon icon="bi:clock-history" width="24" height="24"
          ></iconify-icon>
        </button>
        <button
          class="p-2 bg-[#252423] rounded-full hover:bg-[#1d1c1b] flex"
          on:click={() => ($showSettingsModal = true)}
          title="Settings"
          aria-label="Settings"
        >
          <iconify-icon icon="bi:gear" width="24" height="24"></iconify-icon>
        </button>
      </div>
    </header>

    <div class="bg-[#252423] p-6 rounded-lg shadow-md border border-[#656463]">
      <div class="flex flex-col md:flex-row gap-6">
        <!-- Left side - Recording controls -->
        <div class="w-full md:w-1/4 flex flex-col items-center justify-center">
          <div
            class="{$recording
              ? 'text-[#d7fc70]'
              : 'text-[#8b8685]'} font-medium text-xl h-8 mb-2"
          >
            {#if $recording}
              {formatDuration(durationSeconds)}
            {:else}
              --:--
            {/if}
          </div>

          <div class="relative">
            <!-- Record/Stop button - Wrap in a container to better handle drag events -->
            <div 
              class="relative"
              on:dragenter={handleDragOver}
              on:dragover={handleDragOver}
              on:dragleave={handleDragLeave}
              on:drop={handleDrop}
            >
              <button
                class="p-6
                  {$recording
                  ? 'bg-[#252423] text-[#d7fc70] hover:bg-[#353433]'
                  : $isStartingRecording
                    ? 'bg-[#454443] text-[#090807] border-[#d7fc70] cursor-wait'
                    : dragOver 
                      ? 'bg-[#090807] text-white border-dashed'
                      : 'bg-[#d7fc70] text-[#090807] hover:bg-white'}
                  border-2 border-[#d7fc70]
                  rounded-full cursor-pointer
                  flex items-center justify-center shadow-lg relative z-10
                  disabled:opacity-50 disabled:cursor-not-allowed disabled:bg-[#353433] disabled:border-[#353433] disabled:text-[#8b8685]
                "
                style="
                  box-shadow: {$recording
                  ? `0 0 0 ${currentVoiceLevel * 12}px #d7fc7055`
                  : 'none'};
                  width: 88px;
                  height: 88px;
                "
                on:click={() => ($recording ? stopRecording() : startRecording())}
                disabled={$isProcessing || !$apiKey || $isStartingRecording}
              >
                {#if $recording}
                  <iconify-icon icon="bi:stop" width="40" height="40"></iconify-icon>
                {:else if $isStartingRecording}
                  <iconify-icon icon="bi:arrow-repeat" class="animate-spin" width="40" height="40"></iconify-icon>
                {:else if dragOver}
                  <iconify-icon icon="bi:file-earmark-arrow-down" width="40" height="40"></iconify-icon>
                {:else}
                  <iconify-icon icon="bi:mic" width="40" height="40"></iconify-icon>
                {/if}
              </button>
              
              {#if dragOver}
                <div class="absolute inset-0 z-0 rounded-full border-2 border-dashed border-white animate-pulse" style="width: 88px; height: 88px;"></div>
              {/if}
            </div>
          </div>

          <div class="h-8">
            {#if $recording}
              <button
                class="mt-4 p-3 bg-red-400 text-[#e9e8e7] cursor-pointer rounded-full hover:bg-red-700 flex items-center justify-center shadow-md"
                on:click={cancelRecording}
                aria-label="Cancel recording"
              >
                <iconify-icon icon="bi:x-lg" width="20" height="20"
                ></iconify-icon>
              </button>
            {/if}
          </div>

          {#if !$apiKey}
            <p
              class="mt-4 text-amber-400 text-sm text-center flex items-center justify-center"
            >
              <iconify-icon
                icon="bi:exclamation-triangle"
                class="mr-1"
                width="16"
                height="16"
              ></iconify-icon>
              Set API key in settings first
            </p>
          {/if}
        </div>

        <!-- Right side - Transcript -->
        <div class="w-full md:w-3/4">
          <div
            class="bg-[#090807] p-4 rounded-md h-48 mb-4 overflow-y-auto relative border border-[#656463]"
          >
            {#if $isProcessing}
              <div
                class="absolute inset-0 flex items-center justify-center bg-[#090807]/80"
              >
                <div class="flex flex-col items-center">
                  <iconify-icon
                    icon="bi:arrow-repeat"
                    width="32"
                    height="32"
                    class="animate-spin text-[#d7fc70] mb-2"
                  ></iconify-icon>
                  <p class="text-[#8b8685]">Transcribing...</p>
                </div>
              </div>
            {/if}

            {#if $transcript}
              <p>{$transcript}</p>
            {:else}
              <p
                class="text-[#8b8685] text-center h-full flex items-center justify-center"
              >
                Tap the microphone button to start recording
              </p>
            {/if}
          </div>

          <div class="flex gap-2">
            <button
              class="px-3 py-2 bg-[#d7fc70] disabled:bg-[#353433] text-[#090807] disabled:text-[#e9e8e7] rounded-md hover:bg-white flex items-center disabled:opacity-50 disabled:cursor-not-allowed"
              on:click={copyTranscript}
              disabled={!$transcript}
            >
              <span class="relative mr-1 w-5 h-5 flex items-center justify-center">
                <iconify-icon
                  icon="bi:clipboard"
                  width="20"
                  height="20"
                  class="absolute transition-opacity duration-200 {showCopySuccess ? 'opacity-0' : 'opacity-100'}"
                ></iconify-icon>
                <iconify-icon
                  icon="bi:check-lg"
                  width="20"
                  height="20"
                  class="absolute transition-opacity duration-200 {showCopySuccess ? 'opacity-100' : 'opacity-0'}"
                ></iconify-icon>
              </span>
              Copy
            </button>
            <button
              class="px-3 py-2 bg-[#252423] text-[#e9e8e7] rounded-md hover:bg-[#1d1c1b] flex items-center disabled:opacity-50 disabled:cursor-not-allowed"
              on:click={downloadRecording}
              disabled={!$audioURL}
            >
              <iconify-icon
                icon="bi:download"
                width="20"
                height="20"
                class="mr-1"
              ></iconify-icon>
              Download
            </button>
            <button
              class="px-3 py-2 ml-auto bg-[#252423] text-[#e9e8e7] rounded-md hover:bg-[#1d1c1b] flex items-center disabled:opacity-50 disabled:cursor-not-allowed"
              on:click={resetRecording}
              disabled={!$transcript}
            >
              <iconify-icon
                icon="bi:arrow-counterclockwise"
                width="20"
                height="20"
                class="mr-1"
              ></iconify-icon>
              New
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>

  <footer class="mt-8 text-center text-sm text-[#8b8685]">
    <p>
      <a href="https://github.com/dtinth/voice-to-text-web">Source code on GitHub</a>
    </p>
  </footer>
</main>

<!-- Settings Modal -->
{#if $showSettingsModal}
  <div
    class="fixed inset-0 bg-black/70 flex items-center justify-center z-50 p-4"
    on:click={(e) =>
      handleModalOutsideClick(e, () => ($showSettingsModal = false))}
  >
    <div
      class="bg-[#252423] p-6 rounded-lg shadow-xl max-w-md w-full border border-[#656463]"
    >
      <div class="flex justify-between items-center mb-4">
        <h2 class="text-xl font-bold text-[#8b8685]">Settings</h2>
        <button
          class="p-1 hover:bg-[#1d1c1b] rounded-full text-[#e9e8e7] flex"
          on:click={() => ($showSettingsModal = false)}
        >
          <iconify-icon icon="bi:x-lg" width="24" height="24"></iconify-icon>
        </button>
      </div>

      <div class="mb-4">
        <label
          class="block mb-2 text-sm font-medium text-[#e9e8e7]"
          for="apiKey"
        >
          Gemini API Key
        </label>
        <input
          type="password"
          id="apiKey"
          bind:value={$apiKey}
          class="w-full p-2 border border-[#656463] rounded-md bg-[#090807] text-[#e9e8e7]"
          placeholder="Enter your Gemini API key"
        />
        <p class="mt-1 text-sm text-[#8b8685]">
          Your API key is stored locally in your browser.
        </p>
      </div>

      <button
        class="w-full px-4 py-2 bg-indigo-600 text-[#e9e8e7] rounded-md hover:bg-indigo-700"
        on:click={() => ($showSettingsModal = false)}
      >
        Save & Close
      </button>
    </div>
  </div>
{/if}

<!-- History Modal -->
{#if $showHistoryModal}
  <div
    class="fixed inset-0 bg-black/70 flex items-center justify-center z-50 p-4"
    on:click={(e) =>
      handleModalOutsideClick(e, () => ($showHistoryModal = false))}
  >
    <div
      class="bg-[#252423] p-6 rounded-lg shadow-xl max-w-2xl w-full h-[80vh] flex flex-col border border-[#656463]"
    >
      <div class="flex justify-between items-center mb-4">
        <h2 class="text-xl font-bold text-[#8b8685]">Transcription History</h2>
        <div class="flex items-center gap-2">
          <button
            class="px-3 py-1 border-red-400 border text-red-400 text-sm rounded-md hover:bg-[#353433]"
            on:click={clearHistory}
          >
            Clear All
          </button>
          <button
            class="p-1 hover:bg-[#1d1c1b] rounded-full text-[#e9e8e7] flex"
            on:click={() => ($showHistoryModal = false)}
          >
            <iconify-icon icon="bi:x-lg" width="24" height="24"></iconify-icon>
          </button>
        </div>
      </div>

      <div class="overflow-y-auto flex-1">
        {#if $history.length === 0}
          <p class="text-[#8b8685] text-center py-4">No history found</p>
        {:else}
          <div class="space-y-4">
            {#each $history as item}
              <div class="border-b border-[#656463] pb-4">
                <div class="flex justify-between text-sm text-[#8b8685] mb-1">
                  <span>{new Date(item.date).toLocaleString()}</span>
                  <div class="flex gap-2">
                    <button
                      class="hover:text-indigo-400 flex"
                      on:click={(event) => {
                        navigator.clipboard.writeText(item.text);
                        const button = event.currentTarget as HTMLElement;
                        const clipboardIcon = button.querySelector('[icon="bi:clipboard"]');
                        const checkIcon = button.querySelector('[icon="bi:check-lg"]');
                        if (clipboardIcon && checkIcon) {
                          clipboardIcon.classList.add('opacity-0');
                          clipboardIcon.classList.remove('opacity-100');
                          checkIcon.classList.add('opacity-100');
                          checkIcon.classList.remove('opacity-0');
                          setTimeout(() => {
                            clipboardIcon.classList.add('opacity-100');
                            clipboardIcon.classList.remove('opacity-0');
                            checkIcon.classList.add('opacity-0');
                            checkIcon.classList.remove('opacity-100');
                          }, 2000);
                        }
                      }}
                    >
                      <span class="relative flex items-center justify-center">
                        <iconify-icon
                          icon="bi:clipboard"
                          width="20"
                          height="20"
                          class="relative transition-opacity duration-200 opacity-100"
                        ></iconify-icon>
                        <iconify-icon
                          icon="bi:check-lg"
                          width="20"
                          height="20"
                          class="absolute transition-opacity duration-200 opacity-0"
                        ></iconify-icon>
                      </span>
                    </button>
                    <button
                      class="hover:text-red-400 flex"
                      on:click={() => deleteHistoryItem(item.id)}
                    >
                      <iconify-icon icon="bi:trash" width="20" height="20"
                      ></iconify-icon>
                    </button>
                  </div>
                </div>
                <p class="text-[#e9e8e7]">{item.text}</p>
              </div>
            {/each}
          </div>
        {/if}
      </div>
    </div>
  </div>
{/if}

<style>
  :global(body) {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
      Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
    background-color: #353433;
  }
</style>
