<!-- NeonSignature.vue -->
<template>
  <div class="neon">
    <h2 class="title">霓虹手写签名板</h2>

    <!-- 霓虹脉冲画布 -->
    <div class="canvas-wrap">
      <canvas
        ref="canvas"
        :width="canvasW"
        :height="canvasH"
        @pointerdown="start"
        @pointermove="move"
        @pointerup="stop"
      />
    </div>

    <!-- 毛玻璃工具栏 -->
    <div class="toolbar">
      <button @click="clear">清空</button>
      <button @click="exportCanvas">导出 PNG</button>
      <button @click="exportWebp">导出 WebP</button>
    </div>

    <!-- 实时预览 -->
    <div class="preview">
      <p>实时预览</p>
      <img id="previewImg" :src="previewUrl" v-if="previewUrl" />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

/* -------- 基础配置 -------- */
const canvasW = 800
const canvasH = 400
const canvas  = ref(null)
const ctx     = ref(null)
const previewUrl = ref('')

/* -------- 签名状态 -------- */
let drawing = false
let lastX = 0, lastY = 0, lastT = 0

/* -------- 生命周期 -------- */
onMounted(() => {
  const c = canvas.value
  c.width = canvasW
  c.height = canvasH
  ctx.value = c.getContext('2d')
  ctx.value.lineCap = 'round'
  ctx.value.lineJoin = 'round'
  drawStarBackground() // 星点背景
})

onUnmounted(() => particles.length = 0)

/* -------- 星点背景 -------- */
function drawStarBackground() {
  for (let i = 0; i < 120; i++) {
    ctx.value.fillStyle = 'rgba(255,255,255,' + Math.random() * 0.8 + ')'
    ctx.value.beginPath()
    ctx.value.arc(Math.random() * canvasW, Math.random() * canvasH, Math.random() * 1.2, 0, Math.PI * 2)
    ctx.value.fill()
  }
}

/* -------- 彩虹渐变 -------- */
function rainbowGradient(x1, y1, x2, y2) {
  const g = ctx.value.createLinearGradient(x1, y1, x2, y2)
  const colors = ['#ff0080', '#ff8000', '#80ff00', '#00ff80', '#0080ff', '#8000ff']
  colors.forEach((c, i) => g.addColorStop(i / (colors.length - 1), c))
  return g
}

/* -------- 压感粗细 -------- */
function calcLineWidth(deltaT, distance) {
  const speed = distance / deltaT
  return Math.max(1, Math.min(20, 20 - speed * 0.8))
}

/* -------- 绘制事件 -------- */
function start(e) {
  drawing = true
  lastX = e.offsetX
  lastY = e.offsetY
  lastT = performance.now()
}

function move(e) {
  if (!drawing) return
  const x = e.offsetX, y = e.offsetY, t = performance.now()
  const dx = x - lastX, dy = y - lastY, distance = Math.hypot(dx, dy)
  const deltaT = t - lastT || 1
  const lineWidth = calcLineWidth(deltaT, distance)

  ctx.value.save()
  ctx.value.lineWidth = lineWidth
  ctx.value.strokeStyle = rainbowGradient(lastX, lastY, x, y)
  ctx.value.beginPath()
  ctx.value.moveTo(lastX, lastY)
  ctx.value.lineTo(x, y)
  ctx.value.stroke()
  ctx.value.restore()

  lastX = x; lastY = y; lastT = t
}

function stop(e) {
  drawing = false
  particleExplode(e.offsetX, e.offsetY)
  updatePreview() // 实时预览
}

/* -------- 粒子爆炸 -------- */
const particles = []
class Particle {
  constructor(x, y) {
    this.x = x; this.y = y
    this.vx = (Math.random() - 0.5) * 4
    this.vy = (Math.random() - 0.5) * 4
    this.life = 1
    this.color = `hsl(${Math.random() * 360}, 100%, 50%)`
  }
  update() {
    this.x += this.vx; this.y += this.vy
    this.life -= 0.015
  }
  draw() {
    ctx.value.save()
    ctx.value.globalAlpha = this.life
    ctx.value.fillStyle = this.color
    ctx.value.beginPath()
    ctx.value.arc(this.x, this.y, 2, 0, Math.PI * 2)
    ctx.value.fill()
    ctx.value.restore()
  }
}
function particleExplode(x, y) {
  for (let i = 0; i < 30; i++) particles.push(new Particle(x, y))
  animateParticles()
}
function animateParticles() {
  particles.forEach((p, i) => {
    p.update(); p.draw()
    if (p.life <= 0) particles.splice(i, 1)
  })
  if (particles.length) requestAnimationFrame(animateParticles)
}

/* -------- 实时预览 -------- */
function updatePreview() {
  canvas.value.toBlob(blob => {
    previewUrl.value = URL.createObjectURL(blob)
  })
}

/* -------- 导出功能 -------- */
function downloadBlob(blob, fileName) {
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = fileName
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  setTimeout(() => URL.revokeObjectURL(url), 3000)
}
function exportCanvas() {
  canvas.value.toBlob(blob => downloadBlob(blob, 'signature.png'), 'image/png')
}
function exportWebp() {
  canvas.value.toBlob(blob => downloadBlob(blob, 'signature.webp'), 'image/webp', 0.8)
}

/* -------- 清空 -------- */
function clear() {
  ctx.value.clearRect(0, 0, canvasW, canvasH)
  particles.length = 0
  drawStarBackground()
  previewUrl.value = ''
}
</script>

<style scoped>
.neon {
  min-height: 100vh;
  background: radial-gradient(ellipse at center, #111 0%, #000 100%);
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}
.title {
  color: #00ffff;
  text-shadow: 0 0 10px #00ffff;
  margin-bottom: 20px;
}
.canvas-wrap {
  position: relative;
  border-radius: 16px;
  padding: 4px;
  background: linear-gradient(45deg, #00ffff, #ff00ff, #00ffff);
  background-size: 400% 400%;
  animation: pulse 3s ease infinite;
  box-shadow: 0 0 20px #00ffff, 0 0 40px #ff00ff;
}
@keyframes pulse {
  0%   { background-position: 0% 50%; }
  50%  { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}
canvas {
  display: block;
  border-radius: 12px;
  background: #000;
  touch-action: none;
}
.toolbar {
  margin-top: 20px;
  display: flex;
  gap: 12px;
  padding: 8px 16px;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.05);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.1);
}
button {
  position: relative;
  padding: 8px 20px;
  border: 0;
  border-radius: 8px;
  background: rgba(0, 255, 255, 0.1);
  color: #00ffff;
  font-size: 14px;
  cursor: pointer;
  overflow: hidden;
  transition: all 0.3s;
}
button::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  border-radius: 50%;
  background: rgba(0, 255, 255, 0.5);
  transform: translate(-50%, -50%);
  transition: width 0.4s, height 0.4s;
}
button:hover::before {
  width: 300px;
  height: 300px;
}
button:hover {
  color: #000;
  box-shadow: 0 0 15px #00ffff;
}
.preview {
  margin-top: 20px;
  text-align: center;
  color: #0f0;
  font-size: 13px;
}
.preview img {
  max-width: 150px;
  border-radius: 8px;
  box-shadow: 0 0 10px #0f0;
}
</style>