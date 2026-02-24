<script setup>
import { ref, computed, watch, onMounted } from "vue";

const LANGUAGES = [
  {
    code: "it",
    label: "Italiano",
    csvPath: "data/italian_core_8000_wordforms.csv",
    wordformsV1Path: "data/italian_lexicon_wordforms_v1.csv",
    lemmasV1Path: "data/italian_lexicon_lemmas_v1.csv",
    domainsPath: "data/domains_v1.csv",
  },
];

// Lexicon (v1) for tokenizer + mastery
const wordformLookup = ref(new Map()); // wordform -> { lemma_id, lemma, pos, cefr, english_gloss, ... }
const lemmasById = ref(new Map());     // lemma_id -> { lemma, domain, ... }
const lexiconLoaded = ref(false);
const lexiconLoadError = ref("");

function parseWordformsV1(text) {
  const lines = text.split(/\r?\n/).map((l) => l.trim()).filter(Boolean);
  if (!lines.length) return [];
  const h = lines[0].split(",").map((c) => c.trim());
  const get = (row, name) => {
    const i = h.indexOf(name);
    return i >= 0 ? row[i]?.trim() ?? "" : "";
  };
  const out = [];
  for (let i = 1; i < lines.length; i++) {
    const cols = lines[i].split(",").map((c) => c.trim());
    const wordform = get(cols, "wordform");
    if (!wordform) continue;
    out.push({
      wordform,
      lemma_id: get(cols, "lemma_id"),
      pos: get(cols, "pos"),
      cefr: get(cols, "cefr"),
      english_gloss: get(cols, "english_gloss"),
      tense: get(cols, "tense"),
      person: get(cols, "person"),
    });
  }
  return out;
}

function parseLemmasV1(text) {
  const lines = text.split(/\r?\n/).map((l) => l.trim()).filter(Boolean);
  if (!lines.length) return [];
  const h = lines[0].split(",").map((c) => c.trim());
  const get = (row, name) => {
    const i = h.indexOf(name);
    return i >= 0 ? row[i]?.trim() ?? "" : "";
  };
  const out = [];
  for (let i = 1; i < lines.length; i++) {
    const cols = lines[i].split(",").map((c) => c.trim());
    const lemma_id = get(cols, "lemma_id");
    if (!lemma_id) continue;
    out.push({
      lemma_id,
      lemma: get(cols, "lemma"),
      pos: get(cols, "pos"),
      cefr: get(cols, "cefr"),
      english_gloss: get(cols, "english_gloss"),
      frequency_rank: get(cols, "frequency_rank"),
      domain: get(cols, "domain"),
      difficulty: get(cols, "difficulty"),
    });
  }
  return out;
}

function parseDomainsV1(text) {
  const lines = text.split(/\r?\n/).map((l) => l.trim()).filter(Boolean);
  if (!lines.length) return [];
  const h = lines[0].split(",").map((c) => c.trim());
  const domainIdx = h.indexOf("domain");
  const descIdx = h.indexOf("description");
  const out = [];
  for (let i = 1; i < lines.length; i++) {
    const cols = lines[i].split(",").map((c) => c.trim());
    const domain = domainIdx >= 0 ? cols[domainIdx] ?? "" : "";
    if (!domain) continue;
    out.push({ domain, description: descIdx >= 0 ? cols[descIdx] ?? "" : "" });
  }
  return out;
}

async function loadLexicon() {
  const lang = LANGUAGES.find((l) => l.code === selectedLanguageCode.value);
  if (!lang?.wordformsV1Path) return;
  lexiconLoadError.value = "";
  try {
    const [wfRes, lemRes] = await Promise.all([
      fetch(lang.wordformsV1Path),
      fetch(lang.lemmasV1Path),
    ]);
    if (!wfRes.ok || !lemRes.ok) throw new Error("Failed to load lexicon");
    const wfRows = parseWordformsV1(await wfRes.text());
    const lemRows = parseLemmasV1(await lemRes.text());
    const lemmaMap = new Map();
    lemRows.forEach((r) => lemmaMap.set(r.lemma_id, r));
    const wfMap = new Map();
    wfRows.forEach((r) => {
      const lemma = lemmaMap.get(r.lemma_id);
      wfMap.set(r.wordform.toLowerCase(), {
        ...r,
        lemma: lemma?.lemma ?? r.wordform,
      });
    });
    wordformLookup.value = wfMap;
    lemmasById.value = lemmaMap;
    lexiconLoaded.value = true;
  } catch (e) {
    console.error(e);
    lexiconLoadError.value = "Lexicon not loaded (v1 CSVs missing?). Reading/Generate need it.";
    lexiconLoaded.value = false;
  }
}

