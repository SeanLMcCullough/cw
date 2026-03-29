<template>
  <q-layout view="hHh lpR fFf" class="bg-grey-1">
    <q-page-container>
      <q-page class="window-height flex flex-center q-pa-md">
        <div class="full-width" style="max-width: 1200px">
          <q-card v-if="appState === 'SETUP'" class="q-pa-lg shadow-2" bordered>
            <q-card-section class="text-center">
              <h4 class="text-h4 q-mt-none q-mb-md">
                VK Farnsworth CW Trainer
              </h4>
              <p class="text-subtitle1 text-grey-8">
                Configure your session parameters
              </p>
            </q-card-section>

            <q-card-section class="row q-col-gutter-md">
              <div class="col-12 col-md-6">
                <q-input
                  v-model="callsign"
                  label="Your Callsign (VK only)"
                  outlined
                  :rules="[
                    (val) =>
                      /^VK[0-9][A-Z]{1,4}$/i.test(val) ||
                      'Must be a valid VK callsign',
                  ]"
                  autocapitalize="characters"
                />
              </div>
              <div class="col-12 col-md-6">
                <q-select
                  v-model="selectedScript"
                  :options="scripts"
                  option-label="title"
                  label="Select Practice Script"
                  outlined
                />
              </div>

              <div class="col-12 col-md-6">
                <div class="text-caption">
                  Character Speed (WPM): {{ charWpm }}
                </div>
                <q-slider
                  v-model="charWpm"
                  :min="10"
                  :max="40"
                  :step="1"
                  label
                  color="primary"
                />
              </div>
              <div class="col-12 col-md-6">
                <div class="text-caption">
                  Effective Speed (WPM): {{ effWpm }}
                </div>
                <q-slider
                  v-model="effWpm"
                  :min="5"
                  :max="charWpm"
                  :step="1"
                  label
                  color="primary"
                />
              </div>

              <div class="col-12 col-md-6">
                <div class="text-caption">
                  Tone Frequency (Hz): {{ toneFreq }}
                </div>
                <q-slider
                  v-model="toneFreq"
                  :min="400"
                  :max="1000"
                  :step="10"
                  label
                  color="secondary"
                />
              </div>
              <div class="col-12 col-md-6">
                <div class="text-caption">
                  Tolerance Threshold (%): {{ tolerance * 100 }}
                </div>
                <q-slider
                  v-model="tolerance"
                  :min="0.1"
                  :max="0.5"
                  :step="0.05"
                  label
                  color="negative"
                />
              </div>
            </q-card-section>

            <q-card-actions align="center" class="q-mt-md">
              <q-btn
                size="lg"
                color="primary"
                label="Start Session"
                @click="startSession"
                :disable="!isValidSetup"
              />
            </q-card-actions>
          </q-card>

          <div
            v-if="appState === 'RUNNING'"
            class="column items-center full-width"
          >
            <q-card class="full-width q-mb-lg shadow-1" bordered>
              <q-card-section class="text-h5 text-center script-container">
                <div
                  v-for="(line, index) in compiledScript"
                  :key="index"
                  :class="{
                    'text-grey-4': index < currentLineIndex,
                    'text-primary text-weight-bold': index === currentLineIndex,
                    'text-grey-6': index > currentLineIndex,
                  }"
                  class="q-my-sm transition-colors"
                >
                  <span
                    v-if="line.speaker === 'RX'"
                    class="text-subtitle2 q-mr-sm"
                    >[RX]</span
                  >
                  <span v-else class="text-subtitle2 q-mr-sm text-secondary"
                    >[TX]</span
                  >

                  <span
                    v-if="index === currentLineIndex && line.speaker === 'TX'"
                  >
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
              v-if="isUserTurn"
              class="full-width shadow-2 q-pa-md"
              bordered
            >
              <div class="text-center text-h6 q-mb-md">
                Key Current Letter:
                <strong class="text-uppercase text-primary text-h4">{{
                  currentChar
                }}</strong>
              </div>

              <div class="row justify-center items-center q-gutter-x-sm">
                <div
                  v-for="(element, elIdx) in currentExpectedElements"
                  :key="elIdx"
                  class="morse-element-container bg-grey-3 rounded-borders overflow-hidden"
                  :style="{
                    width: element === '.' ? '40px' : '100px',
                    height: '30px',
                  }"
                >
                  <div
                    class="morse-element-fill full-height"
                    :class="getElementFillColor(elIdx)"
                    :style="{ width: getElementFillWidth(elIdx) + '%' }"
                  ></div>
                </div>
              </div>

              <div class="text-center q-mt-xl text-grey-6">
                Tap or hold SPACEBAR, or click/touch here to key.
              </div>
            </q-card>

            <q-card
              v-else
              class="full-width shadow-2 q-pa-xl text-center bg-grey-2"
              bordered
            >
              <q-spinner-bars color="primary" size="3em" />
              <div class="text-h6 q-mt-md text-grey-8">
                Receiving transmission...
              </div>
            </q-card>

            <div
              v-if="isUserTurn"
              class="fixed-center full-width full-height z-top cursor-pointer"
              style="opacity: 0"
              @mousedown="handleKeydown"
              @mouseup="handleKeyup"
              @touchstart.prevent="handleKeydown"
              @touchend.prevent="handleKeyup"
            ></div>
          </div>

          <q-card
            v-if="appState === 'FINISHED'"
            class="q-pa-lg shadow-2 text-center"
            bordered
          >
            <q-card-section>
              <h2 class="text-h2 q-mb-sm">Session Complete</h2>
              <div class="text-h4 q-mb-lg">
                Grade:
                <span :class="gradeColor" class="text-weight-bold">{{
                  finalGrade
                }}</span>
              </div>
              <p class="text-h6 text-grey-8">
                Accuracy: {{ ((score / totalChars) * 100).toFixed(1) }}%
              </p>
            </q-card-section>
            <q-card-actions align="center">
              <q-btn
                size="lg"
                color="primary"
                label="Practice Again"
                @click="resetSession"
              />
            </q-card-actions>
          </q-card>
        </div>
      </q-page>
    </q-page-container>
  </q-layout>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from "vue";

