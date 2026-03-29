<template>
  <q-layout view="hHh lpR fFf" class="bg-dark text-white app-layout">
    <q-page-container>
      <q-page class="window-height flex flex-center q-pa-md">
        <div class="full-width column q-gutter-y-lg" style="max-width: 1200px">
          <q-card
            class="bg-grey-10 text-white flat bordered-custom rounded-custom q-pa-lg"
          >
            <q-card-section class="row q-col-gutter-lg items-center">
              <div class="col-12 col-md-6 column q-gutter-y-md">
                <div class="text-h6 text-grey-4 q-mb-xs">
                  Session Parameters
                </div>
                <div class="row q-gutter-x-md">
                  <q-input
                    v-model="callsign"
                    label="Your Callsign"
                    dark
                    outlined
                    class="col"
                    :disable="isRunning"
                    :rules="[
                      (val) =>
                        /^VK[0-9][A-Z]{1,4}$/i.test(val) || 'VK calls only',
                    ]"
                    autocapitalize="characters"
                  />
                  <q-select
                    v-model="selectedScript"
                    :options="scripts"
                    option-label="title"
                    label="Practice Script"
                    dark
                    outlined
                    class="col"
                    :disable="isRunning"
                  />
                </div>
                <div class="row q-gutter-x-lg">
                  <div class="col">
                    <div class="text-caption text-grey-5">
                      Char Speed: {{ charWpm }} WPM
                    </div>
                    <q-slider
                      v-model="charWpm"
                      :min="10"
                      :max="40"
                      :step="1"
                      dark
                      color="primary"
                      :disable="isRunning"
                    />
                  </div>
                  <div class="col">
                    <div class="text-caption text-grey-5">
                      Effective Speed: {{ effWpm }} WPM
                    </div>
                    <q-slider
                      v-model="effWpm"
                      :min="5"
                      :max="charWpm"
                      :step="1"
                      dark
                      color="primary"
                      :disable="isRunning"
                    />
                  </div>
                </div>
              </div>

              <div class="col-12 col-md-4 column q-gutter-y-md">
                <div class="text-h6 text-grey-4 q-mb-xs">Live Tuning</div>
                <div>
                  <div class="text-caption text-grey-5">
                    Tone Frequency: {{ toneFreq }} Hz
                  </div>
                  <q-slider
                    v-model="toneFreq"
                    :min="400"
                    :max="1000"
                    :step="10"
                    dark
                    color="secondary"
                  />
                </div>
                <div>
                  <div class="text-caption text-grey-5">
                    Tolerance (±): {{ (tolerance * 100).toFixed(0) }}%
                  </div>
                  <q-slider
                    v-model="tolerance"
                    :min="0.1"
                    :max="0.8"
                    :step="0.05"
                    dark
                    color="negative"
                  />
                </div>
              </div>

              <div class="col-12 col-md-2 flex flex-center">
                <q-btn
                  v-if="!isRunning"
                  size="xl"
                  color="primary"
                  class="full-width rounded-custom text-weight-bold"
                  label="START"
                  @click="startSession"
                  :disable="!isValidSetup"
                />
                <q-btn
                  v-else
                  size="xl"
                  color="negative"
                  class="full-width rounded-custom text-weight-bold"
                  label="STOP"
                  @click="stopSession"
                />
              </div>
            </q-card-section>
          </q-card>

          <div
            v-if="appState === 'RUNNING'"
            class="column items-center full-width q-gutter-y-lg"
          >
            <q-card
              class="full-width bg-grey-10 flat bordered-custom rounded-custom"
            >
              <q-card-section
                class="text-h4 text-center script-container q-pa-lg"
              >
                <div
                  v-for="(line, index) in compiledScript"
                  :key="index"
                  :class="{
                    'text-grey-7': index < currentLineIndex,
                    'text-white': index === currentLineIndex,
                    'text-grey-8': index > currentLineIndex,
                  }"
                  class="q-my-sm transition-colors"
                >
                  <span
                    v-if="line.speaker === 'RX'"
                    class="text-h5 q-mr-md text-secondary"
                    >[RX]</span
                  >
                  <span v-else class="text-h5 q-mr-md text-primary">[TX]</span>

                  <span v-if="index === currentLineIndex">
                    <span
                      v-for="(char, charIdx) in line.text"
                      :key="charIdx"
                      :class="getCharStatusClass(charIdx)"
                    >
                      {{ char }}
                    </span>
                  </span>
                  <span v-else>{{ line.text }}</span>
                </div>
              </q-card-section>
            </q-card>

            <q-card
              class="full-width bg-grey-10 flat bordered-custom rounded-custom q-pa-xl text-center relative-position"
            >
              <div class="text-h5 q-mb-lg text-grey-4">
                <span v-if="isUserTurn">Your Turn to Key: </span>
                <span v-else class="text-secondary text-weight-bold"
                  >Receiving Transmission:
                </span>
                <strong class="text-uppercase text-white text-h2 q-ml-sm">{{
                  currentChar === " " ? "SPACE" : currentChar
                }}</strong>
              </div>

              <div
                class="row justify-center items-center q-gutter-x-md"
                style="min-height: 80px"
              >
                <div
                  v-for="(element, elIdx) in currentExpectedElements"
                  :key="elIdx"
                  class="morse-element-container bg-grey-9 rounded-custom overflow-hidden"
                  :style="{
                    width: element === '.' ? '60px' : '160px',
                    height: '40px',
                  }"
                >
                  <div
                    class="morse-element-fill full-height"
                    :class="getElementFillColor(elIdx)"
                    :style="{ width: getElementFillWidth(elIdx) + '%' }"
                  ></div>
                </div>
              </div>

              <div class="q-mt-xl text-h6 text-grey-6">
                <span v-if="isUserTurn"
                  >Tap or hold SPACEBAR, or touch here to key.</span
                >
                <span v-else class="text-secondary"
                  >Listen to the timing and cadence...</span
                >
              </div>

              <div
                v-if="isUserTurn"
                class="absolute-top-left full-width full-height z-top cursor-pointer"
                @mousedown="handleKeydown"
                @mouseup="handleKeyup"
                @touchstart.prevent="handleKeydown"
                @touchend.prevent="handleKeyup"
              ></div>
            </q-card>
          </div>

          <q-card
            v-if="appState === 'FINISHED'"
            class="bg-grey-10 flat bordered-custom rounded-custom q-pa-xl text-center"
          >
            <h2 class="text-h2 q-mb-md text-white">Session Complete</h2>
            <div class="text-h3 q-mb-lg">
              Grade:
              <span :class="gradeColor" class="text-weight-bold">{{
                finalGrade
              }}</span>
            </div>
            <p class="text-h5 text-grey-5">
              Accuracy:
              {{
                totalChars > 0 ? ((score / totalChars) * 100).toFixed(1) : 0
              }}%
            </p>
          </q-card>
        </div>
      </q-page>
    </q-page-container>
  </q-layout>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch } from "vue";

