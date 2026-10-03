<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'

const WEEKDAYS = ['日', '一', '二', '三', '四', '五', '六']
const STORAGE_KEY = 'vue3clock:settings'

function loadSettings() {
  try {
    return JSON.parse(localStorage.getItem(STORAGE_KEY)) || {}
  } catch {
    return {}
  }
}

const saved = loadSettings()
const now = ref(new Date())
const is24h = ref(saved.is24h ?? true)
const showAnalog = ref(saved.showAnalog ?? true)
const keepAwake = ref(false)
const wakeLockSupported = typeof navigator !== 'undefined' && 'wakeLock' in navigator

watch([is24h, showAnalog], () => {
  try {
    localStorage.setItem(
      STORAGE_KEY,
      JSON.stringify({ is24h: is24h.value, showAnalog: showAnalog.value }),
    )
  } catch {
    /* storage unavailable (private mode) — settings just won't persist */
  }
})

// Tick aligned to the start of each second so the display never lags.
let timer = null
function tick() {
  now.value = new Date()
  timer = setTimeout(tick, 1000 - (Date.now() % 1000))
}

const pad = (n) => String(n).padStart(2, '0')

const hours = computed(() => {
  const h = now.value.getHours()
  return is24h.value ? pad(h) : String(h % 12 || 12)
})
const minutes = computed(() => pad(now.value.getMinutes()))
const seconds = computed(() => pad(now.value.getSeconds()))
const meridiem = computed(() => (now.value.getHours() < 12 ? '上午' : '下午'))

const dateText = computed(() => {
  const d = now.value
  return `${d.getFullYear()} 年 ${d.getMonth() + 1} 月 ${d.getDate()} 日`
})
const weekdayText = computed(() => `星期${WEEKDAYS[now.value.getDay()]}`)
const timeZone = Intl.DateTimeFormat().resolvedOptions().timeZone

// Analog hand angles (degrees).
const secondDeg = computed(() => now.value.getSeconds() * 6)
const minuteDeg = computed(() => now.value.getMinutes() * 6 + now.value.getSeconds() * 0.1)
const hourDeg = computed(() => (now.value.getHours() % 12) * 30 + now.value.getMinutes() * 0.5)

const ticks = Array.from({ length: 60 }, (_, i) => i)

// Screen Wake Lock — keeps the phone screen on while the clock is visible.
let wakeLock = null
async function requestWakeLock() {
  try {
    wakeLock = await navigator.wakeLock.request('screen')
    wakeLock.addEventListener('release', () => {
      wakeLock = null
    })
  } catch {
    keepAwake.value = false
  }
}
async function toggleWakeLock() {
  keepAwake.value = !keepAwake.value
  if (keepAwake.value) {
    await requestWakeLock()
  } else if (wakeLock) {
    await wakeLock.release()
    wakeLock = null
  }
}
// The browser drops the lock when the tab is hidden; re-acquire on return.
function onVisibilityChange() {
  if (document.visibilityState !== 'visible') return
  now.value = new Date()
  if (keepAwake.value && !wakeLock) requestWakeLock()
}

async function toggleFullscreen() {
  try {
    if (document.fullscreenElement) await document.exitFullscreen()
    else await document.documentElement.requestFullscreen()
  } catch {
    /* fullscreen not supported (e.g. iOS Safari) */
  }
}

onMounted(() => {
  tick()
  document.addEventListener('visibilitychange', onVisibilityChange)
})
onBeforeUnmount(() => {
  clearTimeout(timer)
  document.removeEventListener('visibilitychange', onVisibilityChange)
  if (wakeLock) wakeLock.release()
})
</script>