const UNSPLASH_BASE = "https://source.unsplash.com/featured/1600x900";

function buildImageUrl(word, languageLabel) {
  const query = encodeURIComponent(`${word},${languageLabel},minimal,photography`);
  return `${UNSPLASH_BASE}/?${query}`;
}

function parseCsv(text) {
  const lines = text
    .split(/\r?\n/)
    .map((l) => l.trim())
    .filter((l) => l.length > 0);
  if (!lines.length) return [];

  const header = lines[0].split(",").map((h) => h.trim());
  const wordIdx = header.indexOf("word");
  const lemmaIdx = header.indexOf("lemma");
  const posIdx = header.indexOf("part_of_speech");
  const cefrIdx = header.indexOf("cefr");
  const englishIdx = header.indexOf("english");
  const tenseIdx = header.indexOf("tense");
  const personIdx = header.indexOf("person");

  const res = [];
  for (let i = 1; i < lines.length; i++) {
    const cols = lines[i].split(",").map((c) => c.trim());
    const word = cols[wordIdx] || "";
    if (!word) continue;
    res.push({
      word,
      lemma: cols[lemmaIdx] ?? "",
      partOfSpeech: cols[posIdx] ?? "",
      cefr: cols[cefrIdx] ?? "",
      english: cols[englishIdx] ?? "",
      tense: cols[tenseIdx] ?? "",
      person: cols[personIdx] ?? "",
    });
  }
  return res;
}

function loadKnownSetKey(langCode) {
  return `language-index-known:${langCode}`;
}

function loadKnownSet(langCode) {
  try {
    const raw = localStorage.getItem(loadKnownSetKey(langCode));
    if (!raw) return new Set();
    const arr = JSON.parse(raw);
    if (!Array.isArray(arr)) return new Set();
    return new Set(arr);
  } catch {
    return new Set();
  }
}

function saveKnownSet(langCode, set) {
  try {
    const arr = Array.from(set);
    localStorage.setItem(loadKnownSetKey(langCode), JSON.stringify(arr));
  } catch {
    // ignore
  }
}

// Mastery: 0=Unknown, 1=Recognized, 2=Passive, 3=Active, 4=Mastered
const MASTERY_LEVELS = ["Unknown", "Recognized", "Passive", "Active", "Mastered"];
function masteryKey(langCode) {
  return `language-index-mastery:${langCode}`;
}
function loadMastery(langCode) {
  try {
    const raw = localStorage.getItem(masteryKey(langCode));
    if (!raw) return {};
    const o = JSON.parse(raw);
    return typeof o === "object" && o !== null ? o : {};
  } catch {
    return {};
  }
}
function saveMastery(langCode, obj) {
  try {
    localStorage.setItem(masteryKey(langCode), JSON.stringify(obj));
  } catch {
    // ignore
  }
}

const masteryByLemma = ref({});

const languages = ref(LANGUAGES);
const selectedLanguageCode = ref(LANGUAGES[0]?.code ?? "");
const status = ref("idle"); // idle | loading | ready | error
const statusMessage = ref("");

const words = ref([]);
const index = ref(0);
const knownSet = ref(new Set());

const filteredWords = computed(() =>
  words.value.filter((w) => !knownSet.value.has(w.word))
);

const currentWord = computed(() => {
  if (!filteredWords.value.length) return null;
  return filteredWords.value[index.value] ?? null;
});

const total = computed(() => filteredWords.value.length);
const positionLabel = computed(() => {
  if (!total.value) return "";
  return `${index.value + 1} / ${total.value}`;
});
const remainingCount = computed(() => Math.max(0, total.value - (index.value + 1)));

