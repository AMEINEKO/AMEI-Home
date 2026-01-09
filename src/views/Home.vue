<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'

const stage = ref('loading')
const powReady = ref(false)
const powStatus = ref('正在执行访问校验…')
const powProgress = ref(0)
const powState = ref('working')
const powLogs = ref([])
const powLogHidden = ref(false)
const powLogHiding = ref(false)
const powStatusHiding = ref(false)
const powFadeMs = ref(1000)
const hitokoto = ref('May a gentle light find you each day.')
const hitokotoFrom = ref('')
const bgUrl = `https://www.dmoe.cc/random.php?t=${Date.now()}`
const bgImage = ref('none')
const MIN_LOADING_MS = 3000
const MAX_LOADING_MS = 10000
const DIFFICULTY_HEX_ZEROS = 4
const MOBILE_DIFFICULTY_HEX_ZEROS = 3
const CACHE_TTL_MS = 5 * 60 * 1000
const CACHE_KEY = 'pow_ok_until_v1'
const POW_REQUEST_MIN = 3
const POW_REQUEST_MAX = 10
const powEncoder = new TextEncoder()
const POW_MIN_MS = 3000
const POW_LOG_LIMIT = 6
const POW_SUCCESS_HOLD_MS = 1000
const POW_LOG_FADE_MS = 1000
const POW_CACHE_HOLD_MS = 800
const POW_CACHE_FADE_MS = 500
let readyTimer = 0
let minTimer = 0
let maxTimer = 0
let powReadyTimer = 0
let powLogTimer = 0
let powHoldTimer = 0
let isUnmounted = false
let bgImageLoader = null

const digestToHex = (buffer) =>
  Array.from(new Uint8Array(buffer))
    .map((byte) => byte.toString(16).padStart(2, '0'))
    .join('')

const generateRequestId = () => {
  if (crypto?.randomUUID) {
    return crypto.randomUUID()
  }
  return `${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 10)}`
}

const getPowDifficulty = () => {
  if (typeof window === 'undefined') {
    return DIFFICULTY_HEX_ZEROS
  }
  return window.matchMedia('(max-width: 640px)').matches
    ? MOBILE_DIFFICULTY_HEX_ZEROS
    : DIFFICULTY_HEX_ZEROS
}

const getRandomLimit = () =>
  Math.floor(Math.random() * (POW_REQUEST_MAX - POW_REQUEST_MIN + 1)) + POW_REQUEST_MIN

const normalizeCacheMeta = (meta) => {
  const until = Number(meta?.until)
  const windowStart = Number(meta?.windowStart)
  const windowCount = Number(meta?.windowCount)
  const windowLimit = Number(meta?.windowLimit)
  return {
    until: Number.isFinite(until) ? until : 0,
    windowStart: Number.isFinite(windowStart) ? windowStart : 0,
    windowCount: Number.isFinite(windowCount) ? windowCount : 0,
    windowLimit:
      Number.isFinite(windowLimit) && windowLimit >= POW_REQUEST_MIN && windowLimit <= POW_REQUEST_MAX
        ? windowLimit
        : getRandomLimit(),
    graceUsed: Boolean(meta?.graceUsed),
  }
}

const loadCacheMeta = () => {
  try {
    const rawValue = localStorage.getItem(CACHE_KEY)
    if (!rawValue) {
      return null
    }
    try {
      const parsed = JSON.parse(rawValue)
      if (parsed && typeof parsed === 'object') {
        return normalizeCacheMeta(parsed)
      }
    } catch (error) {
      // Fallback to legacy numeric format.
    }
    const until = Number(rawValue)
    if (!Number.isFinite(until)) {
      return null
    }
    return normalizeCacheMeta({ until })
  } catch (error) {
    return null
  }
}

const saveCacheMeta = (meta) => {
  try {
    localStorage.setItem(CACHE_KEY, JSON.stringify(meta))
  } catch (error) {
    // Ignore cache access failures.
  }
}

