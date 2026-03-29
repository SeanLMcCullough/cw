<template>
  <q-layout view="hHh lpR fFf" class="bg-dark text-white app-layout">
    <q-page-container>
      <q-page
        class="q-pa-md flex justify-center"
        :class="appState === 'IDLE' ? 'items-center' : 'items-start'"
      >
        <div class="full-width column q-gutter-y-lg" style="max-width: 1200px">
          <div
            v-if="appState === 'IDLE'"
            class="column items-center justify-center text-center q-pt-xl q-pb-sm fade-in"
          >
            <img
              src="./assets/vk4slm_cw_trainer_logo_600x600.png"
              alt="CW Trainer Logo"
              style="width: 200px; height: 200px; border-radius: 20%"
              class="q-mb-md"
              onerror="this.style.display = 'none'"
            />
            <h1
              class="text-h2 text-weight-bolder text-white q-my-none"
              style="letter-spacing: -1.5px"
            >
              CW Trainer
            </h1>
            <div
              class="text-h5 text-info q-mt-sm text-weight-light"
              style="letter-spacing: 0.5px"
            >
              Learn CW the practical way.
            </div>
          </div>

          <div class="column q-gutter-y-sm shrink-0">
            <q-card
              class="bg-grey-10 text-white flat bordered-custom rounded-custom q-pa-lg"
            >
              <q-card-section class="row q-col-gutter-lg items-center">
                <div class="col-12 col-md-9 column q-gutter-y-md">
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
                      label="Practice Script"
                      dark
                      outlined
                      class="col"
                      :disable="isRunning || scripts.length === 0"
                      :loading="scripts.length === 0"
                    >
                      <template v-slot:option="scope">
                        <q-item v-bind="scope.itemProps">
                          <q-item-section>
                            <q-item-label>{{ scope.opt.title }}</q-item-label>
                            <q-item-label caption class="text-grey-5">{{
                              scope.opt.description
                            }}</q-item-label>
                          </q-item-section>
                        </q-item>
                      </template>
                      <template v-slot:selected-item="scope">
                        {{ scope.opt ? scope.opt.title : "Loading scripts..." }}
                      </template>
                    </q-select>
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

                <div class="col-12 col-md-3 flex flex-center">
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

            <q-expansion-item
              v-model="isAdvancedExpanded"
              icon="tune"
              label="Advanced Tuning & Radio Effects"
              header-class="text-grey-4 text-weight-bold rounded-custom"
              class="bg-grey-10 flat bordered-custom rounded-custom"
            >
              <q-card class="bg-transparent flat">
                <q-card-section class="row q-col-gutter-xl q-pt-none">
                  <div class="col-12 col-md-4 column q-gutter-y-md">
                    <div class="text-subtitle2 text-primary">
                      Transmission (TX)
                    </div>
                    <div>
                      <div class="text-caption text-grey-5">
                        Sidetone Volume
                      </div>
                      <q-slider
                        v-model="txVolume"
                        :min="0.0"
                        :max="1.0"
                        :step="0.05"
                        dark
                        color="primary"
                      />
                    </div>
                    <div>
                      <div class="text-caption text-grey-5">
                        TX Tone: {{ toneFreq }} Hz
                      </div>
                      <q-slider
                        v-model="toneFreq"
                        :min="400"
                        :max="1000"
                        :step="10"
                        dark
                        color="primary"
                      />
                    </div>
                  </div>

                  <div class="col-12 col-md-4 column q-gutter-y-md">
                    <div class="text-subtitle2 text-secondary">
                      Reception (RX)
                    </div>
                    <div>
                      <div class="text-caption text-grey-5">
                        RX Signal Volume
                      </div>
                      <q-slider
                        v-model="rxVolume"
                        :min="0.0"
                        :max="1.0"
                        :step="0.05"
                        dark
                        color="secondary"
                      />
                    </div>
                    <div>
                      <div class="row items-center justify-between q-mb-xs">
                        <div class="text-caption text-grey-4">
                          HF Ionosphere Static
                        </div>
                        <q-toggle
                          v-model="enableNoise"
                          color="secondary"
                          size="sm"
                        />
                      </div>
                      <div :class="enableNoise ? '' : 'disabled-opacity'">
                        <q-slider
                          v-model="noiseVolume"
                          :min="0.01"
                          :max="0.5"
                          :step="0.01"
                          dark
                          color="secondary"
                          :disable="!enableNoise"
                        />
                      </div>
                    </div>
                    <div>
                      <div class="row items-center justify-between q-mb-xs">
                        <div class="text-caption text-grey-4">
                          QSB Signal Fading
                        </div>
                        <q-toggle v-model="enableQsb" color="info" size="sm" />
                      </div>
                      <div :class="enableQsb ? '' : 'disabled-opacity'">
                        <div class="text-caption text-grey-5">Fade Depth</div>
                        <q-slider
                          v-model="qsbDepth"
                          :min="0.1"
                          :max="1.0"
                          :step="0.05"
                          dark
                          color="info"
                          :disable="!enableQsb"
                        />
                      </div>
                    </div>
                  </div>

                  <div class="col-12 col-md-4 column q-gutter-y-md">
                    <div class="text-subtitle2 text-warning">Scoring</div>
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
                        color="warning"
                      />
                    </div>
                  </div>
                </q-card-section>
              </q-card>
            </q-expansion-item>
          </div>

          <template v-if="appState === 'RUNNING'">
            <q-card
              ref="activeSessionRef"
              class="full-width bg-grey-10 flat bordered-custom rounded-custom keying-window relative-position"
            >
              <div
                class="column items-center justify-center full-width full-height q-pa-lg z-top pointer-events-none"
              >
                <div
                  class="preview-box bg-grey-2 rounded-custom q-pa-md q-mb-xl full-width text-center shadow-2 bordered-custom transition-all"
                >
                  <div class="preview-text text-weight-bold">
                    <span
                      v-for="(char, idx) in compiledScript[currentLineIndex]
                        ?.text"
                      :key="'preview-' + idx"
                      :class="
                        idx < currentCharIndex
                          ? 'text-primary'
                          : idx === currentCharIndex
                            ? 'text-dark text-weight-bolder'
                            : 'text-grey-7'
                      "
                      class="transition-all"
                    >
                      {{ char === " " ? "\u00A0" : char }}
                    </span>
                  </div>
                  <div
                    class="text-caption text-grey-7 q-mt-sm text-uppercase"
                    style="letter-spacing: 1px"
                  >
                    <span v-if="isUserTurn">Message to Send</span>
                    <span v-else>Incoming Message</span>
                  </div>
                </div>

                <div
                  class="active-message-viewport q-mb-xl transition-all relative-position"
                  ref="activeMessageRef"
                >
                  <span class="viewport-spacer"></span>
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
                  <span class="viewport-spacer"></span>
                </div>

                <div
                  class="text-h5 q-mb-lg text-grey-4 text-center transition-all"
                >
                  <span v-if="isUserTurn">Your turn to key.</span>
                  <span v-else class="text-secondary text-weight-bold"
                    >Receiving...
                  </span>
                </div>

                <div
                  class="column items-stretch transition-all visualizer-wrapper"
                >
                  <div
                    class="row justify-center items-center transition-all morse-row"
                  >
                    <div
                      v-for="(element, elIdx) in currentExpectedElements"
                      :key="elIdx"
                      class="morse-element-container bg-grey-9 overflow-hidden morse-el"
                      :class="
                        element === '.'
                          ? 'morse-dot rounded-custom'
                          : 'morse-dash'
                      "
                    >
                      <div
                        class="morse-element-fill full-height"
                        :class="getElementFillColor(elIdx)"
                        :style="{ width: getElementFillWidth(elIdx) + '%' }"
                      ></div>
                    </div>
                  </div>

                  <div
                    v-if="isUserTurn"
                    class="bg-grey-9 rounded-custom overflow-hidden q-mt-md transition-all"
                    style="height: 6px; width: 100%"
                  >
                    <div
                      class="bg-info full-height"
                      style="transition: none"
                      :style="{ width: pacerFill + '%' }"
                    ></div>
                  </div>
                </div>

                <div
                  class="q-mt-xl text-h6 transition-all"
                  :class="
                    isWaitingForGap
                      ? 'text-warning text-weight-bold'
                      : 'text-grey-6'
                  "
                >
                  <span v-if="isUserTurn && isWaitingForGap"></span>
                  <span v-else-if="isUserTurn"
                    >Tap or hold SPACEBAR, or touch here to key.</span
                  >
                  <span v-else class="text-secondary"></span>
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
              <q-card-section
                class="script-history-text history-scroll-area"
                ref="historyScrollRef"
              >
                <div
                  v-for="(line, index) in historyLines"
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

                <div
                  v-if="compiledScript[currentLineIndex]"
                  class="q-mb-md text-white fade-in active-history-line"
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

    <q-footer
      class="bg-dark text-grey-7 text-center q-pa-md flat"
      style="border-top: 1px solid rgba(255, 255, 255, 0.05)"
    >
      <div class="text-caption" style="letter-spacing: 0.5px">
        Copyright {{ new Date().getFullYear() }} Sean McCullough - Made in
        Australia by VK4SLM -
        <a href="https://mccullough.xyz" target="_blank" class="footer-link"
          >mccullough.xyz</a
        >
        -
        <a
          href="https://github.com/SeanLMcCullough/cw"
          target="_blank"
          class="footer-link"
          >GitHub Repository</a
        >
      </div>
    </q-footer>
  </q-layout>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch, nextTick } from "vue";

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
  description: string;
  lines: ScriptLine[];
}