const deckSize = computed(() => words.value.length);
const knownCount = computed(() => knownSet.value.size);
const unknownLeftCount = computed(() => Math.max(0, deckSize.value - knownCount.value));

const confirmedWordsList = computed(() =>
  Array.from(knownSet.value).sort((a, b) => a.localeCompare(b))
);

const sidebarOpen = ref(false);

function toggleSidebar() {
  sidebarOpen.value = !sidebarOpen.value;
}

function closeSidebar() {
  sidebarOpen.value = false;
}

function clearProgress() {
  const langCode = selectedLanguageCode.value;
  knownSet.value = new Set();
  masteryByLemma.value = {};
  saveKnownSet(langCode, knownSet.value);
  saveMastery(langCode, masteryByLemma.value);
  if (filteredWords.value.length) {
    index.value = 0;
    setBackgroundForIndex(0);
  }
}

const languageLabel = computed(() => {
  const lang = languages.value.find((l) => l.code === selectedLanguageCode.value);
  return lang?.label ?? "";
});

const isKnown = computed(() => {
  const cw = currentWord.value;
  if (!cw) return false;
  return knownSet.value.has(cw.word);
});

// Background image handling
const currentImageUrl = ref("");
const nextImageUrl = ref("");
const showNextAsCurrent = ref(false);

function preloadImage(url) {
  if (!url) return;
  const img = new Image();
  img.src = url;
}

function setBackgroundForIndex(idx) {
  const list = filteredWords.value;
  const w = list[idx];
  if (!w) return;
  const url = buildImageUrl(w.word, languageLabel.value || "");
  nextImageUrl.value = url;
  preloadImage(url);
  showNextAsCurrent.value = true;
  setTimeout(() => {
    currentImageUrl.value = url;
    showNextAsCurrent.value = false;
  }, 350);
}

async function loadLanguageWords(langCode) {
  const lang = languages.value.find((l) => l.code === langCode);
  if (!lang) return;
  status.value = "loading";
  statusMessage.value = "Loading deck…";

  try {
    const res = await fetch(lang.csvPath);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const text = await res.text();
    const parsed = parseCsv(text);
    if (!parsed.length) {
      throw new Error("CSV did not contain any words.");
    }
    // Randomize deck order once per load
    for (let i = parsed.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [parsed[i], parsed[j]] = [parsed[j], parsed[i]];
    }
    words.value = parsed;
    knownSet.value = loadKnownSet(lang.code);
    masteryByLemma.value = loadMastery(lang.code);
    index.value = 0;
    status.value = "ready";
    statusMessage.value = "";
    // setBackgroundForIndex(0);
    loadLexicon();
  } catch (err) {
    console.error(err);
    status.value = "error";
    statusMessage.value = "Could not load this language deck.";
    words.value = [];
  }
}

function go(delta = 1) {
  const list = filteredWords.value;
  if (!list.length) return;
  index.value = (index.value + delta + list.length) % list.length;
  // setBackgroundForIndex(index.value);
}

function markKnown() {
  const cw = currentWord.value;
  if (!cw) return;
  if (!knownSet.value.has(cw.word)) {
    const nextSet = new Set(knownSet.value);
    nextSet.add(cw.word);
    knownSet.value = nextSet;
    saveKnownSet(selectedLanguageCode.value, nextSet);
  }
  const info = wordformLookup.value.get(cw.word?.toLowerCase?.());
  if (info?.lemma_id) {
    const next = { ...masteryByLemma.value, [info.lemma_id]: Math.max(masteryByLemma.value[info.lemma_id] ?? 0, 3) };
    masteryByLemma.value = next;
    saveMastery(selectedLanguageCode.value, next);
  }
  index.value = Math.max(0, Math.min(index.value, filteredWords.value.length - 1));
  // setBackgroundForIndex(index.value);
}

function skip() {
  const cw = currentWord.value;
  const info = cw ? wordformLookup.value.get(cw.word?.toLowerCase?.()) : null;
  if (info?.lemma_id) {
    const current = masteryByLemma.value[info.lemma_id] ?? 0;
    if (current === 0) {
      const next = { ...masteryByLemma.value, [info.lemma_id]: 1 };
      masteryByLemma.value = next;
      saveMastery(selectedLanguageCode.value, next);
    }
  }
  go(1);
}