const readPowCache = () => {
  const now = Date.now()
  const meta = loadCacheMeta()
  if (!meta) {
    return { skip: false }
  }
  if (meta.until > now) {
    let windowStart = meta.windowStart
    let windowCount = meta.windowCount
    let windowLimit = meta.windowLimit
    if (!Number.isFinite(windowStart) || windowStart <= 0 || now - windowStart > CACHE_TTL_MS) {
      windowStart = now
      windowCount = 0
      windowLimit = getRandomLimit()
    }
    windowCount += 1
    const updated = {
      ...meta,
      graceUsed: false,
      windowStart,
      windowCount,
      windowLimit,
    }
    saveCacheMeta(updated)
    if (windowCount <= windowLimit) {
      return { skip: true, cached: true, reason: 'limit', windowCount, windowLimit }
    }
    return { skip: false, reason: 'limit', windowCount, windowLimit }
  }
  if (!meta.graceUsed) {
    const updated = { ...meta, graceUsed: true }
    saveCacheMeta(updated)
    return { skip: true, cached: true, reason: 'grace' }
  }
  return { skip: false, reason: 'expired' }
}

const writePowCache = () => {
  const now = Date.now()
  const meta = {
    until: now + CACHE_TTL_MS,
    windowStart: now,
    windowCount: 0,
    windowLimit: getRandomLimit(),
    graceUsed: false,
  }
  saveCacheMeta(meta)
}

const estimatePowProgress = (nonce, expectedNonces) => {
  if (!Number.isFinite(expectedNonces) || expectedNonces <= 0) {
    return 0
  }
  return Math.min(0.98, nonce / expectedNonces)
}

const formatElapsed = (ms) => `${(ms / 1000).toFixed(2)}s`

const formatRate = (rate) => {
  if (!Number.isFinite(rate) || rate <= 0) {
    return '0/s'
  }
  if (rate >= 1000) {
    return `${(rate / 1000).toFixed(1)}k/s`
  }
  return `${Math.round(rate)}/s`
}

const pushPowLog = (line) => {
  powLogs.value.push(line)
  if (powLogs.value.length > POW_LOG_LIMIT) {
    powLogs.value.shift()
  }
}