// Fixed missing hyphen, period, and comma
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
  "-": "-....-",
  ".": ".-.-.-",
  ",": "--..--",
};

// --- State ---
const appState = ref<AppState>("IDLE");
const callsign = ref("VK4SLM");
const selectedScript = ref<ScriptDef | null>(null);
const scripts = ref<ScriptDef[]>([]);

const charWpm = ref(15);
const effWpm = ref(15);

// UI State
const isAdvancedExpanded = ref(false);
const isAcceptingInput = ref(false);
const isWaitingForGap = ref(false);
let isFirstCharOfTx = true;

// Audio State
const toneFreq = ref(700);
const rxToneFreq = ref(700);
const tolerance = ref(0.3);
const txVolume = ref(1.0);
const rxVolume = ref(1.0);
const enableNoise = ref(true);
const noiseVolume = ref(0.05);
const enableQsb = ref(true);
const qsbDepth = ref(0.9);

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

// DOM Refs
const activeMessageRef = ref<HTMLElement | null>(null);
const activeSessionRef = ref<any>(null);
const historyScrollRef = ref<any>(null);

// Audio Engine Refs
let audioCtx: AudioContext | null = null;
let oscillator: OscillatorNode | null = null;
let gainNode: GainNode | null = null;
let noiseSource: AudioBufferSourceNode | null = null;
let noiseFilter: BiquadFilterNode | null = null;
let noiseGain: GainNode | null = null;
let noiseLfo: OscillatorNode | null = null;
let noiseLfoGain: GainNode | null = null;