// --- Types & Dictionary ---
type AppState = "IDLE" | "RUNNING" | "FINISHED";
type Speaker = "TX" | "RX";
interface ScriptLine {
  speaker: Speaker;
  text: string;
}
interface ScriptDef {
  id: string;
  title: string;
  lines: ScriptLine[];
}

const morseDict: Record<string, string> = {
  A: ".-",
  B: "-...",
  C: "-.-.",
  D: "-..",
  E: ".",
  F: "..-.",
  G: "--.",
  H: "....",
  I: "..",
  J: ".---",
  K: "-.-",
  L: ".-..",
  M: "--",
  N: "-.",
  O: "---",
  P: ".--.",
  Q: "--.-",
  R: ".-.",
  S: "...",
  T: "-",
  U: "..-",
  V: "...-",
  W: ".--",
  X: "-..-",
  Y: "-.--",
  Z: "--..",
  "0": "-----",
  "1": ".----",
  "2": "..---",
  "3": "...--",
  "4": "....-",
  "5": ".....",
  "6": "-....",
  "7": "--...",
  "8": "---..",
  "9": "----.",
  "/": "-..-.",
  "?": "..--..",
};

// --- Scripts Data ---
const scripts: ScriptDef[] = [
  {
    id: "gen-cq",
    title: "General CQ (Grid Square)",
    lines: [
      { speaker: "TX", text: "CQ CQ CQ DE {CALL} {CALL} K" },
      { speaker: "RX", text: "{CALL} DE VK4XYZ UR 599 599 BK" },
      { speaker: "TX", text: "VK4XYZ TU UR 599 599 QTH QG62 BK" },
      { speaker: "RX", text: "TU 73 EE" },
    ],
  },
  {
    id: "pota",
    title: "POTA Operation",
    lines: [
      { speaker: "TX", text: "CQ POTA CQ POTA DE {CALL} {CALL} K" },
      { speaker: "RX", text: "{CALL} DE VK2POTA UR 599 599 BK" },
      { speaker: "TX", text: "VK2POTA TU UR 599 599 PARK VK-0001 73 EE" },
    ],
  },
];

