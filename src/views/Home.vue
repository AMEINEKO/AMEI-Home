<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'

const stage = ref('loading')
const hitokoto = ref('May a gentle light find you each day.')
const hitokotoFrom = ref('')
const bgUrl = `https://www.dmoe.cc/random.php?t=${Date.now()}`
const bgImage = ref('none')
const MIN_LOADING_MS = 3000
const MAX_LOADING_MS = 10000
let readyTimer = 0
let minTimer = 0
let maxTimer = 0
let isUnmounted = false
let bgImageLoader = null

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
  } catch {
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
})

onBeforeUnmount(() => {
  isUnmounted = true
  window.clearTimeout(readyTimer)
  window.clearTimeout(minTimer)
  window.clearTimeout(maxTimer)
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
      </div>
    </div>
    <main class="page">
      <a-card id="hero" class="hero-card" :bordered="false" hoverable>
        <a-row :gutter="[24, 24]" align="middle">
          <a-col :xs="24" :sm="7" :md="5">
            <div class="hero-avatar">
              <a-avatar
                :size="96"
                shape="square"
                src="https://tc.azuneko.com/i/2026/03/24/3kpedj.webp"
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
                  <span class="lead-en">Code is not just a tool, it's a language to shape new worlds.</span>
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
  overflow: hidden;
}

.hero-avatar :deep(.ant-avatar img) {
  width: 100%;
  height: 100%;
  object-fit: cover;
  object-position: center;
  display: block;
  border-radius: 26px;
  transform: scale(1.06);
  transform-origin: center;
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

@keyframes fadeUp {
  from {
    opacity: 0;
    transform: translateY(16px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