let keydownTime = 0;
let animationFrameId: number | null = null;
let isKeyDown = false;
let keyLockoutTime = 0;
const DEBOUNCE_DELAY = 12;

const pacerFill = ref(0);
let pacerHasStarted = false;
let txTimeout: ReturnType<typeof setTimeout> | null = null;
let pacerFrameId: number | null = null;
let pacerStartTime = 0;
let pacerDuration = 0;

// --- Computed ---
const isRunning = computed(() => appState.value === "RUNNING");

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
function initAudio() {
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

    const bufferSize = audioCtx.sampleRate * 2;
    const buffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
    const output = buffer.getChannelData(0);
    for (let i = 0; i < bufferSize; i++) output[i] = Math.random() * 2 - 1;

    noiseSource = audioCtx.createBufferSource();
    noiseSource.buffer = buffer;
    noiseSource.loop = true;
    noiseFilter = audioCtx.createBiquadFilter();
    noiseFilter.type = "bandpass";
    noiseFilter.frequency.value = 1000;
    noiseFilter.Q.value = 1.0;
    noiseLfo = audioCtx.createOscillator();
    noiseLfo.type = "sine";
    noiseLfo.frequency.value = 0.15;
    noiseLfoGain = audioCtx.createGain();
    noiseLfoGain.gain.value = 300;
    noiseGain = audioCtx.createGain();
    noiseGain.gain.value = 0;

    noiseLfo.connect(noiseLfoGain);
    noiseLfoGain.connect(noiseFilter.frequency);
    noiseSource.connect(noiseFilter);
    noiseFilter.connect(noiseGain);
    noiseGain.connect(audioCtx.destination);

    noiseSource.start();
    noiseLfo.start();
  } else if (audioCtx.state === "suspended") {
    audioCtx.resume();
  }
}

