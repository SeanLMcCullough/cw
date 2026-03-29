<template>
  <q-layout view="hHh lpR fFf" class="bg-dark text-white app-layout">
    <q-page-container>
      <q-page
        class="q-pa-md flex justify-center"
        :class="appState === 'IDLE' ? 'items-center' : 'items-start'"
      >
        <div class="full-width column q-gutter-y-lg" style="max-width: 1200px">
          <q-card
            class="bg-grey-10 text-white flat bordered-custom rounded-custom q-pa-lg shrink-0"
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
                  :disable="!callsign || !selectedScript"
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

          <template v-if="appState === 'RUNNING'">
            <q-card
              class="full-width bg-grey-10 flat bordered-custom rounded-custom keying-window relative-position"
            >
              <div
                class="column items-center justify-center full-width full-height q-pa-lg z-top pointer-events-none"
              >
                <div class="active-message-container q-mb-xl transition-all">
                  <span
                    v-for="(char, charIdx) in compiledScript[currentLineIndex]
                      ?.text"
                    :key="charIdx"
                    class="active-message-char fade-in"
                    :class="[
                      getCharStatusClass(charIdx),
                      char === ' ' ? 'word-space' : '',
                    ]"
                  >
                    {{ char === " " ? "\u00A0" : char }}
                  </span>
                </div>

                <div
                  class="text-h5 q-mb-lg text-grey-4 text-center transition-all"
                >
                  <span v-if="isUserTurn">Your Turn to Key: </span>
                  <span v-else class="text-secondary text-weight-bold"
                    >Receiving:
                  </span>
                  <strong class="text-uppercase text-white text-h3 q-ml-sm">{{
                    currentChar === " " ? "SPACE" : currentChar
                  }}</strong>
                </div>

                <div
                  class="row justify-center items-center q-gutter-x-md transition-all"
                  style="height: 60px"
                >
                  <div
                    v-for="(element, elIdx) in currentExpectedElements"
                    :key="elIdx"
                    class="morse-element-container bg-grey-9 rounded-custom overflow-hidden"
                    :style="{
                      width: element === '.' ? '80px' : '200px',
                      height: '100%',
                    }"
                  >
                    <div
                      class="morse-element-fill full-height"
                      :class="getElementFillColor(elIdx)"
                      :style="{ width: getElementFillWidth(elIdx) + '%' }"
                    ></div>
                  </div>
                </div>

                <div class="q-mt-xl text-h6 text-grey-6 transition-all">
                  <span v-if="isUserTurn"
                    >Tap or hold SPACEBAR, or touch here to key.</span
                  >
                  <span v-else class="text-secondary"
                    >Listen to the timing and cadence...</span
                  >
                </div>
              </div>

              <div
                v-if="isUserTurn"
                class="absolute-top-left full-width full-height cursor-pointer"
                @mousedown="handleKeydown"
                @mouseup="handleKeyup"
                @touchstart.prevent="handleKeydown"
                @touchend.prevent="handleKeyup"
              ></div>
            </q-card>

            <q-card
              class="full-width bg-grey-10 flat bordered-custom rounded-custom history-window q-pa-md"
            >
              <q-card-section class="text-h5 script-history-text">
                <div
                  v-if="compiledScript[currentLineIndex]"
                  class="q-mb-md text-white fade-in text-weight-bolder"
                  style="font-size: 1.2em"
                >
                  <span :class="isUserTurn ? 'text-primary' : 'text-secondary'">
                    [{{ isUserTurn ? "TX" : "RX" }}]
                  </span>
                  {{
                    compiledScript[currentLineIndex].text.substring(
                      0,
                      revealedLength,
                    )
                  }}<span class="cursor-blink">_</span>
                </div>

                <div
                  v-for="(line, index) in reversedHistoryLines"
                  :key="'hist-' + index"
                  class="q-mb-sm text-grey-6 fade-in"
                >
                  <span
                    :class="
                      line.speaker === 'RX' ? 'text-secondary' : 'text-primary'
                    "
                    >[{{ line.speaker }}]</span
                  >
                  {{ line.text }}
                </div>
              </q-card-section>
            </q-card>
          </template>

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
            <q-btn
              class="q-mt-lg rounded-custom"
              size="lg"
              color="primary"
              label="Back to Setup"
              @click="appState = 'IDLE'"
            />
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
const callsign = ref("VK4ABC");
const selectedScript = ref<ScriptDef>(scripts[0]);
const charWpm = ref(20);
const effWpm = ref(10);
const toneFreq = ref(700);
const tolerance = ref(0.3);