const runPow = async () => {
  if (!crypto?.subtle) {
    throw new Error('WebCrypto is unavailable.')
  }
  window.clearTimeout(powLogTimer)
  window.clearTimeout(powReadyTimer)
  window.clearTimeout(powHoldTimer)
  powLogHidden.value = false
  powLogHiding.value = false
  powStatusHiding.value = false
  powFadeMs.value = POW_LOG_FADE_MS
  powLogs.value = []
  const requestId = generateRequestId()
  pushPowLog(`请求ID: ${requestId}`)
  const cacheResult = readPowCache()
  if (cacheResult?.skip) {
    if (cacheResult.reason === 'grace') {
      powStatus.value = '缓存过期，首次免验证'
    } else {
      powStatus.value = '缓存命中'
    }
    powProgress.value = 1
    powState.value = 'cached'
    if (cacheResult.reason === 'grace') {
      pushPowLog('缓存过期，首次免验证')
    } else if (cacheResult.reason === 'limit') {
      pushPowLog(`缓存命中，本窗口 ${cacheResult.windowCount}/${cacheResult.windowLimit}`)
    } else {
      pushPowLog('缓存命中，跳过计算')
    }
    return { cached: true }
  }
  powStatus.value = '正在执行访问校验…'
  if (cacheResult?.reason === 'limit') {
    pushPowLog(`访问次数超限(${cacheResult.windowCount}/${cacheResult.windowLimit})，开始验证`)
  } else if (cacheResult?.reason === 'expired') {
    pushPowLog('缓存已过期，开始验证')
  }
  const difficultyZeros = getPowDifficulty()
  const expectedNonces = Math.pow(16, difficultyZeros)
  const timeSlice = Math.floor(Date.now() / 60000)
  const challenge = `${window.location.origin}|${timeSlice}|${requestId}`
  const targetPrefix = '0'.repeat(difficultyZeros)
  let nonce = 0
  const startTime = performance.now()
  const minCompleteAt = startTime + POW_MIN_MS
  let lastYield = startTime
  let lastLogTime = startTime
  let lastLogNonce = 0
  let lastHex = ''

  powProgress.value = 0
  powState.value = 'working'
  pushPowLog(`挑战: ${window.location.origin}`)
  pushPowLog(`难度前缀: ${targetPrefix}`)
  pushPowLog(`最短用时: ${(POW_MIN_MS / 1000).toFixed(1)}s`)
  while (true) {
    const payload = `${challenge}|${nonce}`
    const digest = await crypto.subtle.digest('SHA-256', powEncoder.encode(payload))
    const hex = digestToHex(digest)
    lastHex = hex
    if (hex.startsWith(targetPrefix)) {
      const now = performance.now()
      if (now >= minCompleteAt) {
        powProgress.value = 1
        powState.value = 'verified'
        writePowCache()
        pushPowLog(`完成 @${formatElapsed(now - startTime)} nonce=${nonce}`)
        return { cached: false }
      }
      if (now - lastLogTime > 120) {
        pushPowLog(`提前命中 @${formatElapsed(now - startTime)}，继续计算`)
        lastLogTime = now
        lastLogNonce = nonce
      }
    }
    nonce += 1
    const now = performance.now()
    if (now - lastYield > 120) {
      if (isUnmounted) {
        return { aborted: true }
      }
      powProgress.value = estimatePowProgress(nonce, expectedNonces)
      const deltaMs = now - lastLogTime
      const deltaNonce = nonce - lastLogNonce
      if (deltaMs > 0) {
        const rate = (deltaNonce / deltaMs) * 1000
        const hashPreview = lastHex
          ? ` hash=${lastHex.slice(0, difficultyZeros + 6)}…`
          : ''
        pushPowLog(
          `t=${formatElapsed(now - startTime)} nonce=${nonce} rate=${formatRate(rate)}${hashPreview}`
        )
        lastLogTime = now
        lastLogNonce = nonce
      }
      await new Promise((resolve) => setTimeout(resolve, 0))
      lastYield = now
    }
  }
}

const fetchHitokoto = async () => {
  try {
    const response = await fetch('https://v1.hitokoto.cn?encode=json')
    if (!response.ok) {
      return
    }
    const data = await response.json()
    if (isUnmounted) {
      return
    }
    if (data?.hitokoto) {
      hitokoto.value = data.hitokoto
      hitokotoFrom.value = data.from || ''
    }
  } catch (error) {
    // Keep fallback copy on error.
  }
}

const startOpening = () => {
  if (stage.value !== 'loading') {
    return
  }
  stage.value = 'opening'
  readyTimer = window.setTimeout(() => {
    stage.value = 'ready'
  }, 1200)
}

