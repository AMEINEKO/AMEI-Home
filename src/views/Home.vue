<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'

const stage = ref('loading')
const hitokoto = ref('May a gentle light find you each day.')
const hitokotoFrom = ref('')
const bgUrl = `https://www.dmoe.cc/random.php?t=${Date.now()}`
const bgImage = ref('none')
const emoDiaryTitle = '写在夜里'
const emoDiarySubtitle = `Nana在遇到A妹以前 经历了一段刻骨铭心的感情 至今想起来 她的心还会隐隐作痛...`
const emoDiarySections = [
  [
    '如果你非要离开 那我就当你死了',
    '告别的话再不用说了 你尽管走得干脆一些',
    '我不会埋怨挣扎 因为这两年我已经闹够了',
    '我会在安静时难受 在不眠的床上想你',
  ],
  [
    '如果我非要责怪 那你就当我是个孩子',
    '原谅我当你死了 因为想你已没有意义',
    '我会在凌晨时候散一散步 等着第一家早餐开门',
    '我会养个动物填补我的寂寞 虽然它不能与我共枕',
  ],
  [
    '我不能不感谢你 否则我就是一个小人',
    '你曾说身体是恋爱的本钱 可惜我没有戒掉烟',
    '我不能不感谢你 你从来不嫌弃我没钱',
    '可是不管你画了多少未来 你现在已经离开',
  ],
  [],
  [],
  [],
  [
    '希望你能够明白',
    '所有的责任都在于我',
    '你可以当我死了',
    '可你有个宽广的胸怀',
  ],
  [
    '你会在回家的路上',
    '擦干眼泪',
    '不在你的母亲面前哭泣',
    '你会埋藏悲伤',
    '装出笑容',
    '可是你的心上已有烙印',
  ],
  [
    '我要让你知道',
    '我从不懂得怎么爱你',
    '在那些烂醉如泥的夜里',
    '我唯一想起的人是你',
    '我要让你知道',
    '与你比我是如此小气',
    '你曾说恋爱是两个人的事情',
    '不只是我一个人的原因',
  ],
  [
    '我们都已经开始自由',
    '我要做个压抑时放纵的坏人',
    '你不用再去过问我的生活',
    '我的好坏已经与你无关',
    '你一定会过得很好',
    '你是个不怎么爱漂亮的女人',
    '你可以追求你理想中的生活',
    '我们已是两条路上的人',
  ],
  [],
  [],
  [],
  [
    '我不能不感谢你 否则我就是一个小人',
    '你曾说身体是恋爱的本钱 至今我没有戒掉烟',
    '我不能不感谢你 你从来不嫌弃我没钱',
    '可是不管你画了多少个未来 你现在已经离开',
  ],
  [
    '我要让你知道 我从不懂得怎么爱你',
    '在那些烂醉如泥的夜里 我唯一想起的人是你',
    '我要让你知道 与你比我是如此小气',
    '你曾说恋爱是两个人的事情 不只是我一个人的原因',
  ],
  [
    '我们都已经开始自由 我要做个压抑时放纵的坏人',
    '你不用再去过问我的生活 我的好坏已经与你无关',
    '你一定会过得很好 你是个不怎么爱漂亮的女人',
    '你可以追求你理想中的生活',
    '我们已是两条路上的人'
  ],
  [
    '已是两条路上的人'
  ]
]
const MAX_LOADING_MS = 10000
let readyTimer = 0
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
  let timedOut = false
  let imageLoaded = false
  const img = new Image()
  bgImageLoader = img

  const openFromLoading = () => {
    if (isUnmounted) {
      return
    }
    if (stage.value !== 'loading') {
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
    openFromLoading()
  }
  img.onerror = () => {
    if (bgImageLoader !== img || isUnmounted) {
      return
    }
    bgImage.value = 'none'
  }
  img.src = bgUrl
  maxTimer = window.setTimeout(() => {
    if (imageLoaded) {
      return
    }
    timedOut = true
    bgImage.value = 'none'
    openFromLoading()
  }, MAX_LOADING_MS)
  fetchHitokoto()
})

