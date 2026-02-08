<template>
  <div class="flex-center min-h-screen font-mono duration-1000" :style="{ backgroundColor: color, color: fontColor }">
    <small class="vue absolute top-1 right-1 font-extrabold">{{ svgAll[selected].length }}</small>

    <main class="grid" :style="{ width: range + '%', gridTemplateColumns: `repeat(${num}, minmax(0, 1fr))` }" ref="mainRef">
      <img
        v-for="value in svgAll[selected]"
        :src="`https://raw.githubusercontent.com/AaaRynt/vscode-material-icon-theme/e6bc94766159101048d2503314357500eb4d2659/icons/${value}`"
        :alt="value"
        :key="value"
        @click="remove(value)"
      />
    </main>

    <nav class="fixed bottom-4 flex items-center gap-2 rounded-full px-4 py-0.5 shadow-lg/50 backdrop-blur-xs">
      <input type="color" v-model="color" class="rounded-4xl" />
      <select v-model="selected" class="w-28 rounded-sm text-center text-xs outline-1">
        <option disabled value="">Please select one</option>
        <option v-for="value in Object.keys(svgAll)" :key="value" :value="value">{{ value }}</option>
      </select>
      <button class="w-6 duration-200 hover:text-blue-400" @click="download">
        <RiFileDownloadLine />
      </button>
      <button class="w-6 duration-200 hover:text-blue-400" @click="svgAll[selected] = shuffle(svgAll[selected])">
        <RiShuffleLine />
      </button>
      <input type="number" min="1" max="200" v-model="num" class="w-20 rounded-sm pl-1 outline-1" placeholder="col" />
      <input type="range" list="list" min="0" max="100" v-model="range" class="cursor-ew-resize" />
      <datalist id="list">
        <option :value="value * 25" v-for="value in 3" :key="value"></option>
      </datalist>
    </nav>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, reactive } from 'vue'
import svgRaw from '@/svg.json'
import { RiFileDownloadLine, RiShuffleLine } from './svg/index'
import { shuffle } from './utils/Fisher–Yates'

const svgAll = reactive(svgRaw)
const color = ref<string>('#282c34')
const selected = ref<keyof typeof svgAll>('Files')
const num = ref<number>(30)
const range = ref<number>(80)
const fontColor = computed((): string => {
  const hex = color.value.slice(1)
  const r = (255 - parseInt(hex.substring(0, 2), 16)).toString(16).padStart(2, '0')
  const g = (255 - parseInt(hex.substring(2, 4), 16)).toString(16).padStart(2, '0')
  const b = (255 - parseInt(hex.substring(4, 6), 16)).toString(16).padStart(2, '0')

  return `#${r}${g}${b}`
})

// GENERATE BY GPT-5.2
const remove = (value: string): void => {
  svgAll[selected.value] = svgAll[selected.value].filter((v) => v !== value)
}
const mainRef = ref<HTMLElement | null>(null)

const download = async () => {
  if (!mainRef.value) return

  const images = await Promise.all(
    svgAll[selected.value].map(
      (name) =>
        new Promise<HTMLImageElement>((resolve, reject) => {
          const img = new Image()
          img.crossOrigin = 'anonymous' // 跨域处理
          img.onload = () => resolve(img)
          img.onerror = (e) => reject(e)
          img.src = `https://raw.githubusercontent.com/AaaRynt/vscode-material-icon-theme/e6bc94766159101048d2503314357500eb4d2659/icons/${name}`
        }),
    ),
  )

  const cols = num.value
  const size = 40 // 页面显示大小
  const gap = 0
  const rows = Math.ceil(images.length / cols)

  const scale = 4 // 高清倍数，Retina 级别
  const canvas = document.createElement('canvas')
  canvas.width = (cols * size + (cols - 1) * gap) * scale
  canvas.height = (rows * size + (rows - 1) * gap) * scale

  const ctx = canvas.getContext('2d')
  if (!ctx) return

  ctx.scale(scale, scale) // 放大上下文，但绘制时还是用原 size
  ctx.fillStyle = color.value
  ctx.fillRect(0, 0, canvas.width / scale, canvas.height / scale)

  images.forEach((img, i) => {
    const x = (i % cols) * (size + gap)
    const y = Math.floor(i / cols) * (size + gap)
    ctx.drawImage(img, x, y, size, size)
  })

  const dataUrl = canvas.toDataURL('image/png')
  const link = document.createElement('a')
  link.href = dataUrl
  link.download = `aaaRynt-${new Date()}-${scale}x.png`
  link.click()
}
</script>

<style scoped>
* {
  transition-property: color, background-color, border-color, outline-color, text-decoration-color, fill, stroke, --tw-gradient-from, --tw-gradient-via, --tw-gradient-to;
  transition-timing-function: var(--tw-ease, var(--default-transition-timing-function) /* cubic-bezier(0.4, 0, 0.2, 1) */);
}

.vue {
  background: linear-gradient(315deg, #42d392 25%, #647eff);
  background: -webkit-linear-gradient(315deg, #42d392 25%, #647eff);
  background-clip: text;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
</style>