onMounted(() => {
  let minElapsed = false
  let timedOut = false
  let imageLoaded = false
  const img = new Image()
  bgImageLoader = img

  const maybeOpen = () => {
    if (isUnmounted) {
      return
    }
    if (stage.value !== 'loading') {
      return
    }
    if (!powReady.value) {
      return
    }
    if (!minElapsed) {
      return
    }
    if (!imageLoaded && !timedOut) {
      return
    }
    startOpening()
  }

  img.onload = () => {
    if (timedOut || bgImageLoader !== img || isUnmounted) {
      return
    }
    imageLoaded = true
    window.clearTimeout(maxTimer)
    maxTimer = 0
    bgImage.value = `url("${bgUrl}")`
    maybeOpen()
  }
  img.onerror = () => {
    if (timedOut || bgImageLoader !== img || isUnmounted) {
      return
    }
    timedOut = true
    window.clearTimeout(maxTimer)
    maxTimer = 0
    bgImage.value = 'none'
    maybeOpen()
  }
  img.src = bgUrl
  minTimer = window.setTimeout(() => {
    minElapsed = true
    maybeOpen()
  }, MIN_LOADING_MS)
  maxTimer = window.setTimeout(() => {
    if (imageLoaded) {
      return
    }
    timedOut = true
    bgImage.value = 'none'
    maybeOpen()
  }, MAX_LOADING_MS)
  fetchHitokoto()

  runPow()
    .then((result) => {
      if (isUnmounted || result?.aborted) {
        return
      }
      const isCached = Boolean(result?.cached)
      const fadeMs = isCached ? POW_CACHE_FADE_MS : POW_LOG_FADE_MS
      const holdMs = isCached ? POW_CACHE_HOLD_MS : 0
      powFadeMs.value = fadeMs
      powProgress.value = 1
      powState.value = isCached ? 'cached' : 'verified'
      window.clearTimeout(powLogTimer)
      window.clearTimeout(powHoldTimer)

      const startFadeOut = () => {
        if (isUnmounted) {
          return
        }
        powLogHiding.value = true
        powStatusHiding.value = true
        window.clearTimeout(powLogTimer)
        powLogTimer = window.setTimeout(() => {
          if (isUnmounted) {
            return
          }
          powLogs.value = []
          powLogHidden.value = true
          powLogHiding.value = false
          powStatus.value = '✓ 访问校验完成'
          powStatusHiding.value = false
          window.clearTimeout(powReadyTimer)
          powReadyTimer = window.setTimeout(() => {
            if (isUnmounted) {
              return
            }
            powReady.value = true
            maybeOpen()
          }, POW_SUCCESS_HOLD_MS)
        }, fadeMs)
      }

      if (holdMs > 0) {
        powHoldTimer = window.setTimeout(startFadeOut, holdMs)
      } else {
        startFadeOut()
      }
    })
    .catch((error) => {
      if (isUnmounted) {
        return
      }
      window.clearTimeout(powLogTimer)
      window.clearTimeout(powReadyTimer)
      window.clearTimeout(powHoldTimer)
      powProgress.value = 0
      powState.value = 'failed'
      powLogHiding.value = false
      powLogHidden.value = false
      powStatusHiding.value = false
      powStatus.value = '访问校验失败，请刷新'
      pushPowLog('验证失败，请刷新重试')
      console.error(error)
    })
})

onBeforeUnmount(() => {
  isUnmounted = true
  window.clearTimeout(readyTimer)
  window.clearTimeout(minTimer)
  window.clearTimeout(maxTimer)
  window.clearTimeout(powReadyTimer)
  window.clearTimeout(powLogTimer)
  window.clearTimeout(powHoldTimer)
  if (bgImageLoader) {
    bgImageLoader.onload = null
    bgImageLoader.onerror = null
    bgImageLoader.src = ''
    bgImageLoader = null
  }
})
</script>

