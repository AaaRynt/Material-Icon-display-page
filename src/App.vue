<template>
  <div class="flex-center min-h-screen font-mono duration-1000" :style="{ backgroundColor: color, color: fontColor }">
    <small class="vue fixed top-1 right-1 font-extrabold">{{ selectionLabel }} · {{ currentIconCount }}</small>
    <small v-if="iconStatus" class="fixed top-7 right-1 rounded-full px-3 py-0.5 text-xs shadow-lg/50 backdrop-blur-xs">
      {{ iconStatus }}
    </small>

    <section v-if="isEmptySelection" class="flex min-h-32 items-center justify-center text-sm opacity-70">No icons selected</section>
    <main v-else class="grid" :style="{ width: range + '%', gridTemplateColumns: `repeat(${num}, minmax(0, 1fr))` }" ref="mainRef">
      <img v-for="icon in currentIcons" :src="getIconUrl(icon)" :alt="icon.name" :key="getIconKey(icon)" @load="handleIconLoad(icon)" @error="handleIconError(icon)" @click="remove(icon)" />
    </main>

    <nav class="fixed right-4 bottom-4 left-4 flex flex-wrap items-center justify-center gap-2 rounded-lg px-4 py-2 shadow-lg/50 backdrop-blur-xs">
      <input type="color" v-model="color" class="rounded-4xl" />
      <div class="grid max-h-28 w-[44rem] max-w-[calc(100vw-14rem)] grid-cols-2 gap-x-3 gap-y-1 overflow-y-auto rounded-sm px-2 py-1 text-xs outline-1 sm:grid-cols-3">
        <label v-for="group in iconGroups" :key="group.id" class="flex min-w-0 cursor-pointer items-center gap-1">
          <input type="checkbox" v-model="selectedGroupIds" :value="group.id" class="size-3 accent-blue-400" />
          <span class="truncate">{{ group.label }}</span>
          <span class="opacity-60">{{ group.icons.length }}</span>
        </label>
      </div>
      <button
        class="w-6 duration-200 hover:text-blue-400 disabled:cursor-not-allowed disabled:opacity-40"
        :disabled="isDownloading || isEmptySelection"
        :title="isDownloading ? 'Downloading' : 'Download'"
        @click="download"
      >
        <RiFileDownloadLine />
      </button>
      <button class="w-6 duration-200 hover:text-blue-400 disabled:cursor-not-allowed disabled:opacity-40" :disabled="isEmptySelection" title="Shuffle" @click="shuffleSelected">
        <RiShuffleLine />
      </button>
      <input type="number" min="1" max="200" v-model.number="num" class="w-20 rounded-sm pl-1 outline-1" placeholder="col" />
      <input type="range" list="list" min="0" max="100" v-model.number="range" class="cursor-ew-resize" />
      <datalist id="list">
        <option :value="value * 25" v-for="value in 3" :key="value"></option>
      </datalist>
    </nav>
  </div>
</template>

<script setup lang="ts">
import { computed, reactive, ref, watch } from 'vue'
import svgRaw from '@/svg.json'
import { RiFileDownloadLine, RiShuffleLine } from '@/svg/index'
import { shuffle } from '@/utils/Fisher–Yates'

const MATERIAL_ICON_BASE_URL = 'https://raw.githubusercontent.com/material-extensions/vscode-material-icon-theme/1c8d1f38c8f5ab044633fce63a169291eed1ea83/icons/'
const VSCODE_ICONS_BASE_URL = 'https://raw.githubusercontent.com/vscode-icons/vscode-icons/4cb5fbf2725a9df2322b1dddbd7aca1083cfef34/icons/'

type TSvgAll = typeof svgRaw
type TSvgCategory = keyof TSvgAll
type TIconSource = 'material' | 'vscode'
type TIconGroupId =
  | 'material-all'
  | 'material-files'
  | 'material-light-files'
  | 'material-folders'
  | 'material-open-folders'
  | 'vscode-defaults'
  | 'vscode-files'
  | 'vscode-folders'
  | 'vscode-open-folders'
type TIconItem = {
  name: string
  source: TIconSource
}
type TIconGroup = {
  id: TIconGroupId
  label: string
  icons: TIconItem[]
}
type TLoadedIcon = {
  icon: TIconItem
  image: HTMLImageElement
}
type TMaterialGroupConfig = {
  id: TIconGroupId
  label: string
  category: TSvgCategory
}