function syncNoiseEngine() {
  if (!audioCtx || !noiseGain) return;
  const isTransmitting = isUserTurn.value && isKeyDown;
  const targetVol =
    isRunning.value && enableNoise.value && !isTransmitting
      ? noiseVolume.value
      : 0;
  noiseGain.gain.setTargetAtTime(targetVol, audioCtx.currentTime, 0.01);
}

watch([enableNoise, noiseVolume, isUserTurn, appState], syncNoiseEngine);

function toneOn() {
  if (!audioCtx || !gainNode || !oscillator) return;
  const now = audioCtx.currentTime;
  const activeFreq = isUserTurn.value ? toneFreq.value : rxToneFreq.value;
  oscillator.frequency.setTargetAtTime(activeFreq, now, 0.005);

  let targetVol = isUserTurn.value ? txVolume.value : rxVolume.value;
  if (!isUserTurn.value && enableQsb.value) {
    const qsbWave = (Math.sin(performance.now() / 2500) + 1) / 2;
    targetVol = rxVolume.value - qsbDepth.value * qsbWave * rxVolume.value;
  }

  gainNode.gain.cancelScheduledValues(now);
  gainNode.gain.setValueAtTime(gainNode.gain.value, now);
  gainNode.gain.setTargetAtTime(targetVol, now, 0.005);
}

function toneOff() {
  if (!audioCtx || !gainNode) return;
  const now = audioCtx.currentTime;
  gainNode.gain.cancelScheduledValues(now);
  gainNode.gain.setValueAtTime(gainNode.gain.value, now);
  gainNode.gain.setTargetAtTime(0, now, 0.005);
}

// --- Game Logic ---
async function startSession() {
  if (!selectedScript.value) return;
  initAudio();
  isAdvancedExpanded.value = false;
  isAcceptingInput.value = false;
  isWaitingForGap.value = false;
  isFirstCharOfTx = true;

  const offset = Math.floor(Math.random() * 151) + 50;
  const dir = Math.random() > 0.5 ? 1 : -1;
  let newRxFreq = toneFreq.value + offset * dir;
  rxToneFreq.value = Math.max(400, Math.min(1000, newRxFreq));

  compiledScript.value = selectedScript.value.lines.map((l) => ({
    speaker: l.speaker,
    text: l.text.replace(/\{CALL\}/g, callsign.value.toUpperCase()),
  }));

  historyLines.value = [];
  currentLineIndex.value = 0;
  score.value = 0;
  totalChars.value = 0;
  appState.value = "RUNNING";

  syncNoiseEngine();
  processLine();

  await nextTick();
  if (activeSessionRef.value) {
    const el = activeSessionRef.value.$el || activeSessionRef.value;
    el.scrollIntoView({ behavior: "smooth", block: "start" });
  }
}

function stopSession() {
  appState.value = "IDLE";
  isAcceptingInput.value = false;
  isWaitingForGap.value = false;
  toneOff();
  syncNoiseEngine();
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  if (pacerFrameId) cancelAnimationFrame(pacerFrameId);
  if (txTimeout) clearTimeout(txTimeout);
}