<template>
  <div class="home" :data-stage="stage" :style="{ '--bg-image': bgImage }">
    <div class="loader" aria-hidden="true">
      <div class="loader-core">
        <div class="loader-orbit"></div>
        <div
          class="loader-status"
          v-if="stage === 'loading'"
          :style="{ '--log-fade-ms': `${powFadeMs}ms` }"
        >
          <div
            class="loader-progress"
            :data-state="powState"
            role="progressbar"
            :aria-valuenow="Math.round(powProgress * 100)"
            aria-valuemin="0"
            aria-valuemax="100"
          >
            <div
              class="loader-progress-bar"
              :style="{ width: `${Math.round(powProgress * 100)}%` }"
            ></div>
          </div>
          <div class="loader-progress-text" :class="{ 'is-hiding': powStatusHiding }">
            <span class="loader-status-text">{{ powStatus }}</span>
            <span
              v-if="powState === 'working' || powStatusHiding"
              class="loader-status-spinner"
              aria-hidden="true"
            >
              <span class="dot"></span>
              <span class="dot"></span>
              <span class="dot"></span>
            </span>
          </div>
          <div
            class="loader-log"
            :class="{ 'is-hiding': powLogHiding, 'is-hidden': powLogHidden }"
          >
            <div
              v-for="(line, index) in powLogs"
              :key="`pow-log-${index}`"
              class="loader-log-line"
            >
              {{ line }}
            </div>
          </div>
        </div>
      </div>
    </div>
    <main class="page">
      <a-card
        id="hero"
        class="hero-card"
        :bordered="false"
        hoverable
      >
        <a-row :gutter="[24, 24]" align="middle">
          <a-col :xs="24" :sm="7" :md="5">
            <div class="hero-avatar">
              <a-avatar
                :size="96"
                shape="square"
                src="https://easyimage2.azuneko.com/i/2025/09/14/12lwvxm.webp"
                alt="AMEI avatar"
              />
            </div>
          </a-col>
          <a-col :xs="24" :sm="17" :md="19">
            <a-space direction="vertical" size="middle" class="hero-content">
              <div>
                <a-typography-title :level="1" class="hero-title">AMEI</a-typography-title>
                <a-typography-paragraph class="hero-lead">
                  <span class="lead-zh">代码不是工具，而是改变世界的语言。</span>
                  <span class="lead-en">Code is not just a tool — it’s a language to shape new worlds.</span>
                </a-typography-paragraph>
              </div>
              <a-space size="small" wrap class="hero-actions">
                <a-button
                  type="primary"
                  shape="round"
                  href="https://github.com/AMEINEKO"
                  target="_blank"
                  rel="noreferrer"
                >
                  GitHub
                </a-button>
                <a-button
                  shape="round"
                  href="https://komari.azuneko.com/"
                  target="_blank"
                  rel="noreferrer"
                >
                  Komari Monitor
                </a-button>
                <a-button
                  shape="round"
                  href="https://blog.azuneko.com/"
                  target="_blank"
                  rel="noreferrer"
                >
                  Blog
                </a-button>
              </a-space>
              <div class="fun-section">
                <a-typography-title :level="4" class="fun-title">好玩的东西</a-typography-title>
                <a-space size="small" wrap class="fun-actions">
                  <a-button
                    shape="round"
                    href="https://www.bigtech-simulator.com/"
                    target="_blank"
                    rel="noreferrer"
                  >
                    互联网大厂模拟器
                  </a-button>
                </a-space>
              </div>
            </a-space>
          </a-col>
        </a-row>
      </a-card>
    </main>
    <footer class="site-footer">
      <div class="footer-inner">
        <p class="footer-copy">
          "{{ hitokoto }}"
          <span v-if="hitokotoFrom" class="footer-from">- {{ hitokotoFrom }}</span>
        </p>
      </div>
    </footer>
  </div>
</template>

<style scoped>
.loader-core {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0;
  position: relative;
  z-index: 1;
}

.loader-status {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  width: min(260px, 70vw);
  margin-top: 1cm;
}

.loader-progress {
  width: 100%;
  height: 10px;
  border-radius: 999px;
  background: rgba(246, 247, 249, 0.2);
  border: 1px solid rgba(246, 247, 249, 0.3);
  overflow: hidden;
  box-shadow: 0 0 16px rgba(154, 220, 210, 0.18);
}

.loader-progress-bar {
  height: 100%;
  width: 0%;
  border-radius: inherit;
  background: linear-gradient(90deg, rgba(154, 220, 210, 0.35), rgba(154, 220, 210, 0.9));
  transition: width 0.2s ease;
}

.loader-progress[data-state='cached'] .loader-progress-bar,
.loader-progress[data-state='verified'] .loader-progress-bar {
  background: linear-gradient(90deg, rgba(154, 220, 210, 0.55), rgba(154, 220, 210, 1));
}

.loader-progress[data-state='failed'] .loader-progress-bar {
  background: linear-gradient(90deg, rgba(239, 68, 68, 0.45), rgba(239, 68, 68, 0.95));
}