// --- Types & Dictionary ---
type AppState = "SETUP" | "RUNNING" | "FINISHED";
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
  " ": " ", // Space handled specially
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
  {
    id: "adv-cq",
    title: "Advanced General (Ragchew Intro)",
    lines: [
      { speaker: "TX", text: "CQ CQ DE {CALL} K" },
      { speaker: "RX", text: "{CALL} DE VK3BEE UR 579 579 NAME BOB BK" },
      { speaker: "TX", text: "VK3BEE TU BOB UR 599 599 QTH MELB HW? BK" },
      { speaker: "RX", text: "SOLID CPY TU 73" },
      { speaker: "TX", text: "73 EE" },
    ],
  },
];

// --- State: Configuration ---
const appState = ref<AppState>("SETUP");
const callsign = ref("VK4TEST");
const selectedScript = ref<ScriptDef>(scripts[0]);
const charWpm = ref(20);
const effWpm = ref(10);
const toneFreq = ref(700);
const tolerance = ref(0.3); // 30% timing tolerance threshold

// --- State: Game Loop ---
const compiledScript = ref<ScriptLine[]>([]);
const currentLineIndex = ref(0);
const currentCharIndex = ref(0);
const currentElementIndex = ref(0); // Index within the A-Z morse breakdown
const isUserTurn = ref(false);

// State: Scoring & Visuals
const charStatuses = ref<("pending" | "active" | "passed" | "failed")[]>([]);
const elementFills = ref<number[]>([]); // 0 to 100 percentage
const elementResults = ref<("pending" | "passed" | "failed")[]>([]);
const score = ref(0);
const totalChars = ref(0);

// State: Timing tracking
let audioCtx: AudioContext | null = null;
let oscillator: OscillatorNode | null = null;
let gainNode: GainNode | null = null;
let keydownTime = 0;
let animationFrameId: number | null = null;
let isKeyDown = false;

// --- Computed Properties ---
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

// Strict math: dot duration in ms based on CHARACTER WPM
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

const gradeColor = computed(() => {
  const map: Record<string, string> = {
    S: "text-purple",
    A: "text-green",
    B: "text-blue",
    C: "text-orange",
    D: "text-deep-orange",
    E: "text-red",
  };
  return map[finalGrade.value];
});

// --- Audio Engine ---
const initAudio = () => {
  if (!audioCtx) {
    audioCtx = new (
      window.AudioContext || (window as any).webkitAudioContext
    )();
  }
};

const startTone = () => {
  if (!audioCtx) return;
  if (audioCtx.state === "suspended") audioCtx.resume();

  oscillator = audioCtx.createOscillator();
  gainNode = audioCtx.createGain();

  oscillator.type = "sine";
  oscillator.frequency.value = toneFreq.value;

  // Envelope to prevent clicks (Attack)
  gainNode.gain.setValueAtTime(0, audioCtx.currentTime);
  gainNode.gain.setTargetAtTime(1, audioCtx.currentTime, 0.005);

  oscillator.connect(gainNode);
  gainNode.connect(audioCtx.destination);
  oscillator.start();
};