function processLine() {
  if (appState.value !== "RUNNING") return;
  if (currentLineIndex.value >= compiledScript.value.length) {
    appState.value = "FINISHED";
    isAcceptingInput.value = false;
    isWaitingForGap.value = false;
    syncNoiseEngine();
    return;
  }
  revealedLength.value = 0;
  const line = compiledScript.value[currentLineIndex.value];
  charStatuses.value = Array(line.text.length).fill("pending");

  if (line.speaker === "TX") {
    isUserTurn.value = true;
    currentCharIndex.value = 0;
    revealedLength.value = 1;
    isFirstCharOfTx = true;
    setupPlayableChar(0);
  } else {
    isUserTurn.value = false;
    playRXLine();
  }
}

// BUGFIX: Allow 800ms delay for animations to finish before swapping to the next line
function finishLine() {
  setTimeout(() => {
    if (appState.value !== "RUNNING") return;
    historyLines.value.push(compiledScript.value[currentLineIndex.value]);
    currentLineIndex.value++;
    processLine();
  }, 800);
}

async function scrollToCurrentChar() {
  await nextTick();
  if (!activeMessageRef.value) return;
  const container = activeMessageRef.value;
  const chars = container.querySelectorAll(".active-message-char");
  const activeEl = chars[currentCharIndex.value] as HTMLElement;
  if (activeEl) {
    const targetScrollLeft =
      activeEl.offsetLeft -
      container.offsetWidth / 2 +
      activeEl.offsetWidth / 2;
    container.scrollTo({ left: targetScrollLeft, behavior: "smooth" });
  }
}

function setupPlayableChar(accumulatedDelay: number) {
  if (appState.value !== "RUNNING") return;
  if (txTimeout) clearTimeout(txTimeout);
  const line = compiledScript.value[currentLineIndex.value];
  if (currentCharIndex.value >= line.text.length) {
    txTimeout = setTimeout(finishLine, accumulatedDelay);
    return;
  }
  const char = line.text[currentCharIndex.value];
  if (char === " ") {
    charStatuses.value[currentCharIndex.value] = "passed";
    const farnsworthRatio = charWpm.value / effWpm.value;
    const wordSpaceMs = dotMs.value * 4 * farnsworthRatio;
    currentCharIndex.value++;
    revealedLength.value = currentCharIndex.value + 1;
    setupPlayableChar(accumulatedDelay + wordSpaceMs);
    return;
  }

  charStatuses.value[currentCharIndex.value] = "active";
  const expectedCount = currentExpectedElements.value.length;
  currentElementIndex.value = 0;
  elementFills.value = Array(expectedCount).fill(0);
  elementResults.value = Array(expectedCount).fill("pending");
  scrollToCurrentChar();

  if (isUserTurn.value) {
    isAcceptingInput.value = false;
    if (accumulatedDelay > 50) isWaitingForGap.value = true;
    pacerHasStarted = false;
    if (pacerFrameId) cancelAnimationFrame(pacerFrameId);
    pacerFill.value = 0;
    if (currentExpectedElements.value.length > 0) {
      pacerDuration = currentExpectedElements.value.reduce((total, el, idx) => {
        const elMs = el === "." ? dotMs.value : dashMs.value;
        const gapMs =
          idx < currentExpectedElements.value.length - 1 ? dotMs.value : 0;
        return total + elMs + gapMs;
      }, 0);
    }

    setTimeout(() => {
      if (appState.value === "RUNNING" && isUserTurn.value)
        isAcceptingInput.value = true;
    }, 20);

    txTimeout = setTimeout(
      () => {
        if (appState.value !== "RUNNING" || !isUserTurn.value) return;
        isWaitingForGap.value = false;
        isAcceptingInput.value = true;
        if (isFirstCharOfTx) {
          isFirstCharOfTx = false;
        } else if (!pacerHasStarted) {
          pacerHasStarted = true;
          pacerStartTime = performance.now();
          const updatePacer = () => {
            const elapsed = performance.now() - pacerStartTime;
            pacerFill.value = Math.min((elapsed / pacerDuration) * 100, 100);
            if (pacerFill.value < 100 && appState.value === "RUNNING")
              pacerFrameId = requestAnimationFrame(updatePacer);
          };
          pacerFrameId = requestAnimationFrame(updatePacer);
        }
      },
      Math.max(20, accumulatedDelay),
    );
  }
}