function onTapAnywhere(e) {
  if (viewMode.value !== "cards") return;
  if (justFinishedHoldSkipping.value) return;
  if (!e.target?.closest?.(".card")) return;
  const tag = e.target?.tagName?.toLowerCase?.();
  if (tag === "button" || tag === "select" || tag === "option") return;
  go(1);
}

const swipeStart = ref({ x: 0, y: 0 });
const currentPointer = ref({ x: 0, y: 0 });
const SWIPE_THRESHOLD = 50;

const holdSkipTimer = ref(null);
const holdSkipIntervalId = ref(null);
const justFinishedHoldSkipping = ref(false);
let holdSkipReleaseCleanup = null;

const HOLD_INITIAL_DELAY = 220;
const HOLD_MIN_DELAY = 45;
const HOLD_ACCEL = 0.92;

function clearHoldSkip() {
  if (holdSkipTimer.value != null) {
    clearTimeout(holdSkipTimer.value);
    holdSkipTimer.value = null;
  }
  if (holdSkipIntervalId.value != null) {
    clearTimeout(holdSkipIntervalId.value);
    holdSkipIntervalId.value = null;
    justFinishedHoldSkipping.value = true;
    setTimeout(() => { justFinishedHoldSkipping.value = false; }, 200);
    if (holdSkipReleaseCleanup) {
      holdSkipReleaseCleanup();
      holdSkipReleaseCleanup = null;
    }
  }
}

function scheduleHoldRepeat(repeatAction, delay) {
  holdSkipIntervalId.value = window.setTimeout(() => {
    repeatAction();
    const nextDelay = Math.max(HOLD_MIN_DELAY, delay * HOLD_ACCEL);
    scheduleHoldRepeat(repeatAction, nextDelay);
  }, delay);
}

function onCardTouchMove(e) {
  if (!e.touches?.length && e.buttons === 0) return;
  if (e.touches?.length) {
    currentPointer.value = { x: e.touches[0].clientX, y: e.touches[0].clientY };
  } else {
    currentPointer.value = { x: e.clientX, y: e.clientY };
  }
}

function onCardTouchStart(e) {
  if (e.touches?.length) {
    swipeStart.value = { x: e.touches[0].clientX, y: e.touches[0].clientY };
    currentPointer.value = { x: e.touches[0].clientX, y: e.touches[0].clientY };
  } else {
    swipeStart.value = { x: e.clientX, y: e.clientY };
    currentPointer.value = { x: e.clientX, y: e.clientY };
  }
  clearHoldSkip();
  holdSkipTimer.value = window.setTimeout(() => {
    holdSkipTimer.value = null;
    const dx = currentPointer.value.x - swipeStart.value.x;
    const repeatAction = dx < -SWIPE_THRESHOLD ? () => go(-1) : () => go(1);
    scheduleHoldRepeat(repeatAction, HOLD_INITIAL_DELAY);
    const onRelease = () => clearHoldSkip();
    holdSkipReleaseCleanup = () => {
      window.removeEventListener("touchend", onRelease, true);
      window.removeEventListener("mouseup", onRelease, true);
    };
    window.addEventListener("touchend", onRelease, true);
    window.addEventListener("mouseup", onRelease, true);
  }, 400);
}

function onCardTouchEnd(e) {
  const wasHoldSkipping = holdSkipIntervalId.value != null;
  clearHoldSkip();
  if (wasHoldSkipping) return;

  let endX, endY;
  if (e.changedTouches?.length) {
    endX = e.changedTouches[0].clientX;
    endY = e.changedTouches[0].clientY;
  } else {
    endX = e.clientX;
    endY = e.clientY;
  }
  const dx = endX - swipeStart.value.x;
  const dy = endY - swipeStart.value.y;
  const absX = Math.abs(dx);
  const absY = Math.abs(dy);
  if (absX < SWIPE_THRESHOLD && absY < SWIPE_THRESHOLD) return;
  if (absX > absY) {
    if (dx < 0) go(-1);
    else go(1);
  } else {
    // Vertical swipe: no-op in static mode
  }
}