const stopTone = () => {
  if (!audioCtx || !gainNode || !oscillator) return;

  // Envelope to prevent clicks (Release)
  gainNode.gain.setTargetAtTime(0, audioCtx.currentTime, 0.005);

  const oscToStop = oscillator;
  setTimeout(() => oscToStop.stop(), 50); // wait for release decay

  oscillator = null;
  gainNode = null;
};

// --- Game Logic ---
const startSession = () => {
  initAudio();

  // Compile script (replace {CALL})
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

const resetSession = () => {
  appState.value = "SETUP";
};

const processLine = () => {
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
    simulateRX();
  }
};

const setupCharState = () => {
  const line = compiledScript.value[currentLineIndex.value];

  // Skip spaces automatically
  while (
    currentCharIndex.value < line.text.length &&
    line.text[currentCharIndex.value] === " "
  ) {
    currentCharIndex.value++;
  }

  if (currentCharIndex.value >= line.text.length) {
    // Line finished
    currentLineIndex.value++;
    setTimeout(processLine, 1000);
    return;
  }

  // Init visual state for new character
  if (charStatuses.value.length !== line.text.length) {
    charStatuses.value = Array(line.text.length).fill("pending");
  }
  charStatuses.value[currentCharIndex.value] = "active";

  const expectedCount = currentExpectedElements.value.length;
  currentElementIndex.value = 0;
  elementFills.value = Array(expectedCount).fill(0);
  elementResults.value = Array(expectedCount).fill("pending");
};

// --- Keying Handlers ---
const handleGlobalKeydown = (e: KeyboardEvent) => {
  if (e.code === "Space" && appState.value === "RUNNING" && isUserTurn.value) {
    e.preventDefault();
    handleKeydown();
  }
};

const handleGlobalKeyup = (e: KeyboardEvent) => {
  if (e.code === "Space" && appState.value === "RUNNING" && isUserTurn.value) {
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

  // Calculate percentage filled
  let pct = (elapsed / targetMs) * 100;
  elementFills.value[currentElementIndex.value] = Math.min(pct, 100);

  // Auto-fail / timeout if held too long (Target + Tolerance)
  if (elapsed > targetMs * (1 + tolerance.value)) {
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

  const errorRatio = Math.abs(elapsed - targetMs) / targetMs;
  const passed = errorRatio <= tolerance.value;

  elementResults.value[currentElementIndex.value] = passed
    ? "passed"
    : "failed";

  if (!passed) {
    // Fail the whole character if one element fails
    charStatuses.value[currentCharIndex.value] = "failed";
    totalChars.value++;
    setTimeout(advanceToNextChar, 500);
    return;
  }

  // Advance element or character
  currentElementIndex.value++;
  if (currentElementIndex.value >= currentExpectedElements.value.length) {
    // Character complete and passed
    charStatuses.value[currentCharIndex.value] = "passed";
    score.value++;
    totalChars.value++;
    setTimeout(advanceToNextChar, 300);
  }
};

const advanceToNextChar = () => {
  currentCharIndex.value++;
  setupCharState();
};

// --- RX Simulation ---
const simulateRX = async () => {
  const line = compiledScript.value[currentLineIndex.value];
  const charSpeedMs = dotMs.value;
  const effSpeedRatio = charWpm.value / effWpm.value; // rough spacing multiplier

  // Just a simple visual delay simulation for the prototype RX phase
  const estimatedRxTime = line.text.length * (charSpeedMs * 10 * effSpeedRatio);

  await new Promise((resolve) => setTimeout(resolve, estimatedRxTime));

  currentLineIndex.value++;
  processLine();
};

// --- Helper Functions for UI ---
const getCharStatusClass = (idx: number) => {
  const status = charStatuses.value[idx];
  if (status === "active")
    return "text-primary bg-blue-1 rounded-borders q-px-xs";
  if (status === "passed") return "text-positive";
  if (status === "failed") return "text-negative";
  return "text-grey-5";
};

const getElementFillColor = (idx: number) => {
  const res = elementResults.value[idx];
  if (res === "passed") return "bg-positive";
  if (res === "failed") return "bg-negative";
  return "bg-primary";
};

const getElementFillWidth = (idx: number) => {
  return elementFills.value[idx] || 0;
};

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
.transition-colors {
  transition: color 0.3s ease;
}
.script-container {
  font-family: monospace;
  font-size: 1.5rem;
  line-height: 2;
}
.morse-element-container {
  position: relative;
  box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.1);
}
.morse-element-fill {
  transition: background-color 0.2s;
  border-radius: inherit;
}
</style>