// --- Keying Handlers ---
function handleKeydown() {
  const now = performance.now();
  if (
    now < keyLockoutTime ||
    !isAcceptingInput.value ||
    isKeyDown ||
    !isUserTurn.value
  )
    return;
  isKeyDown = true;
  keydownTime = now;
  keyLockoutTime = now + DEBOUNCE_DELAY;
  toneOn();
  syncNoiseEngine();
  if (!pacerHasStarted) {
    pacerHasStarted = true;
    pacerStartTime = now;
    const updatePacer = () => {
      const elapsed = performance.now() - pacerStartTime;
      pacerFill.value = Math.min((elapsed / pacerDuration) * 100, 100);
      if (pacerFill.value < 100 && appState.value === "RUNNING")
        pacerFrameId = requestAnimationFrame(updatePacer);
    };
    pacerFrameId = requestAnimationFrame(updatePacer);
  }
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  animationFrameId = requestAnimationFrame(updateVisualizer);
}

function handleKeyup() {
  if (!isKeyDown || !isUserTurn.value) return;
  const now = performance.now();
  if (now < keyLockoutTime) {
    setTimeout(handleKeyup, keyLockoutTime - now);
    return;
  }
  isKeyDown = false;
  keyLockoutTime = now + DEBOUNCE_DELAY;
  toneOff();
  syncNoiseEngine();
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  evaluateElement();
}

function updateVisualizer() {
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
}

function evaluateElement() {
  if (currentElementIndex.value >= currentExpectedElements.value.length) return;
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
    isAcceptingInput.value = false;
    const allPassed = elementResults.value.every((res) => res === "passed");
    charStatuses.value[currentCharIndex.value] = allPassed
      ? "passed"
      : "failed";
    if (allPassed) score.value++;
    totalChars.value++;
    const farnsworthRatio = charWpm.value / effWpm.value;
    const letterSpaceMs = dotMs.value * 3 * farnsworthRatio;
    currentCharIndex.value++;
    revealedLength.value = currentCharIndex.value + 1;
    setupPlayableChar(letterSpaceMs);
  }
}

// --- RX Simulation ---
function sleep(ms: number) {
  return new Promise((r) => setTimeout(r, ms));
}

async function playRXLine() {
  const line = compiledScript.value[currentLineIndex.value];
  const farnsworthRatio = charWpm.value / effWpm.value;
  for (let i = 0; i < line.text.length; i++) {
    if (appState.value !== "RUNNING") return;
    currentCharIndex.value = i;
    revealedLength.value = i + 1;
    if (line.text[i] === " ") {
      charStatuses.value[i] = "rx";
      await new Promise((r) =>
        setTimeout(r, dotMs.value * 4 * farnsworthRatio),
      );
      continue;
    }
    const code = morseDict[line.text[i]];
    if (!code) continue;
    charStatuses.value[currentCharIndex.value] = "active";
    currentElementIndex.value = 0;
    elementFills.value = Array(code.length).fill(0);
    elementResults.value = Array(code.length).fill("pending");
    scrollToCurrentChar();
    for (let e = 0; e < code.length; e++) {
      if (appState.value !== "RUNNING") return;
      currentElementIndex.value = e;
      const duration = code[e] === "." ? dotMs.value : dashMs.value;
      toneOn();
      elementResults.value[e] = "rx";
      await playRXElementFill(duration, e);
      toneOff();
      if (e < code.length - 1)
        await new Promise((r) => setTimeout(r, dotMs.value));
    }
    charStatuses.value[i] = "rx";
    await new Promise((r) => setTimeout(r, dashMs.value * farnsworthRatio));
  }
  finishLine();
}

function playRXElementFill(durationMs: number, index: number) {
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
}

function handleGlobalKeydown(e: KeyboardEvent) {
  if (e.repeat) return;
  if (e.code === "Space" && isRunning.value && isUserTurn.value) {
    e.preventDefault();
    handleKeydown();
  }
}
function handleGlobalKeyup(e: KeyboardEvent) {
  if (e.code === "Space" && isRunning.value && isUserTurn.value) {
    e.preventDefault();
    handleKeyup();
  }
}