const iconBaseUrlMap: Record<TIconSource, string> = {
  material: MATERIAL_ICON_BASE_URL,
  vscode: VSCODE_ICONS_BASE_URL,
}
const materialGroupConfigs: TMaterialGroupConfig[] = [
  { id: 'material-all', label: 'Material all', category: 'Material-Icon' },
  { id: 'material-files', label: 'Material files', category: 'Files' },
  { id: 'material-light-files', label: 'Material light', category: 'Files+Light' },
  { id: 'material-folders', label: 'Material folders', category: 'Folders Close' },
  { id: 'material-open-folders', label: 'Material open', category: 'Folders Open' },
]
const materialCategories: TSvgCategory[] = ['Files', 'Files+Light', 'Folders Close', 'Folders Open', 'Folders All', 'Files &..Open', 'Material-Icon']
const svgAll = reactive(Object.fromEntries(Object.entries(svgRaw).map(([category, icons]) => [category, [...icons]])) as TSvgAll)
const color = ref<string>('#282c34')
const selectedGroupIds = ref<TIconGroupId[]>(['material-all'])
const shuffledIconKeys = ref<string[]>([])
const loadedIconKeys = ref<string[]>([])
const failedIconKeys = ref<string[]>([])
const num = ref<number>(71)
const range = ref<number>(90)
const isDownloading = ref<boolean>(false)
const downloadError = ref<string>('')

const createIconItems = (names: string[], source: TIconSource): TIconItem[] => names.map((name) => ({ name, source }))
const isVscodeFileIcon = (name: string): boolean => name.startsWith('file_type_')
const isVscodeFolderIcon = (name: string): boolean => name.startsWith('folder_type_') && !name.endsWith('_opened.svg')
const isVscodeOpenFolderIcon = (name: string): boolean => name.startsWith('folder_type_') && name.endsWith('_opened.svg')
const isVscodeDefaultIcon = (name: string): boolean => name.startsWith('default_')
const addUniqueValue = (values: string[], value: string): string[] => (values.includes(value) ? values : [...values, value])

const iconGroups = computed((): TIconGroup[] => {
  const vscodeIcons = svgAll['vscode-icons']

  return [
    ...materialGroupConfigs.map((group) => ({
      id: group.id,
      label: group.label,
      icons: createIconItems(svgAll[group.category], 'material'),
    })),
    { id: 'vscode-defaults', label: 'VS Code default', icons: createIconItems(vscodeIcons.filter(isVscodeDefaultIcon), 'vscode') },
    { id: 'vscode-files', label: 'VS Code files', icons: createIconItems(vscodeIcons.filter(isVscodeFileIcon), 'vscode') },
    { id: 'vscode-folders', label: 'VS Code folders', icons: createIconItems(vscodeIcons.filter(isVscodeFolderIcon), 'vscode') },
    { id: 'vscode-open-folders', label: 'VS Code open', icons: createIconItems(vscodeIcons.filter(isVscodeOpenFolderIcon), 'vscode') },
  ]
})
const selectedIconGroups = computed((): TIconGroup[] => iconGroups.value.filter((group) => selectedGroupIds.value.includes(group.id)))
const baseIcons = computed((): TIconItem[] => {
  const iconMap = new Map<string, TIconItem>()

  selectedIconGroups.value.forEach((group) => {
    group.icons.forEach((icon) => {
      iconMap.set(getIconKey(icon), icon)
    })
  })

  return [...iconMap.values()]
})
const currentIcons = computed((): TIconItem[] => {
  if (shuffledIconKeys.value.length === 0) return baseIcons.value

  const iconMap = new Map(baseIcons.value.map((icon) => [getIconKey(icon), icon]))
  const orderedIcons = shuffledIconKeys.value.flatMap((key) => {
    const icon = iconMap.get(key)
    if (!icon) return []
    iconMap.delete(key)
    return [icon]
  })

  return [...orderedIcons, ...iconMap.values()]
})
const currentIconKeys = computed((): string[] => currentIcons.value.map(getIconKey))
const currentIconKeySet = computed((): Set<string> => new Set(currentIconKeys.value))
const currentIconCount = computed((): number => currentIcons.value.length)
const selectedGroupCount = computed((): number => selectedIconGroups.value.length)
const isEmptySelection = computed((): boolean => currentIconCount.value === 0)
const loadedIconCount = computed((): number => loadedIconKeys.value.filter((key) => currentIconKeySet.value.has(key)).length)
const erroredIconCount = computed((): number => failedIconKeys.value.filter((key) => currentIconKeySet.value.has(key)).length)
const pendingIconCount = computed((): number => Math.max(currentIconCount.value - loadedIconCount.value - erroredIconCount.value, 0))
const selectionLabel = computed((): string => {
  if (selectedGroupCount.value === 0) return 'No category'
  if (selectedGroupCount.value === 1) return selectedIconGroups.value[0]!.label
  return `${selectedGroupCount.value} categories`
})
const iconStatus = computed((): string => {
  const status: string[] = []

  if (!isEmptySelection.value && pendingIconCount.value > 0) status.push(`Loading ${pendingIconCount.value}`)
  if (erroredIconCount.value > 0) status.push(`Error ${erroredIconCount.value}`)
  if (downloadError.value) status.push(downloadError.value)

  return status.join(' · ')
})
const fontColor = computed((): string => {
  const invert = (hex: string) => (255 - parseInt(hex, 16)).toString(16).padStart(2, '0')
  const hex = color.value.slice(1)
  return `#${invert(hex.slice(0, 2))}${invert(hex.slice(2, 4))}${invert(hex.slice(4, 6))}`
})