const compiledScript = ref<ScriptLine[]>([]);
const historyLines = ref<ScriptLine[]>([]);
const revealedLength = ref(0);

const currentLineIndex = ref(0);
const currentCharIndex = ref(0);
const currentElementIndex = ref(0);
const isUserTurn = ref(false);

const charStatuses = ref<("pending" | "active" | "passed" | "failed" | "rx")[]>(
  [],
);
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

const reversedHistoryLines = computed(() => [...historyLines.value].reverse());

const currentChar = computed(() => {
  if (!compiledScript.value[currentLineIndex.value]) return "";
  return (
    compiledScript.value[currentLineIndex.value].text[currentCharIndex.value] ||
    ""
  );
});

const currentExpectedElements = computed(() => {
  const code = morseDict[currentChar.value.toUpperCase()];
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
  if (!audioCtx) {
    audioCtx = new (
      window.AudioContext || (window as any).webkitAudioContext
    )();
    oscillator = audioCtx.createOscillator();
    gainNode = audioCtx.createGain();
    oscillator.type = "sine";
    oscillator.frequency.value = toneFreq.value;
    gainNode.gain.value = 0;
    oscillator.connect(gainNode);
    gainNode.connect(audioCtx.destination);
    oscillator.start();
  } else if (audioCtx.state === "suspended") {
    audioCtx.resume();
  }
};

const toneOn = () => {
  if (!audioCtx || !gainNode) return;
  const now = audioCtx.currentTime;
  gainNode.gain.cancelScheduledValues(now);
  gainNode.gain.setValueAtTime(gainNode.gain.value, now);
  gainNode.gain.linearRampToValueAtTime(1, now + 0.01);
};

const toneOff = () => {
  if (!audioCtx || !gainNode) return;
  const now = audioCtx.currentTime;
  gainNode.gain.cancelScheduledValues(now);
  gainNode.gain.setValueAtTime(gainNode.gain.value, now);
  gainNode.gain.linearRampToValueAtTime(0, now + 0.01);
};

watch(toneFreq, (newFreq) => {
  if (oscillator && audioCtx)
    oscillator.frequency.setValueAtTime(newFreq, audioCtx.currentTime);
});

// --- Game Logic ---
const startSession = () => {
  initAudio();
  compiledScript.value = selectedScript.value.lines.map((l) => ({
    speaker: l.speaker,
    text: l.text.replace(/\{CALL\}/g, callsign.value.toUpperCase()),
  }));
  historyLines.value = [];
  currentLineIndex.value = 0;
  score.value = 0;
  totalChars.value = 0;
  appState.value = "RUNNING";
  processLine();
};

const stopSession = () => {
  appState.value = "IDLE";
  toneOff();
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
};

const processLine = () => {
  if (appState.value !== "RUNNING") return;
  if (currentLineIndex.value >= compiledScript.value.length) {
    appState.value = "FINISHED";
    return;
  }
  revealedLength.value = 0;
  const line = compiledScript.value[currentLineIndex.value];

  charStatuses.value = Array(line.text.length).fill("pending");

  if (line.speaker === "TX") {
    isUserTurn.value = true;
    currentCharIndex.value = 0;
    setupCharState();
  } else {
    isUserTurn.value = false;
    playRXLine();
  }
};

const finishLine = () => {
  historyLines.value.push(compiledScript.value[currentLineIndex.value]);
  currentLineIndex.value++;
  setTimeout(processLine, 800);
};

const setupCharState = () => {
  const line = compiledScript.value[currentLineIndex.value];

  if (isUserTurn.value) {
    while (
      currentCharIndex.value < line.text.length &&
      line.text[currentCharIndex.value] === " "
    ) {
      currentCharIndex.value++;
      revealedLength.value = currentCharIndex.value;
    }
    if (currentCharIndex.value >= line.text.length) {
      finishLine();
      return;
    }
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
  toneOn();
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  animationFrameId = requestAnimationFrame(updateVisualizer);
};

const handleKeyup = () => {
  if (!isKeyDown || !isUserTurn.value) return;
  isKeyDown = false;
  toneOff();
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  evaluateElement();
};

const updateVisualizer = () => {
  if (!isKeyDown) return;
  const elapsed = performance.now() - keydownTime;
  const expectedElement =
    currentExpectedElements.value[currentElementIndex.value];
  const targetMs = expectedElement === "." ? dotMs.value : dashMs.value;

  elementFills.value[currentElementIndex.value] = Math.min(
    (elapsed / targetMs) * 100,
    100,
  );

  if (elapsed > targetMs * 3) handleKeyup();
  else animationFrameId = requestAnimationFrame(updateVisualizer);
};

const evaluateElement = () => {
  const elapsed = performance.now() - keydownTime;
  const expectedElement =
    currentExpectedElements.value[currentElementIndex.value];
  const targetMs = expectedElement === "." ? dotMs.value : dashMs.value;

  const passed = Math.abs(elapsed - targetMs) / targetMs <= tolerance.value;
  elementResults.value[currentElementIndex.value] = passed
    ? "passed"
    : "failed";
  currentElementIndex.value++;

  if (currentElementIndex.value >= currentExpectedElements.value.length) {
    const allPassed = elementResults.value.every((res) => res === "passed");
    charStatuses.value[currentCharIndex.value] = allPassed
      ? "passed"
      : "failed";
    if (allPassed) score.value++;
    totalChars.value++;

    revealedLength.value = currentCharIndex.value + 1;
    setTimeout(() => {
      currentCharIndex.value++;
      setupCharState();
    }, 300);
  }
};

// --- RX Simulation ---
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

const playRXLine = async () => {
  const line = compiledScript.value[currentLineIndex.value];
  const farnsworthRatio = charWpm.value / effWpm.value;

  for (let i = 0; i < line.text.length; i++) {
    if (appState.value !== "RUNNING") return;

    currentCharIndex.value = i;
    revealedLength.value = i + 1;
    const char = line.text[i];

    if (char === " ") {
      charStatuses.value[i] = "rx";
      await sleep(dashMs.value * farnsworthRatio * 2);
      continue;
    }

    const code = morseDict[char];
    if (!code) continue;

    setupCharState();

    for (let e = 0; e < code.length; e++) {
      if (appState.value !== "RUNNING") return;
      currentElementIndex.value = e;
      const duration = code[e] === "." ? dotMs.value : dashMs.value;

      toneOn();
      elementResults.value[e] = "rx";
      await playRXElementFill(duration, e);
      toneOff();

      await sleep(dotMs.value);
    }
    charStatuses.value[i] = "rx";
    await sleep(dashMs.value * farnsworthRatio);
  }
  finishLine();
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

// --- Helper Functions ---
const getCharStatusClass = (idx: number) => {
  const status = charStatuses.value[idx];
  if (status === "active")
    return "text-dark bg-primary rounded-borders q-px-sm py-xs active-bounce";
  if (status === "passed") return "text-green-4";
  if (status === "failed") return "text-red-5";
  if (status === "rx") return "text-secondary";
  if (isUserTurn.value) return "text-grey-6";
  return "text-transparent";
};

const getElementFillColor = (idx: number) => {
  const res = elementResults.value[idx];
  if (res === "passed") return "bg-green-5";
  if (res === "failed") return "bg-red-5";
  if (res === "rx") return "bg-secondary";
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
.app-layout {
  font-family: "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
}
.rounded-custom {
  border-radius: 24px !important;
}
.bordered-custom {
  border: 1px solid rgba(255, 255, 255, 0.08);
}
.transition-all {
  transition: all 0.3s ease;
}

/* History Window */
.history-window {
  min-height: 150px;
}
.script-history-text {
  font-family: "Courier New", Courier, monospace;
  font-weight: bold;
  line-height: 1.6;
  letter-spacing: 1px;
}
.cursor-blink {
  animation: blink 1s step-end infinite;
}
@keyframes blink {
  50% {
    opacity: 0;
  }
}

/* Keying Window */
.keying-window {
  min-height: 450px;
}

/* Scalable Active Message Area */
.active-message-container {
  width: 100%;
  text-align: center;
  font-size: clamp(1.5rem, 4vw, 3rem);
  font-family: "Courier New", Courier, monospace;
  font-weight: 900;
  line-height: 2;
  word-wrap: break-word;
}

.active-message-char {
  transition:
    color 0.2s,
    background-color 0.2s;
  display: inline-block;
  margin: 0 2px;
}

/* FIX: Guaranteed visible space width for our injected non-breaking space */
.word-space {
  width: 0.6em;
}

.text-transparent {
  color: transparent;
}

/* Morse Elements */
.morse-element-container {
  box-shadow: inset 0 2px 8px rgba(0, 0, 0, 0.3);
}
.morse-element-fill {
  border-radius: inherit;
}

/* Animations */
.fade-in {
  animation: fadeIn 0.3s ease-in-out;
}
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(5px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.active-bounce {
  animation: pop 0.2s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
@keyframes pop {
  0% {
    transform: translateY(5px);
  }
  100% {
    transform: translateY(0);
  }
}
</style>