function getCharStatusClass(idx: number) {
  const status = charStatuses.value[idx];
  if (status === "active")
    return isWaitingForGap.value &&
      isUserTurn.value &&
      idx === currentCharIndex.value
      ? "text-dark bg-grey-6"
      : "text-dark bg-primary active-bounce";
  if (status === "passed") return "text-green-4";
  if (status === "failed") return "text-red-5";
  if (status === "rx") return "text-secondary";
  return isUserTurn.value ? "text-grey-6" : "text-transparent";
}
function getElementFillColor(idx: number) {
  const res = elementResults.value[idx];
  if (res === "passed") return "bg-green-5";
  if (res === "failed") return "bg-red-5";
  if (res === "rx") return "bg-secondary";
  return "bg-primary";
}
function getElementFillWidth(idx: number) {
  return elementFills.value[idx] || 0;
}

// --- Lifecycle, UI Tracking, & Persistence ---
const STORAGE_KEY = "cw-trainer-settings-v3";

watch(
  [historyLines, revealedLength],
  async () => {
    await nextTick();
    if (historyScrollRef.value) {
      const el = historyScrollRef.value.$el || historyScrollRef.value;
      el.scrollTop = el.scrollHeight;
    }
  },
  { deep: true },
);

onMounted(async () => {
  try {
    const response = await fetch("/scripts.json");
    if (response.ok) {
      scripts.value = await response.json();
      if (!selectedScript.value && scripts.value.length > 0) {
        selectedScript.value = scripts.value[0];
      }
    }
  } catch (e) {
    console.error("Failed to load scripts.json", e);
  }

  const saved = localStorage.getItem(STORAGE_KEY);
  if (saved) {
    try {
      const parsed = JSON.parse(saved);
      if (parsed.callsign) callsign.value = parsed.callsign;
      if (parsed.scriptId) {
        const found = scripts.value.find((s) => s.id === parsed.scriptId);
        if (found) selectedScript.value = found;
      }
      if (parsed.charWpm) charWpm.value = parsed.charWpm;
      if (parsed.effWpm) effWpm.value = parsed.effWpm;
      if (parsed.toneFreq) toneFreq.value = parsed.toneFreq;
      if (parsed.tolerance) tolerance.value = parsed.tolerance;
      if (parsed.txVolume !== undefined) txVolume.value = parsed.txVolume;
      if (parsed.rxVolume !== undefined) rxVolume.value = parsed.rxVolume;
      if (parsed.enableNoise !== undefined)
        enableNoise.value = parsed.enableNoise;
      if (parsed.noiseVolume) noiseVolume.value = parsed.noiseVolume;
      if (parsed.enableQsb !== undefined) enableQsb.value = parsed.enableQsb;
      if (parsed.qsbDepth !== undefined) qsbDepth.value = parsed.qsbDepth;
      if (parsed.isAdvancedExpanded !== undefined)
        isAdvancedExpanded.value = parsed.isAdvancedExpanded;
    } catch (e) {
      console.warn("Failed to parse settings", e);
    }
  }
  window.addEventListener("keydown", handleGlobalKeydown);
  window.addEventListener("keyup", handleGlobalKeyup);
});

watch(
  [
    callsign,
    selectedScript,
    charWpm,
    effWpm,
    toneFreq,
    tolerance,
    txVolume,
    rxVolume,
    enableNoise,
    noiseVolume,
    enableQsb,
    qsbDepth,
    isAdvancedExpanded,
  ],
  () => {
    const settings = {
      callsign: callsign.value,
      scriptId: selectedScript.value?.id,
      charWpm: charWpm.value,
      effWpm: effWpm.value,
      toneFreq: toneFreq.value,
      tolerance: tolerance.value,
      txVolume: txVolume.value,
      rxVolume: rxVolume.value,
      enableNoise: enableNoise.value,
      noiseVolume: noiseVolume.value,
      enableQsb: enableQsb.value,
      qsbDepth: qsbDepth.value,
      isAdvancedExpanded: isAdvancedExpanded.value,
    };
    localStorage.setItem(STORAGE_KEY, JSON.stringify(settings));
  },
  { deep: true },
);