.loader-progress-text {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  font-size: 0.85rem;
  color: rgba(246, 247, 249, 0.72);
  letter-spacing: 0.02em;
  text-align: center;
  max-width: 260px;
  transition: opacity var(--log-fade-ms, 1000ms) ease, transform var(--log-fade-ms, 1000ms) ease;
}

.loader-progress-text.is-hiding {
  opacity: 0;
  transform: translateY(6px);
}

.loader-status-text {
  display: inline-block;
  white-space: nowrap;
}

.loader-status-spinner {
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.loader-status-spinner .dot {
  width: 6px;
  height: 6px;
  border-radius: 999px;
  background: rgba(246, 247, 249, 0.8);
  animation: status-bounce 0.9s infinite ease-in-out;
}

.loader-status-spinner .dot:nth-child(2) {
  animation-delay: 0.15s;
}

.loader-status-spinner .dot:nth-child(3) {
  animation-delay: 0.3s;
}

.loader-log {
  width: 100%;
  font-size: 0.75rem;
  color: rgba(246, 247, 249, 0.58);
  line-height: 1.45;
  text-align: center;
  font-variant-numeric: tabular-nums;
  margin-top: 2px;
  max-height: 160px;
  opacity: 1;
  overflow: hidden;
  transition: opacity var(--log-fade-ms, 1000ms) ease,
    max-height var(--log-fade-ms, 1000ms) ease,
    margin-top var(--log-fade-ms, 1000ms) ease;
}

.loader-log.is-hiding {
  opacity: 0;
  max-height: 0;
  margin-top: 0;
}

.loader-log.is-hidden {
  display: none;
}

.loader-log-line {
  word-break: break-word;
}

.hero-card {
  border-radius: 18px;
  border: 1px solid var(--line);
  background: var(--glass);
  box-shadow: 0 18px 36px rgba(0, 0, 0, 0.35);
  backdrop-filter: blur(18px);
  -webkit-backdrop-filter: blur(18px);
  overflow: hidden;
  opacity: 0;
  transform: translateY(16px);
  animation: fadeUp 0.9s ease forwards;
  animation-play-state: paused;
  transition: transform 0.4s ease, border-color 0.4s ease, box-shadow 0.4s ease;
}

.home[data-stage='ready'] .hero-card {
  animation-play-state: running;
}

.hero-card:hover {
  transform: translateY(-4px);
  border-color: rgba(154, 220, 210, 0.4);
  box-shadow: 0 26px 46px rgba(0, 0, 0, 0.45);
}

.hero-card :deep(.ant-card-body) {
  padding: clamp(28px, 4vw, 70px);
  background: transparent;
}

.hero-avatar {
  width: 96px;
  height: 96px;
  border-radius: 32px;
  padding: 4px;
  background: linear-gradient(140deg, rgba(154, 220, 210, 0.65), rgba(184, 166, 255, 0.55));
  border: 1px solid rgba(255, 255, 255, 0.16);
  box-shadow: 0 16px 30px rgba(0, 0, 0, 0.45);
  display: grid;
  place-items: center;
}

.hero-avatar :deep(.ant-avatar) {
  width: 100%;
  height: 100%;
  border-radius: 26px;
  background: rgba(0, 0, 0, 0.2);
}

.hero-avatar :deep(.ant-avatar img) {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  border-radius: 26px;
}

.hero-content {
  width: 100%;
}

.hero-actions {
  width: 100%;
}

.hero-actions :deep(.ant-btn) {
  border-color: rgba(255, 255, 255, 0.16);
  color: var(--muted);
  background: rgba(10, 12, 16, 0.45);
  box-shadow: none;
  transition: border-color 0.3s ease, color 0.3s ease, background 0.3s ease;
}

.hero-actions :deep(.ant-btn:hover) {
  color: var(--text);
  border-color: rgba(154, 220, 210, 0.6);
  background: rgba(154, 220, 210, 0.12);
}

.hero-actions :deep(.ant-btn-primary) {
  color: #0f1718;
  background: rgba(154, 220, 210, 0.85);
  border-color: rgba(154, 220, 210, 0.85);
}

.hero-actions :deep(.ant-btn-primary:hover) {
  background: rgba(154, 220, 210, 1);
  border-color: rgba(154, 220, 210, 1);
}

.fun-section {
  margin-top: 2rem;
  width: 100%;
}

.fun-title {
  font-size: 1.2rem;
  margin-bottom: 0.8rem;
  color: var(--text);
  font-weight: 500;
  text-align: left;
}

.fun-actions {
  width: 100%;
}

.fun-actions :deep(.ant-btn) {
  border-color: rgba(255, 255, 255, 0.16);
  color: var(--muted);
  background: rgba(10, 12, 16, 0.45);
  box-shadow: none;
  transition: border-color 0.3s ease, color 0.3s ease, background 0.3s ease;
}

.fun-actions :deep(.ant-btn:hover) {
  color: var(--text);
  border-color: rgba(154, 220, 210, 0.6);
  background: rgba(154, 220, 210, 0.12);
}

.hero-title {
  display: inline-block;
  font-family: 'Shippori Mincho', 'Times New Roman', serif;
  font-size: clamp(2.2rem, 4.8vw, 3.6rem);
  margin: 0;
  font-weight: 600;
  line-height: 1.05;
  background-image: linear-gradient(
    120deg,
    rgba(154, 220, 210, 0.55),
    rgba(246, 247, 249, 0.9),
    rgba(184, 166, 255, 0.55)
  );
  background-size: 200% 100%;
  background-position: 0% 50%;
  color: transparent;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  -webkit-background-clip: text;
  animation: title-shine 6s linear infinite;
}

.hero-lead {
  margin: 1rem 0 1.6rem;
  color: var(--muted);
  max-width: 32rem;
}

.hero-lead .lead-zh,
.hero-lead .lead-en {
  display: block;
}

.hero-lead .lead-zh {
  font-size: 1.05rem;
  color: var(--text);
  letter-spacing: 0.02em;
}

.hero-lead .lead-en {
  margin-top: 0.45rem;
  font-size: 0.95rem;
  color: rgba(246, 247, 249, 0.7);
}

@media (max-width: 820px) {
  .hero-avatar {
    margin: 0 auto;
  }

  .hero-content {
    text-align: center;
  }

  .hero-actions {
    justify-content: center;
  }

  .fun-title {
    text-align: center;
  }

  .fun-actions {
    justify-content: center;
  }
}

  @media (max-width: 600px) {
  .hero-card :deep(.ant-card-body) {
    padding-inline: 22px;
    padding-block: calc(22px + 1cm);
  }

  .hero-avatar {
    width: 80px;
    height: 80px;
    border-radius: 26px;
    transform: translateX(-8px);
  }

  .hero-avatar :deep(.ant-avatar) {
    border-radius: 22px;
  }

  .hero-title {
    font-size: clamp(1.9rem, 9vw, 2.6rem);
  }

  .hero-lead {
    margin: 0.8rem 0 1.2rem;
  }

  .hero-actions {
    flex-direction: column;
    align-items: stretch;
  }

  .hero-actions :deep(.ant-btn) {
    width: 100%;
    justify-content: center;
  }

  .fun-actions :deep(.ant-btn) {
    width: 100%;
    justify-content: center;
  }
}

@media (prefers-reduced-motion: reduce) {
  .hero-title {
    animation: none;
    background-position: 50% 50%;
  }
}

@keyframes title-shine {
  to {
    background-position: 200% 50%;
  }
}

@keyframes status-bounce {
  0%,
  80%,
  100% {
    transform: translateY(0);
    opacity: 0.4;
  }
  40% {
    transform: translateY(-4px);
    opacity: 1;
  }
}
</style>