// --- State ---
const appState = ref<AppState>("IDLE");
const callsign = ref("VK4TEST");
const selectedScript = ref<ScriptDef>(scripts[0]);
const charWpm = ref(20);
const effWpm = ref(10);
const toneFreq = ref(700);
const tolerance = ref(0.3); // 30% timing tolerance

const compiledScript = ref<ScriptLine[]>([]);
const currentLineIndex = ref(0);
const currentCharIndex = ref(0);
const currentElementIndex = ref(0);
const isUserTurn = ref(false);

const charStatuses = ref<("pending" | "active" | "passed" | "failed")[]>([]);
const elementFills = ref<number[]>([]);
const elementResults = ref<("pending" | "passed" | "failed" | "rx")[]>([]);
const score = ref(0);
const totalChars = ref(0);

// Audio & Timing Refs
let audioCtx: AudioContext | null = null;
let oscillator: OscillatorNode | null = null;
let gainNode: GainNode | null = null;
let keydownTime = 0;
let animationFrameId: number | null = null;
let isKeyDown = false;

// --- Computed ---
const isRunning = computed(() => appState.value === "RUNNING");
const isValidSetup = computed(
  () => /^VK[0-9][A-Z]{1,4}$/i.test(callsign.value) && selectedScript.value,
);

const currentChar = computed(() => {
  if (!compiledScript.value[currentLineIndex.value]) return "";
  return (
    compiledScript.value[currentLineIndex.value].text[currentCharIndex.value] ||
    ""
  );
});

const currentExpectedElements = computed(() => {
  const char = currentChar.value.toUpperCase();
  const code = morseDict[char];
  return code ? code.split("") : [];
});

const dotMs = computed(() => 1200 / charWpm.value);
const dashMs = computed(() => dotMs.value * 3);

const finalGrade = computed(() => {
  if (totalChars.value === 0) return "E";
  const pct = score.value / totalChars.value;
  if (pct >= 0.95) return "S";
  if (pct >= 0.9) return "A";
  if (pct >= 0.8) return "B";
  if (pct >= 0.7) return "C";
  if (pct >= 0.6) return "D";
  return "E";
});
const gradeColor = computed(
  () =>
    ({
      S: "text-purple-4",
      A: "text-green-4",
      B: "text-blue-4",
      C: "text-orange-4",
      D: "text-deep-orange-4",
      E: "text-red-5",
    })[finalGrade.value],
);

// --- Audio Engine ---
const initAudio = () => {
  if (!audioCtx)
    audioCtx = new (
      window.AudioContext || (window as any).webkitAudioContext
    )();
};

const startTone = () => {
  if (!audioCtx) return;
  if (audioCtx.state === "suspended") audioCtx.resume();

  oscillator = audioCtx.createOscillator();
  gainNode = audioCtx.createGain();
  oscillator.type = "sine";
  oscillator.frequency.value = toneFreq.value;
  gainNode.gain.setValueAtTime(0, audioCtx.currentTime);
  gainNode.gain.setTargetAtTime(1, audioCtx.currentTime, 0.005);
  oscillator.connect(gainNode);
  gainNode.connect(audioCtx.destination);
  oscillator.start();
};