onUnmounted(() => {
  window.removeEventListener("keydown", handleGlobalKeydown);
  window.removeEventListener("keyup", handleGlobalKeyup);
  if (animationFrameId) cancelAnimationFrame(animationFrameId);
  if (pacerFrameId) cancelAnimationFrame(pacerFrameId);
  if (txTimeout) clearTimeout(txTimeout);
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
.disabled-opacity {
  opacity: 0.4;
  pointer-events: none;
  transition: opacity 0.3s;
}

/* History Window Auto-Scroll */
.history-window {
  height: 350px;
  display: flex;
  flex-direction: column;
}
.history-scroll-area {
  overflow-y: auto;
  flex-grow: 1;
  scroll-behavior: smooth;
  scrollbar-width: thin;
  scrollbar-color: rgba(255, 255, 255, 0.1) transparent;
}

.script-history-text {
  font-family: "Courier New", Courier, monospace;
  font-weight: bold;
  line-height: 1.6;
  letter-spacing: 1px;
  word-wrap: break-word;
  font-size: clamp(1rem, 3vw, 1.5rem);
}
.active-history-line {
  font-size: clamp(1.2rem, 4vw, 1.8rem);
}
.cursor-blink {
  animation: blink 1s step-end infinite;
}
@keyframes blink {
  50% {
    opacity: 0;
  }
}
.keying-window {
  min-height: 450px;
}

/* Heads-Up Preview Box */
.preview-box {
  min-height: 80px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}
.preview-text {
  font-family: "Courier New", Courier, monospace;
  font-size: clamp(1.2rem, 4vw, 2rem);
  letter-spacing: 2px;
  white-space: pre-wrap; /* Allows the browser to wrap naturally on spaces */
  word-break: break-word;
}

/* Determinisitc Single-Line Viewport */
.active-message-viewport {
  width: 100%;
  text-align: left;
  white-space: nowrap;
  overflow-x: hidden;
  overflow-y: hidden;
  padding-top: 10px;
  padding-bottom: 15px;
  font-size: clamp(2rem, 5vw, 4rem);
  font-family: "Courier New", Courier, monospace;
  font-weight: 900;
  mask-image: linear-gradient(
    to right,
    transparent 0%,
    black 15%,
    black 85%,
    transparent 100%
  );
  -webkit-mask-image: linear-gradient(
    to right,
    transparent 0%,
    black 15%,
    black 85%,
    transparent 100%
  );
}
.viewport-spacer {
  display: inline-block;
  width: 50%;
}
.active-message-char {
  transition:
    color 0.2s,
    background-color 0.2s;
  display: inline-block;
  margin: 0 4px;
  padding: 4px 8px;
  border-radius: 8px;
}
.word-space {
  width: 0.6em;
}
.text-transparent {
  color: transparent;
}
.morse-element-container {
  box-shadow: inset 0 2px 8px rgba(0, 0, 0, 0.3);
}
.morse-element-fill {
  border-radius: inherit;
}
.footer-link {
  color: inherit;
  text-decoration: none;
  transition: color 0.2s ease;
}
.footer-link:hover {
  color: white;
}

/* Visualizer Responsiveness */
.visualizer-wrapper {
  width: max-content;
  max-width: 100%;
  margin: 0 auto;
}
.morse-row {
  height: 60px;
  gap: 16px;
  width: 100%;
  flex-wrap: nowrap !important;
}
.morse-el {
  height: 100%;
  min-width: 0;
}
.morse-dot {
  width: 80px;
  flex-shrink: 1;
}
.morse-dash {
  width: 200px;
  flex-shrink: 1;
  border-radius: 0 !important; /* Force strict 90-degree corners */
}
@media (max-width: 600px) {
  .morse-row {
    height: 40px;
    gap: 8px;
  }
  .morse-dot {
    width: 40px;
  }
  .morse-dash {
    width: 100px;
  }
}
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