function getIconKey(icon: TIconItem): string {
  return `${icon.source}:${icon.name}`
}
const getIconUrl = (icon: TIconItem): string => `${iconBaseUrlMap[icon.source]}${icon.name}`
const resetIconState = (): void => {
  loadedIconKeys.value = []
  failedIconKeys.value = []
  downloadError.value = ''
}
const markFailedIconKeys = (keys: string[]): void => {
  keys.forEach((key) => {
    failedIconKeys.value = addUniqueValue(failedIconKeys.value, key)
  })
}
const handleIconLoad = (icon: TIconItem): void => {
  const key = getIconKey(icon)
  loadedIconKeys.value = addUniqueValue(loadedIconKeys.value, key)
  failedIconKeys.value = failedIconKeys.value.filter((failedKey) => failedKey !== key)
}
const handleIconError = (icon: TIconItem): void => {
  markFailedIconKeys([getIconKey(icon)])
}
const remove = (icon: TIconItem): void => {
  if (icon.source === 'vscode') {
    svgAll['vscode-icons'] = svgAll['vscode-icons'].filter((name) => name !== icon.name)
    return
  }

  materialCategories.forEach((category) => {
    svgAll[category] = svgAll[category].filter((name) => name !== icon.name)
  })
}
const shuffleSelected = (): void => {
  shuffledIconKeys.value = shuffle(currentIconKeys.value)
}
const mainRef = ref<HTMLElement | null>(null)

const loadIconImage = (icon: TIconItem): Promise<TLoadedIcon> =>
  new Promise<TLoadedIcon>((resolve, reject) => {
    const img = new Image()
    img.crossOrigin = 'anonymous'
    img.onload = () => resolve({ icon, image: img })
    img.onerror = () => reject(new Error(getIconKey(icon)))
    img.alt = icon.name
    img.src = getIconUrl(icon)
  })

const createDownloadName = (): string =>
  selectionLabel.value
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '') || 'icons'
const download = async (): Promise<void> => {
  if (!mainRef.value || isDownloading.value) return
  downloadError.value = ''

  if (isEmptySelection.value) {
    downloadError.value = 'Empty'
    return
  }

  isDownloading.value = true

  try {
    const icons = [...currentIcons.value]
    const imageResults = await Promise.allSettled(icons.map(loadIconImage))
    const images = imageResults.filter((result): result is PromiseFulfilledResult<TLoadedIcon> => result.status === 'fulfilled').map((result) => result.value)
    const failedKeys = imageResults.flatMap((result, index) => (result.status === 'rejected' ? [getIconKey(icons[index]!)] : []))

    if (failedKeys.length > 0) {
      markFailedIconKeys(failedKeys)
      downloadError.value = `Skipped ${failedKeys.length}`
    }

    if (images.length === 0) {
      downloadError.value = 'No images'
      return
    }

    const cols = Math.max(num.value, 1)
    const size = 40
    const gap = 0
    const rows = Math.ceil(images.length / cols)

    const scale = 4
    const canvas = document.createElement('canvas')
    canvas.width = (cols * size + (cols - 1) * gap) * scale
    canvas.height = (rows * size + (rows - 1) * gap) * scale

    const ctx = canvas.getContext('2d')
    if (!ctx) return

    ctx.scale(scale, scale)
    ctx.fillStyle = color.value
    ctx.fillRect(0, 0, canvas.width / scale, canvas.height / scale)

    images.forEach(({ image }, i) => {
      const x = (i % cols) * (size + gap)
      const y = Math.floor(i / cols) * (size + gap)
      ctx.drawImage(image, x, y, size, size)
    })

    const dataUrl = canvas.toDataURL('image/png')
    const link = document.createElement('a')
    link.href = dataUrl
    link.download = `aaaRynt-${createDownloadName()}-${new Date().toISOString().replaceAll(':', '-')}-${scale}x.png`
    link.click()
  } finally {
    isDownloading.value = false
  }
}

watch(
  () => selectedGroupIds.value,
  () => {
    shuffledIconKeys.value = []
  },
  { deep: true },
)
watch(
  () => currentIconKeys.value,
  () => resetIconState(),
  { immediate: true },
)
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