const stopTone = () => {
  if (!audioCtx || !gainNode || !oscillator) return;
  gainNode.gain.setTargetAtTime(0, audioCtx.currentTime, 0.005);
  const oscToStop = oscillator;
  setTimeout(() => oscToStop.stop(), 50);
  oscillator = null;
  gainNode = null;
};

// Live update tone freq if running
watch(toneFreq, (newFreq) => {
  if (oscillator)
    oscillator.frequency.setValueAtTime(newFreq, audioCtx!.currentTime);
});

// --- Game Logic ---
const startSession = () => {
  initAudio();
  compiledScript.value = selectedScript.value.lines.map((line) => ({
    speaker: line.speaker,
    text: line.text.replace(/\{CALL\}/g, callsign.value.toUpperCase()),
  }));
  currentLineIndex.value = 0;
  score.value = 0;
  totalChars.value = 0;
  appState.value = "RUNNING";
  processLine();
};

const stopSession = () => {
  appState.value = "IDLE";
  stopTone();
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
};

const processLine = () => {
  if (appState.value !== "RUNNING") return;
  if (currentLineIndex.value >= compiledScript.value.length) {
    appState.value = "FINISHED";
    return;
  }
  const line = compiledScript.value[currentLineIndex.value];
  if (line.speaker === "TX") {
    isUserTurn.value = true;
    currentCharIndex.value = 0;
    setupCharState();
  } else {
    isUserTurn.value = false;
    playRXLine();
  }
};

const setupCharState = () => {
  const line = compiledScript.value[currentLineIndex.value];

  if (isUserTurn.value) {
    // Skip spaces automatically during TX
    while (
      currentCharIndex.value < line.text.length &&
      line.text[currentCharIndex.value] === " "
    ) {
      currentCharIndex.value++;
    }
    if (currentCharIndex.value >= line.text.length) {
      currentLineIndex.value++;
      setTimeout(processLine, 1000);
      return;
    }
  }

  if (charStatuses.value.length !== line.text.length) {
    charStatuses.value = Array(line.text.length).fill("pending");
  }
  charStatuses.value[currentCharIndex.value] = "active";

  const expectedCount = currentExpectedElements.value.length;
  currentElementIndex.value = 0;
  elementFills.value = Array(expectedCount).fill(0);
  elementResults.value = Array(expectedCount).fill("pending");
};

// --- Keying Handlers (TX) ---
const handleGlobalKeydown = (e: KeyboardEvent) => {
  if (e.repeat) return;
  if (e.code === "Space" && isRunning.value && isUserTurn.value) {
    e.preventDefault();
    handleKeydown();
  }
};
const handleGlobalKeyup = (e: KeyboardEvent) => {
  if (e.code === "Space" && isRunning.value && isUserTurn.value) {
    e.preventDefault();
    handleKeyup();
  }
};

const handleKeydown = () => {
  if (isKeyDown || !isUserTurn.value) return;
  isKeyDown = true;
  keydownTime = performance.now();
  startTone();
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  animationFrameId = requestAnimationFrame(updateVisualizer);
};

const handleKeyup = () => {
  if (!isKeyDown || !isUserTurn.value) return;
  isKeyDown = false;
  stopTone();
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  evaluateElement();
};

const updateVisualizer = () => {
  if (!isKeyDown) return;
  const elapsed = performance.now() - keydownTime;
  const expectedElement =
    currentExpectedElements.value[currentElementIndex.value];
  const targetMs = expectedElement === "." ? dotMs.value : dashMs.value;

  let pct = (elapsed / targetMs) * 100;
  elementFills.value[currentElementIndex.value] = Math.min(pct, 100);

  // Fail-safe timeout: only auto-kill if held absurdly long (3x target duration)
  if (elapsed > targetMs * 3) {
    handleKeyup();
  } else {
    animationFrameId = requestAnimationFrame(updateVisualizer);
  }
};