onBeforeUnmount(() => {
  isUnmounted = true
  window.clearTimeout(readyTimer)
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
        <a-row :gutter="[24, 24]" align="top">
          <a-col :xs="24" :sm="7" :md="5" class="hero-avatar-col">
            <a-avatar class="hero-avatar" :size="96" shape="square"
              src="https://easyimage2.788848.xyz/i/2026/05/09/12u41j3.webp" alt="AMEI avatar" />
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
                <a-button type="primary" shape="round" href="https://github.com/AMEINEKO" target="_blank"
                  rel="noreferrer">
                  GitHub
                </a-button>
                <a-button shape="round" href="https://komari.azuneko.com/" target="_blank" rel="noreferrer">
                  Komari Monitor
                </a-button>
                <a-button shape="round" href="https://blog.azuneko.com/" target="_blank" rel="noreferrer">
                  Blog
                </a-button>
              </a-space>
              <div class="fun-section">
                <a-typography-title :level="4" class="fun-title">好玩的东西</a-typography-title>
                <a-space size="small" wrap class="fun-actions">
                  <a-button shape="round" href="https://www.bigtech-simulator.com/" target="_blank" rel="noreferrer">
                    互联网大厂模拟器
                  </a-button>
                </a-space>
              </div>
              <div class="fun-section">
                <a-typography-title :level="4" class="fun-title">Nana的emo时间</a-typography-title>
                <section class="emo-diary" aria-label="歌词日记">
                  <div class="emo-diary-head">
                    <p class="emo-diary-kicker">Diary Fragment</p>
                    <h3 class="emo-diary-title">{{ emoDiaryTitle }}</h3>
                    <p class="emo-diary-subtitle">{{ emoDiarySubtitle }}</p>
                  </div>
                  <div class="emo-diary-body">
                    <div v-for="(section, sectionIndex) in emoDiarySections" :key="`emo-section-${sectionIndex}`"
                      class="emo-diary-section">
                      <p v-for="(line, lineIndex) in section" :key="`emo-line-${sectionIndex}-${lineIndex}`"
                        class="emo-diary-line">
                        {{ line }}
                      </p>
                    </div>
                  </div>
                </section>
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

.hero-avatar-col {
  display: flex;
  justify-content: center;
}

.hero-avatar {
  border-radius: 26px;
  background: rgba(0, 0, 0, 0.2);
  box-shadow: 0 16px 30px rgba(0, 0, 0, 0.45);
}

.hero-avatar :deep(img) {
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

.emo-diary {
  position: relative;
  width: min(100%, 760px);
  padding: clamp(28px, 4vw, 42px) clamp(22px, 4vw, 40px);
  border-radius: 16px;
  border: 1px solid rgba(255, 255, 255, 0.08);
  background:
    linear-gradient(180deg, rgba(255, 248, 242, 0.12), rgba(255, 248, 242, 0.06)),
    rgba(17, 18, 24, 0.26);
  box-shadow:
    inset 0 1px 0 rgba(255, 255, 255, 0.08),
    0 18px 38px rgba(0, 0, 0, 0.2);
  overflow: hidden;
}

.emo-diary::before {
  content: '';
  position: absolute;
  inset: 0;
  background:
    linear-gradient(180deg, rgba(255, 255, 255, 0.05), transparent 22%),
    repeating-linear-gradient(180deg,
      transparent 0,
      transparent 39px,
      rgba(255, 255, 255, 0.035) 39px,
      rgba(255, 255, 255, 0.035) 40px);
  opacity: 0.45;
  pointer-events: none;
}

.emo-diary-head,
.emo-diary-body {
  position: relative;
  z-index: 1;
}

.emo-diary-head {
  margin-bottom: 28px;
}

.emo-diary-kicker {
  margin: 0 0 10px;
  font-family: 'Cormorant Garamond', 'Times New Roman', serif;
  font-size: 0.92rem;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  color: rgba(246, 247, 249, 0.48);
}

.emo-diary-title {
  margin: 0;
  font-family: 'Cormorant Garamond', 'Times New Roman', serif;
  font-size: clamp(1.9rem, 4vw, 2.8rem);
  font-weight: 500;
  line-height: 1.1;
  color: rgba(255, 250, 246, 0.96);
}

.emo-diary-subtitle {
  margin: 12px 0 0;
  max-width: 30rem;
  font-family: 'LXGW WenKai', 'STKaiti', serif;
  font-size: 0.98rem;
  line-height: 1.8;
  color: rgba(246, 247, 249, 0.62);
}

.emo-diary-body {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.emo-diary-section {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.emo-diary-line {
  margin: 0;
  font-family: 'LXGW WenKai', 'STKaiti', serif;
  font-size: clamp(1.08rem, 1.5vw, 1.22rem);
  line-height: 1.95;
  color: rgba(255, 250, 246, 0.9);
  text-shadow: 0 1px 10px rgba(0, 0, 0, 0.18);
}

.hero-title {
  display: inline-block;
  font-family: 'Shippori Mincho', 'Times New Roman', serif;
  font-size: clamp(2.2rem, 4.8vw, 3.6rem);
  margin: 0;
  font-weight: 600;
  line-height: 1.05;
  background-image: linear-gradient(120deg,
      rgba(154, 220, 210, 0.55),
      rgba(246, 247, 249, 0.9),
      rgba(184, 166, 255, 0.55));
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

  .emo-diary {
    margin-inline: auto;
    text-align: left;
  }
}

@media (max-width: 600px) {
  .hero-card :deep(.ant-card-body) {
    padding-inline: 22px;
    padding-block: calc(22px + 1cm);
  }

  .hero-avatar {
    border-radius: 26px;
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

  .emo-diary {
    padding: 24px 20px;
  }

  .emo-diary-head {
    margin-bottom: 22px;
  }

  .emo-diary-body {
    gap: 20px;
  }

  .emo-diary-section {
    gap: 4px;
  }

  .emo-diary-line {
    font-size: 1.02rem;
    line-height: 1.85;
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

</style>