// --- View mode: cards | reading | generate ---
const viewMode = ref("cards");
const menuOpen = ref(false);
const readingText = ref("");

function tokenizeForHighlight(text) {
  if (!text.trim()) return [];
  const tokens = [];
  const re = /[\p{L}\p{M}\p{N}']+/gu;
  let lastEnd = 0;
  let m;
  while ((m = re.exec(text)) !== null) {
    if (m.index > lastEnd) {
      tokens.push({ type: "punctuation", value: text.slice(lastEnd, m.index), level: null });
    }
    const word = m[0];
    const lower = word.toLowerCase();
    const info = wordformLookup.value.get(lower);
    let level = null;
    if (info) {
      level = masteryByLemma.value[info.lemma_id] ?? 0;
    }
    tokens.push({ type: "word", value: word, lemma_id: info?.lemma_id, level, english_gloss: info?.english_gloss ?? "" });
    lastEnd = re.lastIndex;
  }
  if (lastEnd < text.length) {
    tokens.push({ type: "punctuation", value: text.slice(lastEnd), level: null });
  }
  return tokens;
}

const highlightedTokens = computed(() => tokenizeForHighlight(readingText.value));

function cycleWordState(token) {
  if (token.type !== "word" || !token.lemma_id) return;
  const wordform = token.value.toLowerCase();
  const lid = token.lemma_id;
  const currentMastery = masteryByLemma.value[lid] ?? 0;
  const isConfirmed = knownSet.value.has(wordform) || currentMastery >= 3;
  if (isConfirmed) {
    const nextSet = new Set(knownSet.value);
    nextSet.delete(wordform);
    knownSet.value = nextSet;
    saveKnownSet(selectedLanguageCode.value, nextSet);
    masteryByLemma.value = { ...masteryByLemma.value, [lid]: 1 };
    saveMastery(selectedLanguageCode.value, masteryByLemma.value);
  } else if (currentMastery === 1) {
    masteryByLemma.value = { ...masteryByLemma.value, [lid]: 0 };
    saveMastery(selectedLanguageCode.value, masteryByLemma.value);
  } else {
    const nextSet = new Set(knownSet.value);
    nextSet.add(wordform);
    knownSet.value = nextSet;
    saveKnownSet(selectedLanguageCode.value, nextSet);
    masteryByLemma.value = { ...masteryByLemma.value, [lid]: 3 };
    saveMastery(selectedLanguageCode.value, masteryByLemma.value);
  }
}

watch(
  selectedLanguageCode,
  (code) => {
    if (!code) return;
    loadLanguageWords(code);
  },
  { immediate: true }
);

watch(filteredWords, (list) => {
  index.value = Math.max(0, Math.min(index.value, list.length - 1));
});

onMounted(() => {
  window.addEventListener("keydown", (e) => {
    if (e.key === "ArrowRight" || e.key === " ") {
      e.preventDefault();
      go(1);
    } else if (e.key === "ArrowLeft") {
      e.preventDefault();
      go(-1);
    } else if (e.key === "Enter") {
      e.preventDefault();
      markKnown();
    }
  });
});
</script>

<template>
  <div class="app-root" @click="onTapAnywhere">
    <div
      class="bg-layer"
      :style="{
        backgroundImage: currentImageUrl ? 'url(' + currentImageUrl + ')' : 'none',
        opacity: showNextAsCurrent ? 0 : 1,
      }"
    ></div>
    <div
      class="bg-layer next"
      :style="{
        backgroundImage: nextImageUrl ? 'url(' + nextImageUrl + ')' : 'none',
        opacity: showNextAsCurrent ? 1 : 0,
      }"
    ></div>
    <div class="bg-overlay"></div>

    <div class="content">
      <div class="top-row">
        <div class="app-title">
          <div class="app-title-main">Language Index</div>
          <div class="app-title-sub">
            Gently mark what you truly know.
          </div>
        </div>
        <button
          type="button"
          class="hamburger-btn"
          aria-label="Menu"
          aria-expanded="menuOpen"
          @click.stop="menuOpen = !menuOpen"
        >
          <span class="hamburger-icon">☰</span>
        </button>
      </div>

      <!-- Hamburger menu overlay (top right) -->
      <Transition name="menu">
        <div v-if="menuOpen" class="menu-overlay" @click.self="menuOpen = false">
          <div class="menu-panel">
            <nav class="view-tabs" aria-label="View mode">
              <button type="button" class="tab active">Cards</button>
            </nav>
            <div class="menu-section">
              <span class="generate-label">Language</span>
              <div class="lang-chips">
                <button
                  v-for="lang in languages"
                  :key="lang.code"
                  type="button"
                  class="ghost-btn"
                  :class="{ 'lang-active': lang.code === selectedLanguageCode }"
                  @click.stop="selectedLanguageCode = lang.code; menuOpen = false"
                >
                  <span class="dot"></span>
                  {{ lang.label }}
                </button>
              </div>
            </div>
            <div class="menu-section">
              <button
                type="button"
                class="chip chip-clickable"
                :title="knownCount ? 'View confirmed words' : 'No confirmed words yet'"
                @click.stop="toggleSidebar(); menuOpen = false"
              >
                <span class="chip-dot"></span>
                <span v-if="knownCount === 0">No words confirmed yet</span>
                <span v-else>{{ knownCount }} confirmed</span>
              </button>
              <button
                type="button"
                class="ghost-btn"
                @click.stop="clearProgress(); menuOpen = false"
              >
                <span class="dot"></span>
                Clear progress
              </button>
            </div>
          </div>
        </div>
      </Transition>

      <div class="center" v-if="viewMode === 'cards'">
        <div v-if="status === 'loading'" class="status">
          <span>Preparing your deck…</span>
        </div>

        <div v-else-if="status === 'error'" class="status">
          <span>{{ statusMessage || "Could not load this deck." }}</span>
        </div>

        <div v-else-if="!currentWord" class="status">
          <span>All words in this deck are confirmed. Change language or add more words.</span>
        </div>

        <div
          v-else
          class="card"
          @touchstart="onCardTouchStart"
          @touchmove="onCardTouchMove"
          @touchend="onCardTouchEnd"
          @mousedown="onCardTouchStart"
          @mousemove="onCardTouchMove"
          @mouseup="onCardTouchEnd"
        >
          <div class="card-header">
            <div class="card-word">
              {{ currentWord.word }}
            </div>
          </div>
          <div class="card-meta">
            <div v-if="currentWord.english" class="english-pill">
              {{ currentWord.english }}
            </div>
            <div v-if="currentWord.person" class="pos-pill person-pill">
              {{ currentWord.person }}
            </div>
          </div>
          <div class="card-actions">
            <button class="ghost-btn icon-btn" type="button" title="Still learning" @click.stop="skip">
              ❔
            </button>
            <button class="ghost-btn primary icon-btn" type="button" title="I know this one" @click.stop="markKnown">
              👍
            </button>
          </div>
          <div class="card-counter">
            <span class="card-counter-nav">{{ knownCount }}</span>
            <span class="card-counter-remaining">{{ unknownLeftCount }} left</span>
          </div>
        </div>
      </div>

      <div class="bottom-row">
        <div class="tap-hint">
          Tap or swipe. Hold to fast-navigate.
        </div>
      </div>
    </div>

    <!-- Confirmed words sidebar -->
    <div class="sidebar-backdrop" :class="{ open: sidebarOpen }" @click="closeSidebar" aria-hidden="true"></div>
    <aside class="sidebar-panel" :class="{ open: sidebarOpen }" aria-label="Confirmed words list">
      <div class="sidebar-header">
        <h2 class="sidebar-title">Confirmed words</h2>
        <button type="button" class="sidebar-close" title="Close" @click="closeSidebar" aria-label="Close">×</button>
      </div>
      <div class="sidebar-content">
        <p v-if="confirmedWordsList.length === 0" class="sidebar-empty">No words confirmed yet.</p>
        <ul v-else class="sidebar-list">
          <li v-for="word in confirmedWordsList" :key="word" class="sidebar-item">{{ word }}</li>
        </ul>
      </div>
    </aside>
  </div>
</template>