const evaluateElement = () => {
  const elapsed = performance.now() - keydownTime;
  const expectedElement =
    currentExpectedElements.value[currentElementIndex.value];
  const targetMs = expectedElement === "." ? dotMs.value : dashMs.value;

  // Mathematical threshold logic (works for both too short and too long)
  const errorRatio = Math.abs(elapsed - targetMs) / targetMs;
  const passed = errorRatio <= tolerance.value;

  elementResults.value[currentElementIndex.value] = passed
    ? "passed"
    : "failed";
  currentElementIndex.value++;

  // Check if character is fully keyed
  if (currentElementIndex.value >= currentExpectedElements.value.length) {
    const allElementsPassed = elementResults.value.every(
      (res) => res === "passed",
    );
    charStatuses.value[currentCharIndex.value] = allElementsPassed
      ? "passed"
      : "failed";

    if (allElementsPassed) score.value++;
    totalChars.value++;
    setTimeout(advanceToNextChar, 300);
  }
};

const advanceToNextChar = () => {
  currentCharIndex.value++;
  setupCharState();
};

// --- RX Simulation ---
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

const playRXLine = async () => {
  const line = compiledScript.value[currentLineIndex.value];
  const farnsworthRatio = charWpm.value / effWpm.value;

  for (let i = 0; i < line.text.length; i++) {
    if (appState.value !== "RUNNING") return; // Abort if stopped mid-RX

    currentCharIndex.value = i;
    const char = line.text[i];

    if (char === " ") {
      await sleep(dashMs.value * farnsworthRatio * 2); // Word space
      continue;
    }

    const code = morseDict[char];
    if (!code) continue;

    setupCharState();

    for (let e = 0; e < code.length; e++) {
      if (appState.value !== "RUNNING") return;
      currentElementIndex.value = e;
      const duration = code[e] === "." ? dotMs.value : dashMs.value;

      startTone();
      elementResults.value[e] = "rx";
      await playRXElementFill(duration, e);
      stopTone();

      await sleep(dotMs.value); // Standard gap between elements
    }

    charStatuses.value[i] = "passed";
    await sleep(dashMs.value * farnsworthRatio); // Farnsworth gap between chars
  }

  currentLineIndex.value++;
  processLine();
};

const playRXElementFill = (durationMs: number, index: number) => {
  return new Promise<void>((resolve) => {
    const start = performance.now();
    const animate = (time: number) => {
      if (appState.value !== "RUNNING") return resolve();
      const elapsed = time - start;
      elementFills.value[index] = Math.min((elapsed / durationMs) * 100, 100);
      if (elapsed < durationMs) requestAnimationFrame(animate);
      else resolve();
    };
    requestAnimationFrame(animate);
  });
};

// --- Helper Functions for UI ---
const getCharStatusClass = (idx: number) => {
  const status = charStatuses.value[idx];
  if (status === "active")
    return "text-primary bg-grey-9 rounded-borders q-px-sm py-xs";
  if (status === "passed") return "text-green-4";
  if (status === "failed") return "text-red-5";
  return "text-grey-7";
};

const getElementFillColor = (idx: number) => {
  const res = elementResults.value[idx];
  if (res === "passed") return "bg-green-5";
  if (res === "failed") return "bg-red-5";
  if (res === "rx") return "bg-secondary"; // Teal for RX playback
  return "bg-primary";
};

const getElementFillWidth = (idx: number) => elementFills.value[idx] || 0;

// --- Lifecycle ---
onMounted(() => {
  window.addEventListener("keydown", handleGlobalKeydown);
  window.addEventListener("keyup", handleGlobalKeyup);
});
onUnmounted(() => {
  window.removeEventListener("keydown", handleGlobalKeydown);
  window.removeEventListener("keyup", handleGlobalKeyup);
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
});
</script>

<style scoped>
/* Dark Mode Overrides and Layout Styling */
.app-layout {
  font-family: "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
}
.rounded-custom {
  border-radius: 24px !important;
}
.bordered-custom {
  border: 1px solid rgba(255, 255, 255, 0.08);
}
.transition-colors {
  transition: all 0.3s ease;
}
.script-container {
  font-family: "Courier New", Courier, monospace;
  font-weight: bold;
  line-height: 1.8;
  letter-spacing: 2px;
}
.morse-element-container {
  position: relative;
  box-shadow: inset 0 2px 8px rgba(0, 0, 0, 0.3);
}
.morse-element-fill {
  transition: background-color 0.2s;
  border-radius: inherit;
}
</style>