<template>
  <main class="clock" :class="{ 'no-analog': !showAnalog }">
    <svg v-if="showAnalog" class="analog" viewBox="0 0 200 200" aria-hidden="true">
      <circle class="face" cx="100" cy="100" r="96" />
      <line
        v-for="i in ticks"
        :key="i"
        :class="i % 5 === 0 ? 'tick major' : 'tick'"
        x1="100"
        :y1="i % 5 === 0 ? 10 : 12"
        x2="100"
        y2="20"
        :transform="`rotate(${i * 6} 100 100)`"
      />
      <line class="hand hour" x1="100" y1="112" x2="100" y2="52" :transform="`rotate(${hourDeg} 100 100)`" />
      <line class="hand minute" x1="100" y1="116" x2="100" y2="30" :transform="`rotate(${minuteDeg} 100 100)`" />
      <line class="hand second" x1="100" y1="122" x2="100" y2="24" :transform="`rotate(${secondDeg} 100 100)`" />
      <circle class="pin" cx="100" cy="100" r="4" />
    </svg>

    <section class="digital" aria-live="off">
      <div class="time" role="timer" :aria-label="`${hours}:${minutes}:${seconds}`">
        <span v-if="!is24h" class="meridiem">{{ meridiem }}</span>
        <span>{{ hours }}</span><span class="colon">:</span><span>{{ minutes }}</span>
        <span class="seconds">{{ seconds }}</span>
      </div>
      <div class="date">{{ dateText }}　{{ weekdayText }}</div>
      <div class="tz">{{ timeZone }}</div>
    </section>

    <nav class="controls">
      <button type="button" @click="is24h = !is24h">{{ is24h ? '24 小時' : '12 小時' }}</button>
      <button type="button" @click="showAnalog = !showAnalog">
        {{ showAnalog ? '隱藏錶盤' : '顯示錶盤' }}
      </button>
      <button v-if="wakeLockSupported" type="button" :class="{ active: keepAwake }" @click="toggleWakeLock">
        {{ keepAwake ? '螢幕常亮 開' : '螢幕常亮 關' }}
      </button>
      <button type="button" @click="toggleFullscreen">全螢幕</button>
    </nav>
  </main>
</template>

<style scoped>
.clock {
  min-height: 100dvh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: clamp(16px, 4vh, 40px);
  padding: calc(16px + env(safe-area-inset-top)) calc(16px + env(safe-area-inset-right))
    calc(16px + env(safe-area-inset-bottom)) calc(16px + env(safe-area-inset-left));
  box-sizing: border-box;
}

.analog {
  width: min(78vw, 46vh, 360px);
  height: auto;
}
.face {
  fill: var(--face);
  stroke: var(--accent);
  stroke-width: 3;
}
.tick {
  stroke: var(--muted);
  stroke-width: 1;
}
.tick.major {
  stroke: var(--text);
  stroke-width: 3;
}
.hand {
  stroke-linecap: round;
}
.hand.hour {
  stroke: var(--text);
  stroke-width: 7;
}
.hand.minute {
  stroke: var(--text);
  stroke-width: 4;
}
.hand.second {
  stroke: var(--accent);
  stroke-width: 2;
}
.pin {
  fill: var(--accent);
}

.digital {
  text-align: center;
}
.time {
  font-size: clamp(56px, 20vw, 160px);
  font-weight: 200;
  line-height: 1;
  font-variant-numeric: tabular-nums;
  letter-spacing: 0.02em;
}
.no-analog .time {
  font-size: clamp(72px, 26vw, 220px);
}
.colon {
  animation: blink 1s steps(1) infinite;
}
.seconds {
  font-size: 0.35em;
  margin-left: 0.15em;
  color: var(--accent);
}
.meridiem {
  font-size: 0.22em;
  margin-right: 0.3em;
  color: var(--muted);
  vertical-align: middle;
}
.date {
  margin-top: 12px;
  font-size: clamp(16px, 5vw, 28px);
  color: var(--text);
}
.tz {
  margin-top: 4px;
  font-size: clamp(12px, 3.5vw, 16px);
  color: var(--muted);
}

.controls {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 8px;
}
.controls button {
  min-height: 44px; /* comfortable touch target */
  padding: 0 16px;
  border-radius: 22px;
  border: 1px solid var(--border);
  background: var(--button);
  color: var(--text);
  font-size: 15px;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
}
.controls button:active {
  transform: scale(0.96);
}
.controls button.active {
  border-color: var(--accent);
  color: var(--accent);
}

/* Phone in landscape: put the dial beside the digits. */
@media (orientation: landscape) and (max-height: 500px) {
  .clock {
    flex-direction: row;
    flex-wrap: wrap;
    gap: 24px;
  }
  .analog {
    width: min(70vh, 40vw);
  }
  .time {
    font-size: clamp(48px, 12vw, 120px);
  }
  .controls {
    width: 100%;
  }
}

@keyframes blink {
  50% {
    opacity: 0.25;
  }
}
@media (prefers-reduced-motion: reduce) {
  .colon {
    animation: none;
  }
}
</style>
