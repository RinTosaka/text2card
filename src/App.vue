
<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import html2canvas from 'html2canvas'
import JSZip from 'jszip'
import { marked } from 'marked'

const MANUAL_BREAK_MARK = '[[page]]'
const INLINE_STYLE_MARK = '{{文字|color=#ff4d4f;size=28;weight=700}}'

const sizePresets = [
  { id: 'square', name: '正方形', ratioLabel: '1:1', width: 1080, height: 1080 },
  { id: 'xhs', name: '小红书', ratioLabel: '3:4', width: 1080, height: 1440 },
  { id: 'xhs-long', name: '小红书长文', ratioLabel: '3:5', width: 1080, height: 1800 },
  { id: 'instagram', name: 'Instagram', ratioLabel: '4:3', width: 1080, height: 810 },
  { id: 'pinterest', name: 'Pinterest', ratioLabel: '7:5', width: 1400, height: 1000 },
  { id: 'douyin', name: '抖音', ratioLabel: '9:16', width: 1080, height: 1920 },
  { id: 'youtube', name: 'Youtube', ratioLabel: '16:9', width: 1920, height: 1080 },
  { id: 'a4', name: 'A4纸', ratioLabel: '210:297', width: 2100, height: 2970 },
  { id: 'custom', name: '自定义', ratioLabel: '-', width: 1080, height: 1350 },
]

const cardTemplates = [
  { id: 'clean', name: '简约' },
  { id: 'handwriting', name: '手写' },
  { id: 'code', name: '代码' },
  { id: 'sticky', name: '便签' },
  { id: 'memo', name: '备忘录' },
]

const themePresets = [
  {
    id: 'sunrise',
    name: '晨光',
    background: 'linear-gradient(145deg, #ffe3ce 0%, #ffc5b0 45%, #f38e77 100%)',
    textColor: '#3b2318',
  },
  {
    id: 'forest',
    name: '山林',
    background: 'linear-gradient(145deg, #e8f8f0 0%, #bfe3ce 42%, #7bb29a 100%)',
    textColor: '#102e24',
  },
  {
    id: 'ink',
    name: '墨夜',
    background: 'linear-gradient(145deg, #252f45 0%, #3e4e75 56%, #6a70b8 100%)',
    textColor: '#edf2ff',
  },
  {
    id: 'paper',
    name: '纸感',
    background: 'linear-gradient(145deg, #fbf7ef 0%, #efe4d1 100%)',
    textColor: '#2b2520',
  },
  {
    id: 'ocean',
    name: '海盐',
    background: 'linear-gradient(145deg, #d6f0ff 0%, #9fd4f2 45%, #5ea6d8 100%)',
    textColor: '#113247',
  },
  {
    id: 'sakura',
    name: '樱雾',
    background: 'linear-gradient(145deg, #ffe7f2 0%, #fbcbe3 48%, #f2a9cb 100%)',
    textColor: '#4b2238',
  },
  {
    id: 'aurora',
    name: '极光',
    background: 'linear-gradient(145deg, #dff4ff 0%, #cdebd8 46%, #8acfd5 100%)',
    textColor: '#15353a',
  },
  {
    id: 'lava',
    name: '熔岩',
    background: 'linear-gradient(145deg, #ffd7c2 0%, #ff9f84 42%, #d8594f 100%)',
    textColor: '#3d1814',
  },
  {
    id: 'mocha',
    name: '摩卡',
    background: 'linear-gradient(145deg, #f2e6db 0%, #d4bca5 50%, #a5866f 100%)',
    textColor: '#2f2219',
  },
  {
    id: 'mint-soda',
    name: '薄荷汽水',
    background: 'linear-gradient(145deg, #e8fff7 0%, #bcefdc 48%, #79d1b4 100%)',
    textColor: '#10362e',
  },
  {
    id: 'dusk',
    name: '暮色',
    background: 'linear-gradient(145deg, #f7d9d6 0%, #caa9d9 48%, #7f79bf 100%)',
    textColor: '#261f45',
  },
  {
    id: 'royal',
    name: '皇室蓝',
    background: 'linear-gradient(145deg, #2f3f79 0%, #4c5a95 48%, #8894c4 100%)',
    textColor: '#eef3ff',
  },
  {
    id: 'lemon',
    name: '柠光',
    background: 'linear-gradient(145deg, #fff7c9 0%, #ffe98d 46%, #f5c84d 100%)',
    textColor: '#43330f',
  },
  {
    id: 'reef',
    name: '珊瑚礁',
    background: 'linear-gradient(145deg, #ffd8cc 0%, #ffbca6 46%, #ff8f89 100%)',
    textColor: '#4a1d1e',
  },
  {
    id: 'glacier',
    name: '冰川',
    background: 'linear-gradient(145deg, #ecf5ff 0%, #d3e5f8 45%, #a7c8ea 100%)',
    textColor: '#1a3653',
  },
  {
    id: 'vintage',
    name: '复古报刊',
    background: 'linear-gradient(145deg, #f8f0dc 0%, #e4d2a9 48%, #c5a777 100%)',
    textColor: '#3f2f1a',
  },
]

const builtInFontOptions = [
  { id: 'system-default', name: '系统默认', value: "'PingFang SC','Microsoft YaHei','Helvetica Neue',Arial,sans-serif", category: 'zh' },
  { id: 'sans', name: '思源黑体 / Noto Sans SC', value: "'Noto Sans SC','Source Han Sans SC','PingFang SC','Microsoft YaHei',sans-serif", category: 'zh' },
  { id: 'serif', name: '思源宋体 / Noto Serif SC', value: "'Noto Serif SC','Source Han Serif SC','Songti SC','STSong',serif", category: 'zh' },
  { id: 'mono', name: 'Consolas (等宽)', value: "'Consolas','Cascadia Mono','Courier New',monospace", category: 'en' },
]

const ndLocalFontDefinitions = [
  { id: 'nd-zh-alibaba-puhui', name: '阿里巴巴惠普体', family: 'Alibaba-PuHuiTi-Regular', category: 'zh' },
  { id: 'nd-zh-misans', name: '小米-Regular', family: 'MiSans-Regular', category: 'zh' },
  { id: 'nd-zh-lxgw-wenkai-light', name: '霞鹜文楷-light', family: 'LXGW WenKai Light', category: 'zh' },
  { id: 'nd-zh-sourcehan-serif-semibold', name: '思源宋体-SemiBold', family: 'SourceHanSerifCN_SemiBold', category: 'zh', fallback: 'serif' },
  { id: 'nd-zh-sourcehan-serif-bold', name: '思源宋体-Bold', family: 'SourceHanSerifCN_Bold', category: 'zh', fallback: 'serif' },
  { id: 'nd-zh-canger-yuyang', name: '仓耳渔阳体W03', family: 'CangErYuYangTiW03', category: 'zh' },
  { id: 'nd-zh-huiwen-mingchao', name: '汇文明朝体', family: 'Huiwen_mingchao', category: 'zh', fallback: 'serif' },
]

const ndLocalFontOptions = ndLocalFontDefinitions.map((item) => ({
  id: item.id,
  name: item.name,
  value: createFontStack(item.family, item.category, item.fallback),
  category: item.category,
}))

const fontCategoryOptions = [
  { id: 'all', label: '全部' },
  { id: 'zh', label: '中文字体' },
  { id: 'en', label: '英文字体' },
  { id: 'uploaded', label: '我上传的字体' },
]

const fontTargetOptions = [
  { key: 'icon', label: '图标' },
  { key: 'title', label: '标题' },
  { key: 'body', label: '正文' },
  { key: 'author', label: '署名' },
  { key: 'time', label: '时间' },
  { key: 'page', label: '页码' },
  { key: 'watermark', label: '水印' },
]

const iconQuickPick = ['✦', '✿', '☕', '📌', '📝', '💡', '🚀', '🤖']

const previewFrameRef = ref(null)
const measureContentRef = ref(null)
const contentTextareaRef = ref(null)

const isExporting = ref(false)
const previewCardWidth = ref(700)
const pagedContents = ref([''])
const bulkDownloadMode = ref('zip')
const selectedFontCategory = ref('all')
const uploadedFontOptions = ref([])
const uploadedFontObjectUrls = ref([])

const sizePresetId = ref('xhs')
const sizeControl = reactive({ width: 1080, height: 1080 })
const aspectRatioLocked = ref(true)
const lockedAspectRatio = ref(1)

const templateId = ref('clean')

const display = reactive({
  icon: true,
  title: true,
  author: true,
  time: true,
  page: true,
  watermark: true,
})

const cardData = reactive({
  icon: '✦',
  title: '文字卡片标题',
  content: '在左侧输入文案，快速生成用于分享的卡片内容。',
  author: '@Text2Card',
  time: formatNow(),
  watermark: 'TEXT2CARD',
})

const typography = reactive({
  align: 'left',
  lineHeight: 1.5,
  padding: 30,
  radius: 28,
})

const fontFamilyConfig = reactive({
  icon: 'sans',
  title: 'serif',
  body: 'serif',
  author: 'sans',
  time: 'sans',
  page: 'mono',
  watermark: 'sans',
})

const fontSizeConfig = reactive({
  icon: 28,
  title: 34,
  body: 24,
  author: 16,
  time: 15,
  page: 15,
  watermark: 20,
})

const themeMode = ref('preset')
const presetThemeId = ref('sunrise')
const usePresetTextColor = ref(true)
const customTextColor = ref('#1d2430')
const solidColor = ref('#f3eee4')
const gradientStart = ref('#f7d4c3')
const gradientEnd = ref('#f1a58f')
const gradientAngle = ref(140)
const imageUrl = ref('')
const imageOverlay = ref(0.2)
const imageOverlayColor = ref('#000000')
const watermarkOpacity = ref(0.12)

const shadow = reactive({
  enabled: true,
  x: 0,
  y: 16,
  blur: 40,
  spread: 0,
  color: '#121a2a',
  opacity: 0.28,
})

watch(
  sizePresetId,
  (id) => {
    const preset = sizePresets.find((item) => item.id === id)
    if (!preset || id === 'custom') return
    sizeControl.width = preset.width
    sizeControl.height = preset.height
    lockedAspectRatio.value = preset.width / preset.height
  },
  { immediate: true },
)

const currentSize = computed(() => {
  const width = Math.max(320, Number(sizeControl.width) || 1080)
  const height = Math.max(320, Number(sizeControl.height) || 1350)
  const ratio = simplifyRatio(width, height)
  return {
    width,
    height,
    ratio,
  }
})

const currentTheme = computed(() => themePresets.find((item) => item.id === presetThemeId.value) || themePresets[0])
const totalPages = computed(() => Math.max(1, pagedContents.value.length))
const allFontOptions = computed(() => mergeAndDeduplicateFontOptions([...builtInFontOptions, ...ndLocalFontOptions, ...uploadedFontOptions.value]))
const filteredFontOptions = computed(() => {
  if (selectedFontCategory.value === 'all') {
    return allFontOptions.value
  }
  return allFontOptions.value.filter((item) => item.category === selectedFontCategory.value)
})
const renderedTitleHtml = computed(() => renderMarkdownInline(cardData.title, '未命名标题'))
const pagedContentHtml = computed(() => pagedContents.value.map((content) => renderMarkdownBody(content, '请在左侧输入正文内容。')))

const normalizedPlainText = computed(() => normalizeBreakMarks(cardData.content).replace(/\f/g, ''))
const charCount = computed(() => normalizedPlainText.value.replace(/\s/g, '').length)
const paragraphCount = computed(() => normalizedPlainText.value.split(/\n+/).filter((item) => item.trim().length > 0).length)
const statsText = computed(() => `字数 ${charCount.value} · 段落 ${paragraphCount.value} · 共 ${totalPages.value} 页`)

const resolvedTextColor = computed(() => {
  if (themeMode.value === 'preset' && usePresetTextColor.value) {
    return currentTheme.value.textColor
  }
  return normalizeCssColor(customTextColor.value, '#1d2430')
})

const resolvedBackground = computed(() => {
  if (themeMode.value === 'solid') {
    return normalizeCssColor(solidColor.value, '#f3eee4')
  }

  if (themeMode.value === 'gradient') {
    const start = normalizeCssColor(gradientStart.value, '#f7d4c3')
    const end = normalizeCssColor(gradientEnd.value, '#f1a58f')
    return `linear-gradient(${gradientAngle.value}deg, ${start} 0%, ${end} 100%)`
  }

  if (themeMode.value === 'image') {
    if (imageUrl.value) {
      const alpha = Math.max(0, Math.min(0.85, Number(imageOverlay.value) || 0))
      const overlay = rgbaFromColor(imageOverlayColor.value, alpha, '#000000')
      return `linear-gradient(${overlay}, ${overlay}), url("${imageUrl.value}") center / cover no-repeat`
    }
    return normalizeCssColor(solidColor.value, '#f3eee4')
  }

  return currentTheme.value.background
})

const resolvedShadow = computed(() => {
  if (!shadow.enabled) return 'none'
  const rgba = rgbaFromColor(shadow.color, shadow.opacity, '#121a2a')
  return `${shadow.x}px ${shadow.y}px ${shadow.blur}px ${shadow.spread}px ${rgba}`
})

const fontStyle = computed(() => ({
  icon: {
    fontFamily: getFontValue(fontFamilyConfig.icon),
    fontSize: `${fontSizeConfig.icon}px`,
  },
  title: {
    fontFamily: getFontValue(fontFamilyConfig.title),
    fontSize: `${fontSizeConfig.title}px`,
  },
  body: {
    fontFamily: getFontValue(fontFamilyConfig.body),
    fontSize: `${fontSizeConfig.body}px`,
    lineHeight: typography.lineHeight,
  },
  author: {
    fontFamily: getFontValue(fontFamilyConfig.author),
    fontSize: `${fontSizeConfig.author}px`,
  },
  time: {
    fontFamily: getFontValue(fontFamilyConfig.time),
    fontSize: `${fontSizeConfig.time}px`,
  },
  page: {
    fontFamily: getFontValue(fontFamilyConfig.page),
    fontSize: `${fontSizeConfig.page}px`,
  },
  watermark: {
    fontFamily: getFontValue(fontFamilyConfig.watermark),
    fontSize: `${fontSizeConfig.watermark}px`,
  },
}))

const cardStyle = computed(() => ({
  '--card-bg': resolvedBackground.value,
  '--card-text': resolvedTextColor.value,
  '--card-shadow': resolvedShadow.value,
  '--card-padding': `${typography.padding}px`,
  '--card-radius': `${typography.radius}px`,
  '--line-height': typography.lineHeight,
  '--watermark-opacity': watermarkOpacity.value,
  '--sticky-step': `${Math.max(20, Math.round(fontSizeConfig.body * typography.lineHeight))}px`,
  textAlign: typography.align,
  aspectRatio: `${currentSize.value.width} / ${currentSize.value.height}`,
}))

function simplifyRatio(width, height) {
  const gcd = (a, b) => (b === 0 ? a : gcd(b, a % b))
  const factor = gcd(width, height)
  return `${Math.round(width / factor)}:${Math.round(height / factor)}`
}

function normalizeFontOptionName(name) {
  return String(name || '').trim().toLowerCase()
}

function extractPrimaryFamilyFromValue(value) {
  const raw = String(value || '').trim()
  if (!raw) return ''
  const firstToken = raw.split(',')[0] || ''
  return firstToken.trim().replace(/^['"]|['"]$/g, '').toLowerCase()
}

function mergeAndDeduplicateFontOptions(options) {
  const deduped = []
  const seen = new Set()

  for (let i = 0; i < options.length; i += 1) {
    const item = options[i]
    const idKey = `id:${item.id}`
    if (seen.has(idKey)) continue

    const nameKey = `name:${item.category}:${normalizeFontOptionName(item.name)}`
    const familyKey = `family:${item.category}:${extractPrimaryFamilyFromValue(item.value)}`
    if (seen.has(nameKey) || seen.has(familyKey)) continue

    seen.add(idKey)
    if (normalizeFontOptionName(item.name)) {
      seen.add(nameKey)
    }
    if (extractPrimaryFamilyFromValue(item.value)) {
      seen.add(familyKey)
    }
    deduped.push(item)
  }

  return deduped
}

function createFontStack(family, category, fallback = 'sans') {
  const safeFamily = String(family || '').replace(/'/g, "\\'")
  if (fallback === 'mono') {
    return `'${safeFamily}','Consolas','Cascadia Mono','Courier New',monospace`
  }
  if (fallback === 'serif') {
    return `'${safeFamily}','Noto Serif SC','Source Han Serif SC','Songti SC',serif`
  }
  if (category === 'en') {
    return `'${safeFamily}','Inter','Segoe UI',sans-serif`
  }
  if (category === 'hand') {
    return `'${safeFamily}','KaiTi','STKaiti','Comic Sans MS',cursive`
  }
  return `'${safeFamily}','Noto Sans SC','Source Han Sans SC','PingFang SC','Microsoft YaHei',sans-serif`
}

function getFontById(fontId) {
  return allFontOptions.value.find((item) => item.id === fontId)
}

function getFontValue(fontId) {
  const found = getFontById(fontId)
  return found ? found.value : builtInFontOptions[0].value
}

function getSelectableFontOptions(targetKey) {
  const list = filteredFontOptions.value
  const currentId = fontFamilyConfig[targetKey]
  if (list.some((item) => item.id === currentId)) {
    return list
  }
  const current = getFontById(currentId)
  return current ? [current, ...list] : list
}

function sanitizeFontFamilyName(rawName) {
  return rawName
    .replace(/\.[^/.]+$/, '')
    .replace(/[^\w\u4e00-\u9fa5-]/g, '-')
    .replace(/-+/g, '-')
    .replace(/^-|-$/g, '')
}

async function onFontUpload(event) {
  if (typeof FontFace === 'undefined') {
    window.alert('当前浏览器不支持字体上传（FontFace API）。')
    return
  }

  const files = Array.from(event.target?.files || [])
  if (!files.length) return

  let loadedCount = 0
  for (let i = 0; i < files.length; i += 1) {
    const file = files[i]
    const objectUrl = URL.createObjectURL(file)
    const safeName = sanitizeFontFamilyName(file.name || `font-${Date.now()}`)
    const family = `user-font-${safeName}-${Date.now()}-${i}`
    try {
      const fontFace = new FontFace(family, `url("${objectUrl}")`)
      await fontFace.load()
      document.fonts.add(fontFace)
      uploadedFontObjectUrls.value.push(objectUrl)
      uploadedFontOptions.value.push({
        id: `uploaded-${Date.now()}-${i}`,
        name: `${safeName}（上传）`,
        value: `'${family}','Noto Sans SC','PingFang SC','Microsoft YaHei',sans-serif`,
        category: 'uploaded',
      })
      loadedCount += 1
    } catch (error) {
      URL.revokeObjectURL(objectUrl)
      console.error(error)
    }
  }

  event.target.value = ''
  if (loadedCount === 0) {
    window.alert('字体加载失败，请确认文件格式为 ttf/otf/woff/woff2。')
  } else {
    selectedFontCategory.value = 'uploaded'
  }
}

const markdownOptions = Object.freeze({
  gfm: true,
  breaks: true,
  async: false,
})

const styleTokenRegex = /\{\{([^|{}]+)\|([^{}]+)\}\}/g

function escapeHtml(value) {
  return String(value || '')
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;')
}

function parseBooleanFlag(value, defaultValue = false) {
  if (value == null || value === '') return defaultValue
  const text = String(value).trim().toLowerCase()
  if (['1', 'true', 'yes', 'on', 'y'].includes(text)) return true
  if (['0', 'false', 'no', 'off', 'n'].includes(text)) return false
  return defaultValue
}

function normalizeInlineFontSize(rawSize) {
  const text = String(rawSize || '').trim()
  if (!text) return ''

  const match = text.match(/^(\d+(?:\.\d+)?)(px|em|rem|%)?$/i)
  if (!match) return ''
  const numeric = Number.parseFloat(match[1])
  if (!Number.isFinite(numeric) || numeric <= 0) return ''

  const unit = (match[2] || 'px').toLowerCase()
  if (unit === 'px') {
    const clamped = Math.max(8, Math.min(180, numeric))
    return `${clamped}px`
  }
  return `${numeric}${unit}`
}

function parseStyleTokenSpec(specText) {
  const entries = String(specText || '')
    .split(';')
    .map((item) => item.trim())
    .filter(Boolean)

  const styles = []
  for (let i = 0; i < entries.length; i += 1) {
    const entry = entries[i]
    const parts = entry.split(/[:=]/)
    const key = String(parts[0] || '').trim().toLowerCase()
    const value = String(parts.slice(1).join(':') || '').trim()
    if (!key) continue

    if (key === 'color' || key === 'c') {
      const nextColor = normalizeCssColor(value, '')
      if (nextColor) styles.push(`color:${nextColor}`)
      continue
    }

    if (key === 'bg' || key === 'background') {
      const nextBg = normalizeCssColor(value, '')
      if (nextBg) styles.push(`background-color:${nextBg}`)
      continue
    }

    if (key === 'size' || key === 'font-size' || key === 'fs') {
      const nextSize = normalizeInlineFontSize(value)
      if (nextSize) styles.push(`font-size:${nextSize}`)
      continue
    }

    if (key === 'weight' || key === 'w') {
      if (/^(normal|bold|bolder|lighter|[1-9]00)$/i.test(value)) {
        styles.push(`font-weight:${value}`)
      }
      continue
    }

    if (key === 'italic' || key === 'i') {
      styles.push(`font-style:${parseBooleanFlag(value, true) ? 'italic' : 'normal'}`)
      continue
    }

    if (key === 'underline' || key === 'u') {
      styles.push(`text-decoration:${parseBooleanFlag(value, true) ? 'underline' : 'none'}`)
    }
  }

  return styles.join(';')
}

function compileMarkdownInput(rawText) {
  const placeholders = []
  const tokenized = String(rawText || '').replace(styleTokenRegex, (_, text, spec) => {
    const content = escapeHtml(String(text || '').trim())
    const inlineStyle = parseStyleTokenSpec(spec)
    const html = inlineStyle ? `<span class="md-inline-style" style="${inlineStyle}">${content}</span>` : content
    const key = `@@__STYLE_TOKEN_${placeholders.length}__@@`
    placeholders.push({ key, html })
    return key
  })

  let safeText = escapeHtml(tokenized)
  for (let i = 0; i < placeholders.length; i += 1) {
    const item = placeholders[i]
    safeText = safeText.replaceAll(item.key, item.html)
  }
  return safeText
}

function renderMarkdownInline(text, fallback = '') {
  const source = String(text || '').trim() ? String(text || '') : fallback
  const compiled = compileMarkdownInput(source)
  try {
    if (typeof marked.parseInline === 'function') {
      const html = marked.parseInline(compiled, markdownOptions)
      return typeof html === 'string' && html.trim() ? html : escapeHtml(fallback || '')
    }
    const html = marked.parse(compiled, markdownOptions)
    if (typeof html !== 'string') return escapeHtml(fallback || '')
    return html.replace(/^<p>/, '').replace(/<\/p>\s*$/, '')
  } catch (error) {
    console.error(error)
    return escapeHtml(source)
  }
}

function renderMarkdownBody(text, fallback = '') {
  const source = String(text || '').trim() ? String(text || '') : fallback
  const compiled = compileMarkdownInput(source)
  try {
    const html = marked.parse(compiled, markdownOptions)
    if (typeof html === 'string' && html.trim()) {
      return html
    }
  } catch (error) {
    console.error(error)
  }
  return `<p>${escapeHtml(source)}</p>`
}

function formatNow() {
  const now = new Date()
  const pad = (num) => String(num).padStart(2, '0')
  return `${now.getFullYear()}-${pad(now.getMonth() + 1)}-${pad(now.getDate())} ${pad(now.getHours())}:${pad(now.getMinutes())}`
}

function setCurrentTime() {
  cardData.time = formatNow()
}

function normalizeCssColor(color, fallback) {
  const value = String(color || '').trim()
  if (!value) return fallback
  if (typeof CSS !== 'undefined' && CSS.supports && CSS.supports('color', value)) {
    return value
  }
  return fallback
}

let colorCanvasCtx = null
function getColorCanvasCtx() {
  if (colorCanvasCtx) return colorCanvasCtx
  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')
  colorCanvasCtx = ctx
  return ctx
}

function parseColorToRgb(color, fallback = '#000000') {
  const ctx = getColorCanvasCtx()
  if (!ctx) return { r: 0, g: 0, b: 0 }

  const safeColor = normalizeCssColor(color, fallback)
  ctx.fillStyle = safeColor
  const parsed = ctx.fillStyle

  const hexMatch = parsed.match(/^#([0-9a-f]{6})$/i)
  if (hexMatch) {
    const hex = hexMatch[1]
    return {
      r: Number.parseInt(hex.slice(0, 2), 16),
      g: Number.parseInt(hex.slice(2, 4), 16),
      b: Number.parseInt(hex.slice(4, 6), 16),
    }
  }

  const rgbMatch = parsed.match(/^rgba?\((\d+),\s*(\d+),\s*(\d+)/i)
  if (rgbMatch) {
    return {
      r: Number.parseInt(rgbMatch[1], 10),
      g: Number.parseInt(rgbMatch[2], 10),
      b: Number.parseInt(rgbMatch[3], 10),
    }
  }

  return parseColorToRgb(fallback, '#000000')
}

function rgbToHex({ r, g, b }) {
  const toHex = (num) => Math.max(0, Math.min(255, num)).toString(16).padStart(2, '0')
  return `#${toHex(r)}${toHex(g)}${toHex(b)}`
}

function toPickerColor(color, fallback = '#000000') {
  const rgb = parseColorToRgb(color, fallback)
  return rgbToHex(rgb)
}

function rgbaFromColor(color, alpha, fallback = '#000000') {
  const rgb = parseColorToRgb(color, fallback)
  const safeAlpha = Math.max(0, Math.min(1, Number(alpha) || 0))
  return `rgba(${rgb.r}, ${rgb.g}, ${rgb.b}, ${safeAlpha})`
}

function onImageUpload(event) {
  const file = event.target?.files?.[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = () => {
    imageUrl.value = String(reader.result || '')
    themeMode.value = 'image'
  }
  reader.readAsDataURL(file)
}

function clearImage() {
  imageUrl.value = ''
}

function clampDimension(value, fallback) {
  return Math.max(320, Math.min(6000, Number(value) || fallback))
}

function markCustomSize() {
  if (sizePresetId.value !== 'custom') {
    sizePresetId.value = 'custom'
  }
}

function onWidthCommit() {
  const nextWidth = clampDimension(sizeControl.width, 1080)
  sizeControl.width = nextWidth
  if (aspectRatioLocked.value) {
    const ratio = lockedAspectRatio.value > 0 ? lockedAspectRatio.value : 1
    sizeControl.height = clampDimension(Math.round(nextWidth / ratio), 1350)
  }
  markCustomSize()
}

function onHeightCommit() {
  const nextHeight = clampDimension(sizeControl.height, 1350)
  sizeControl.height = nextHeight
  if (aspectRatioLocked.value) {
    const ratio = lockedAspectRatio.value > 0 ? lockedAspectRatio.value : 1
    sizeControl.width = clampDimension(Math.round(nextHeight * ratio), 1080)
  }
  markCustomSize()
}

function toggleAspectRatioLock() {
  aspectRatioLocked.value = !aspectRatioLocked.value
  if (aspectRatioLocked.value) {
    const safeHeight = Math.max(1, Number(sizeControl.height) || 1)
    lockedAspectRatio.value = (Number(sizeControl.width) || 1080) / safeHeight
  }
}

function insertManualBreak() {
  const marker = `\n${MANUAL_BREAK_MARK}\n`
  const textarea = contentTextareaRef.value
  if (!textarea) {
    cardData.content = `${cardData.content}${marker}`
    return
  }

  const start = textarea.selectionStart ?? cardData.content.length
  const end = textarea.selectionEnd ?? start
  cardData.content = `${cardData.content.slice(0, start)}${marker}${cardData.content.slice(end)}`

  nextTick(() => {
    const cursor = start + marker.length
    textarea.focus()
    textarea.setSelectionRange(cursor, cursor)
  })
}

function normalizeBreakMarks(text) {
  return String(text || '')
    .replace(/\r\n/g, '\n')
    .replace(/\[\[page\]\]/gi, '\f')
}

function fitsText(candidate, maxHeight) {
  if (!measureContentRef.value) return true
  measureContentRef.value.innerHTML = renderMarkdownBody(candidate, ' ')
  return measureContentRef.value.scrollHeight <= maxHeight + 1
}

function pickBreakPoint(text, bestFit) {
  if (bestFit >= text.length) return bestFit
  const minLimit = Math.max(1, Math.floor(bestFit * 0.65))

  for (let idx = bestFit; idx >= minLimit; idx -= 1) {
    const ch = text[idx - 1]
    if (ch === '\n') return idx
    if (/[，。！？；：、,.!?;:\s]/.test(ch)) return idx
  }
  return bestFit
}

function paginateSection(sectionText, maxHeight) {
  const pages = []
  let remaining = sectionText
  if (!remaining.length) return ['']

  while (remaining.length) {
    if (fitsText(remaining, maxHeight)) {
      pages.push(remaining.replace(/\s+$/g, ''))
      break
    }

    let low = 1
    let high = remaining.length
    let best = 1

    while (low <= high) {
      const mid = Math.floor((low + high) / 2)
      const candidate = remaining.slice(0, mid)
      if (fitsText(candidate, maxHeight)) {
        best = mid
        low = mid + 1
      } else {
        high = mid - 1
      }
    }

    const cut = pickBreakPoint(remaining, best)
    let current = remaining.slice(0, cut).replace(/\s+$/g, '')
    if (!current.length) {
      current = remaining.slice(0, Math.max(1, cut))
    }

    pages.push(current)
    remaining = remaining.slice(cut).replace(/^[\s\n]+/g, '')
  }

  return pages.length ? pages : ['']
}

async function repaginate() {
  await nextTick()
  if (!measureContentRef.value) return

  const maxHeight = measureContentRef.value.clientHeight
  if (!maxHeight) {
    pagedContents.value = [normalizeBreakMarks(cardData.content).replace(/\f/g, '') || '']
    return
  }

  const normalized = normalizeBreakMarks(cardData.content)
  const manualSections = normalized.split('\f')
  const pages = []

  for (let i = 0; i < manualSections.length; i += 1) {
    const section = manualSections[i]
    pages.push(...paginateSection(section, maxHeight))
  }

  pagedContents.value = pages.length ? pages : ['']
}

function pageText(index) {
  return `${index + 1} / ${totalPages.value}`
}

function updatePreviewCardWidth() {
  if (!previewFrameRef.value) return
  const nextWidth = Math.max(260, Math.min(760, Math.floor(previewFrameRef.value.clientWidth - 30)))
  previewCardWidth.value = nextWidth
}

let repaginateTimer = null
function scheduleRepaginate() {
  if (repaginateTimer) clearTimeout(repaginateTimer)
  repaginateTimer = setTimeout(() => {
    repaginate()
  }, 0)
}

const paginationSignature = computed(() =>
  [
    cardData.content,
    cardData.title,
    cardData.author,
    cardData.time,
    display.icon,
    display.title,
    display.author,
    display.time,
    display.page,
    templateId.value,
    currentSize.value.width,
    currentSize.value.height,
    typography.align,
    typography.lineHeight,
    typography.padding,
    typography.radius,
    fontFamilyConfig.icon,
    fontFamilyConfig.title,
    fontFamilyConfig.body,
    fontFamilyConfig.author,
    fontFamilyConfig.time,
    fontFamilyConfig.page,
    fontSizeConfig.icon,
    fontSizeConfig.title,
    fontSizeConfig.body,
    fontSizeConfig.author,
    fontSizeConfig.time,
    fontSizeConfig.page,
    previewCardWidth.value,
  ].join('|'),
)

watch(
  paginationSignature,
  () => {
    scheduleRepaginate()
  },
  { immediate: true },
)

let resizeObserver = null
let handleWindowResize = null

onMounted(() => {
  updatePreviewCardWidth()
  scheduleRepaginate()

  handleWindowResize = () => {
    updatePreviewCardWidth()
    scheduleRepaginate()
  }

  window.addEventListener('resize', handleWindowResize)

  if (window.ResizeObserver && previewFrameRef.value) {
    resizeObserver = new ResizeObserver(() => {
      updatePreviewCardWidth()
      scheduleRepaginate()
    })
    resizeObserver.observe(previewFrameRef.value)
  }
})

onBeforeUnmount(() => {
  if (repaginateTimer) clearTimeout(repaginateTimer)
  if (handleWindowResize) window.removeEventListener('resize', handleWindowResize)
  if (resizeObserver) resizeObserver.disconnect()
  for (let i = 0; i < uploadedFontObjectUrls.value.length; i += 1) {
    URL.revokeObjectURL(uploadedFontObjectUrls.value[i])
  }
})

function resetAll() {
  selectedFontCategory.value = 'all'

  sizePresetId.value = 'square'
  sizeControl.width = 1080
  sizeControl.height = 1080
  aspectRatioLocked.value = true
  lockedAspectRatio.value = 1

  templateId.value = 'clean'

  display.icon = true
  display.title = true
  display.author = true
  display.time = true
  display.page = true
  display.watermark = true

  cardData.icon = '✦'
  cardData.title = '文字卡片标题'
  cardData.content = '在左侧输入文案，快速生成用于分享的卡片内容。'
  cardData.author = '@Text2Card'
  cardData.time = formatNow()
  cardData.watermark = 'TEXT2CARD'

  typography.align = 'left'
  typography.lineHeight = 1.5
  typography.padding = 56
  typography.radius = 28

  fontFamilyConfig.icon = 'sans'
  fontFamilyConfig.title = 'serif'
  fontFamilyConfig.body = 'serif'
  fontFamilyConfig.author = 'sans'
  fontFamilyConfig.time = 'sans'
  fontFamilyConfig.page = 'mono'
  fontFamilyConfig.watermark = 'sans'

  fontSizeConfig.icon = 28
  fontSizeConfig.title = 34
  fontSizeConfig.body = 24
  fontSizeConfig.author = 16
  fontSizeConfig.time = 15
  fontSizeConfig.page = 15
  fontSizeConfig.watermark = 20

  themeMode.value = 'preset'
  presetThemeId.value = 'sunrise'
  usePresetTextColor.value = true
  customTextColor.value = '#1d2430'
  solidColor.value = '#f3eee4'
  gradientStart.value = '#f7d4c3'
  gradientEnd.value = '#f1a58f'
  gradientAngle.value = 140
  imageUrl.value = ''
  imageOverlay.value = 0.2
  imageOverlayColor.value = '#000000'
  watermarkOpacity.value = 0.12

  shadow.enabled = true
  shadow.x = 0
  shadow.y = 16
  shadow.blur = 40
  shadow.spread = 0
  shadow.color = '#121a2a'
  shadow.opacity = 0.28
}

function getPreviewCards() {
  if (!previewFrameRef.value) return []
  return Array.from(previewFrameRef.value.querySelectorAll('.preview-card'))
}

function getTimeStamp() {
  const now = new Date()
  const pad = (num) => String(num).padStart(2, '0')
  return `${now.getFullYear()}${pad(now.getMonth() + 1)}${pad(now.getDate())}-${pad(now.getHours())}${pad(now.getMinutes())}${pad(now.getSeconds())}`
}

async function downloadCardNode(node, filename) {
  const canvas = await html2canvas(node, {
    scale: 2,
    useCORS: true,
    backgroundColor: null,
  })
  const link = document.createElement('a')
  link.download = filename
  link.href = canvas.toDataURL('image/png')
  link.click()
}

async function renderCardBlob(node) {
  const canvas = await html2canvas(node, {
    scale: 2,
    useCORS: true,
    backgroundColor: null,
  })
  return new Promise((resolve, reject) => {
    canvas.toBlob((blob) => {
      if (blob) {
        resolve(blob)
        return
      }
      reject(new Error('无法生成图片数据'))
    }, 'image/png')
  })
}

function downloadBlob(blob, filename) {
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.download = filename
  link.href = url
  link.click()
  setTimeout(() => URL.revokeObjectURL(url), 1000)
}

async function downloadSinglePage(index) {
  if (isExporting.value) return
  const cards = getPreviewCards()
  const node = cards[index]
  if (!node) return

  isExporting.value = true
  try {
    const stamp = getTimeStamp()
    await downloadCardNode(node, `text-card-${stamp}-p${index + 1}.png`)
  } catch (error) {
    console.error(error)
    window.alert('下载失败，请稍后重试。')
  } finally {
    isExporting.value = false
  }
}

async function downloadAllPages() {
  if (isExporting.value) return
  const cards = getPreviewCards()
  if (!cards.length) return

  isExporting.value = true
  try {
    const stamp = getTimeStamp()
    if (bulkDownloadMode.value === 'zip') {
      const zip = new JSZip()
      for (let i = 0; i < cards.length; i += 1) {
        const blob = await renderCardBlob(cards[i])
        zip.file(`text-card-p${i + 1}-of-${cards.length}.png`, blob)
      }
      const zipBlob = await zip.generateAsync({
        type: 'blob',
        compression: 'DEFLATE',
        compressionOptions: { level: 6 },
      })
      downloadBlob(zipBlob, `text-cards-${stamp}.zip`)
    } else {
      for (let i = 0; i < cards.length; i += 1) {
        await downloadCardNode(cards[i], `text-card-${stamp}-p${i + 1}-of-${cards.length}.png`)
        await new Promise((resolve) => setTimeout(resolve, 120))
      }
    }
  } catch (error) {
    console.error(error)
    window.alert('批量下载失败，请稍后重试。')
  } finally {
    isExporting.value = false
  }
}
</script>

<template>
  <div class="layout">
    <aside class="panel">
      <div class="panel-head">
        <div class="panel-title">
          <h1>文字转卡片工具</h1>
          <p>支持自动分页、手动分页、单页下载与批量下载</p>
        </div>
        <button type="button" class="panel-github" title="GitHub 仓库信息（预留）" disabled>
          GitHub 仓库（预留）
        </button>
      </div>

      <section class="section-block">
        <h2>预设尺寸模板 + 尺寸控制</h2>
        <label class="field">
          <span>预设模板</span>
          <select v-model="sizePresetId">
            <option v-for="preset in sizePresets" :key="preset.id" :value="preset.id">
              {{ preset.name }}（{{ preset.ratioLabel }}）
            </option>
          </select>
        </label>
        <div class="aspect-lock-row">
          <span>锁定纵横比</span>
          <button
            type="button"
            class="aspect-lock-btn"
            :class="{ locked: aspectRatioLocked }"
            :title="aspectRatioLocked ? '已锁定纵横比' : '未锁定纵横比'"
            @click="toggleAspectRatioLock"
          >
            {{ aspectRatioLocked ? '🔒' : '🔓' }}
          </button>
        </div>
        <div class="grid-fields">
          <label class="field">
            <span>宽度（px）</span>
            <input
              v-model.number="sizeControl.width"
              type="number"
              min="320"
              max="6000"
              @change="onWidthCommit"
              @blur="onWidthCommit"
            />
          </label>
          <label class="field">
            <span>高度（px）</span>
            <input
              v-model.number="sizeControl.height"
              type="number"
              min="320"
              max="6000"
              @change="onHeightCommit"
              @blur="onHeightCommit"
            />
          </label>
        </div>
        <div class="stat-box">
          <span>当前比例：{{ currentSize.ratio }}</span>
          <span>当前尺寸：{{ currentSize.width }} × {{ currentSize.height }}</span>
        </div>
      </section>

      <section class="section-block">
        <h2>卡片模板</h2>
        <div class="chips">
          <button
            v-for="item in cardTemplates"
            :key="item.id"
            type="button"
            class="chip"
            :class="{ active: templateId === item.id }"
            @click="templateId = item.id"
          >
            {{ item.name }}
          </button>
        </div>
      </section>

      <section class="section-block">
        <h2>内容与元素</h2>
        <label class="field">
          <span>图标</span>
          <input v-model="cardData.icon" type="text" maxlength="4" />
          <div class="chips tiny">
            <button v-for="icon in iconQuickPick" :key="icon" type="button" class="chip" @click="cardData.icon = icon">
              {{ icon }}
            </button>
          </div>
        </label>

        <label class="field">
          <span>标题（支持 Markdown）</span>
          <input v-model="cardData.title" type="text" maxlength="200" />
          <small class="field-hint">单字样式：<code>{{ INLINE_STYLE_MARK }}</code></small>
        </label>

        <label class="field">
          <span>正文（支持 Markdown，输入 <code>{{ MANUAL_BREAK_MARK }}</code> 可手动分页）</span>
          <textarea ref="contentTextareaRef" v-model="cardData.content" rows="9" maxlength="12000"></textarea>
          <small class="field-hint">单字样式：<code>{{ INLINE_STYLE_MARK }}</code></small>
        </label>

        <div class="inline-actions">
          <button type="button" class="btn ghost mini" @click="insertManualBreak">插入分页符</button>
        </div>

        <label class="field">
          <span>署名</span>
          <input v-model="cardData.author" type="text" maxlength="48" />
        </label>

        <label class="field">
          <span>时间</span>
          <input v-model="cardData.time" type="text" maxlength="40" />
        </label>
        <div class="inline-actions">
          <button type="button" class="btn ghost mini" @click="setCurrentTime">填充当前时间</button>
        </div>

        <label class="field">
          <span>水印</span>
          <input v-model="cardData.watermark" type="text" maxlength="60" />
        </label>

        <div class="stat-box">
          <span>{{ statsText }}</span>
          <span>总页数自动计算，无需手动输入</span>
        </div>

        <div class="toggle-grid">
          <label class="toggle-row"><input v-model="display.icon" type="checkbox" /> 显示图标</label>
          <label class="toggle-row"><input v-model="display.title" type="checkbox" /> 显示标题</label>
          <label class="toggle-row"><input v-model="display.author" type="checkbox" /> 显示署名</label>
          <label class="toggle-row"><input v-model="display.time" type="checkbox" /> 显示时间</label>
          <label class="toggle-row"><input v-model="display.page" type="checkbox" /> 显示页码</label>
          <label class="toggle-row"><input v-model="display.watermark" type="checkbox" /> 显示水印</label>
        </div>
      </section>

      <section class="section-block">
        <h2>字体选择（独立控制 + 字号）</h2>
        <div class="font-tools">
          <label class="field">
            <span>字体分类</span>
            <select v-model="selectedFontCategory">
              <option v-for="item in fontCategoryOptions" :key="item.id" :value="item.id">
                {{ item.label }}
              </option>
            </select>
          </label>
          <label class="field">
            <span>上传字体（ttf/otf/woff/woff2）</span>
            <input type="file" accept=".ttf,.otf,.woff,.woff2,font/ttf,font/otf,font/woff,font/woff2" multiple @change="onFontUpload" />
          </label>
        </div>
        <div class="font-grid">
          <div v-for="item in fontTargetOptions" :key="item.key" class="font-row">
            <span class="font-row-label">{{ item.label }}</span>
            <select v-model="fontFamilyConfig[item.key]">
              <option v-for="font in getSelectableFontOptions(item.key)" :key="font.id" :value="font.id">
                {{ font.name }}
              </option>
            </select>
            <input v-model.number="fontSizeConfig[item.key]" type="number" min="10" max="120" />
          </div>
        </div>
      </section>

      <section class="section-block">
        <h2>版式与阴影</h2>
        <div class="grid-fields">
          <label class="field">
            <span>文字对齐</span>
            <select v-model="typography.align">
              <option value="left">左对齐</option>
              <option value="center">居中</option>
              <option value="right">右对齐</option>
            </select>
          </label>
          <label class="field">
            <span>圆角</span>
            <input v-model.number="typography.radius" type="number" min="0" max="80" />
          </label>
        </div>

        <label class="field">
          <span class="range-head"><span>正文行距</span><strong>{{ typography.lineHeight.toFixed(2) }}</strong></span>
          <input v-model.number="typography.lineHeight" type="range" min="1.1" max="2.5" step="0.05" />
        </label>

        <label class="field">
          <span class="range-head"><span>卡片留白</span><strong>{{ typography.padding }}px</strong></span>
          <input v-model.number="typography.padding" type="range" min="20" max="120" step="2" />
        </label>

        <label class="toggle-row standalone">
          <input v-model="shadow.enabled" type="checkbox" />
          启用阴影
        </label>

        <div class="grid-fields">
          <label class="field">
            <span>阴影色（支持 hex/rgb/hsl/英文色）</span>
            <div class="color-input">
              <input type="color" :value="toPickerColor(shadow.color, '#121a2a')" @input="shadow.color = $event.target.value" />
              <input v-model="shadow.color" type="text" placeholder="#121a2a / rgb(...) / hsl(...) / navy" />
            </div>
          </label>
          <label class="field">
            <span>阴影透明度</span>
            <input v-model.number="shadow.opacity" type="number" min="0" max="1" step="0.05" />
          </label>
        </div>

        <div class="grid-fields">
          <label class="field">
            <span>水平偏移</span>
            <input v-model.number="shadow.x" type="number" min="-120" max="120" />
          </label>
          <label class="field">
            <span>垂直偏移</span>
            <input v-model.number="shadow.y" type="number" min="-120" max="120" />
          </label>
          <label class="field">
            <span>模糊</span>
            <input v-model.number="shadow.blur" type="number" min="0" max="180" />
          </label>
          <label class="field">
            <span>扩散</span>
            <input v-model.number="shadow.spread" type="number" min="-60" max="60" />
          </label>
        </div>
      </section>

      <section class="section-block">
        <h2>主题背景</h2>
        <div class="chips">
          <button type="button" class="chip" :class="{ active: themeMode === 'preset' }" @click="themeMode = 'preset'">预设主题</button>
          <button type="button" class="chip" :class="{ active: themeMode === 'solid' }" @click="themeMode = 'solid'">纯色</button>
          <button type="button" class="chip" :class="{ active: themeMode === 'gradient' }" @click="themeMode = 'gradient'">渐变</button>
          <button type="button" class="chip" :class="{ active: themeMode === 'image' }" @click="themeMode = 'image'">背景图</button>
        </div>

        <div v-if="themeMode === 'preset'" class="inline-top-gap">
          <div class="chips">
            <button
              v-for="item in themePresets"
              :key="item.id"
              type="button"
              class="chip"
              :class="{ active: presetThemeId === item.id }"
              @click="presetThemeId = item.id"
            >
              {{ item.name }}
            </button>
          </div>
          <label class="toggle-row standalone">
            <input v-model="usePresetTextColor" type="checkbox" />
            使用预设文字颜色
          </label>
        </div>

        <div v-if="themeMode === 'solid'" class="inline-top-gap">
          <label class="field">
            <span>纯色（支持 hex/rgb/hsl/英文色）</span>
            <div class="color-input">
              <input type="color" :value="toPickerColor(solidColor, '#f3eee4')" @input="solidColor = $event.target.value" />
              <input v-model="solidColor" type="text" placeholder="#f3eee4 / rgb(...) / hsl(...) / beige" />
            </div>
          </label>
        </div>

        <div v-if="themeMode === 'gradient'" class="grid-fields inline-top-gap">
          <label class="field">
            <span>起始色（支持多格式）</span>
            <div class="color-input">
              <input type="color" :value="toPickerColor(gradientStart, '#f7d4c3')" @input="gradientStart = $event.target.value" />
              <input v-model="gradientStart" type="text" placeholder="#f7d4c3 / rgb(...) / hsl(...)" />
            </div>
          </label>
          <label class="field">
            <span>结束色（支持多格式）</span>
            <div class="color-input">
              <input type="color" :value="toPickerColor(gradientEnd, '#f1a58f')" @input="gradientEnd = $event.target.value" />
              <input v-model="gradientEnd" type="text" placeholder="#f1a58f / rgb(...) / hsl(...)" />
            </div>
          </label>
          <label class="field full">
            <span class="range-head"><span>渐变角度</span><strong>{{ gradientAngle }}°</strong></span>
            <input v-model.number="gradientAngle" type="range" min="0" max="360" step="1" />
          </label>
        </div>

        <div v-if="themeMode === 'image'" class="inline-top-gap">
          <label class="field">
            <span>上传背景图片</span>
            <input type="file" accept="image/*" @change="onImageUpload" />
          </label>
          <div class="inline-actions">
            <button type="button" class="btn ghost mini" :disabled="!imageUrl" @click="clearImage">清除背景图</button>
          </div>
          <label class="field">
            <span>遮罩颜色（支持 hex/rgb/hsl/英文色）</span>
            <div class="color-input">
              <input type="color" :value="toPickerColor(imageOverlayColor, '#000000')" @input="imageOverlayColor = $event.target.value" />
              <input v-model="imageOverlayColor" type="text" placeholder="#000000 / rgba(...) / hsl(...) / black" />
            </div>
          </label>
          <label class="field">
            <span class="range-head"><span>遮罩强度</span><strong>{{ imageOverlay.toFixed(2) }}</strong></span>
            <input v-model.number="imageOverlay" type="range" min="0" max="0.85" step="0.05" />
          </label>
        </div>

        <div class="grid-fields inline-top-gap">
          <label class="field">
            <span>文字颜色（支持多格式）</span>
            <div class="color-input">
              <input
                type="color"
                :value="toPickerColor(customTextColor, '#1d2430')"
                :disabled="themeMode === 'preset' && usePresetTextColor"
                @input="customTextColor = $event.target.value"
              />
              <input
                v-model="customTextColor"
                type="text"
                placeholder="#1d2430 / rgb(...) / hsl(...) / white"
                :disabled="themeMode === 'preset' && usePresetTextColor"
              />
            </div>
          </label>
          <label class="field">
            <span>水印透明度</span>
            <input v-model.number="watermarkOpacity" type="number" min="0" max="1" step="0.05" />
          </label>
        </div>
      </section>

      <div class="actions">
        <button class="btn ghost" type="button" @click="resetAll">恢复默认</button>
      </div>
    </aside>

    <section class="preview-section">
      <div class="preview-head">
        <strong>预设模板：{{ sizePresets.find((item) => item.id === sizePresetId)?.name || '自定义' }}</strong>
        <span>{{ currentSize.width }} × {{ currentSize.height }} · 共 {{ totalPages }} 页</span>
      </div>

      <div class="preview-tools">
        <label class="batch-mode">
          <span>批量下载方式</span>
          <select v-model="bulkDownloadMode">
            <option value="zip">打包 ZIP</option>
            <option value="separate">逐张下载</option>
          </select>
        </label>
        <button class="btn primary mini" type="button" :disabled="isExporting || totalPages < 1" @click="downloadAllPages">
          <span class="download-icon">⬇</span>
          {{ isExporting ? '下载中...' : bulkDownloadMode === 'zip' ? '下载 ZIP' : '逐张下载全部页' }}
        </button>
      </div>

      <div ref="previewFrameRef" class="preview-frame">
        <div class="preview-stack">
          <section v-for="(_, index) in pagedContents" :key="`page-${index}`" class="page-panel">
            <article
              class="card preview-card"
              :class="`card--${templateId}`"
              :style="[cardStyle, { width: `${previewCardWidth}px` }]"
            >
              <button
                class="icon-btn card-download-btn"
                type="button"
                :disabled="isExporting"
                :title="`下载第 ${index + 1} 页`"
                @click="downloadSinglePage(index)"
              >
                <span class="download-icon">⬇</span>
              </button>
              <span v-if="display.watermark" class="card-watermark" :style="fontStyle.watermark">
                {{ cardData.watermark || 'WATERMARK' }}
              </span>

              <header class="card-head">
                <div class="head-left">
                  <span v-if="display.icon" class="card-icon" :style="fontStyle.icon">
                    {{ cardData.icon || '✦' }}
                  </span>
                  <h2 v-if="display.title" class="card-title" :style="fontStyle.title" v-html="renderedTitleHtml"></h2>
                </div>
                <span v-if="display.page" class="card-page" :style="fontStyle.page">{{ pageText(index) }}</span>
              </header>

              <div class="card-content" :style="fontStyle.body" v-html="pagedContentHtml[index]"></div>

              <footer class="card-meta">
                <span v-if="display.author" class="meta-item" :style="fontStyle.author">{{ cardData.author || '@author' }}</span>
                <span v-if="display.time" class="meta-item" :style="fontStyle.time">{{ cardData.time || formatNow() }}</span>
              </footer>
            </article>
          </section>
        </div>
      </div>
    </section>

    <div class="measure-root" aria-hidden="true">
      <article class="card measure-card" :class="`card--${templateId}`" :style="[cardStyle, { width: `${previewCardWidth}px` }]">
        <span v-if="display.watermark" class="card-watermark" :style="fontStyle.watermark">
          {{ cardData.watermark || 'WATERMARK' }}
        </span>

        <header class="card-head">
          <div class="head-left">
            <span v-if="display.icon" class="card-icon" :style="fontStyle.icon">{{ cardData.icon || '✦' }}</span>
            <h2 v-if="display.title" class="card-title" :style="fontStyle.title" v-html="renderedTitleHtml"></h2>
          </div>
          <span v-if="display.page" class="card-page" :style="fontStyle.page">1 / 99</span>
        </header>

        <div ref="measureContentRef" class="card-content" :style="fontStyle.body"></div>

        <footer class="card-meta">
          <span v-if="display.author" class="meta-item" :style="fontStyle.author">{{ cardData.author || '@author' }}</span>
          <span v-if="display.time" class="meta-item" :style="fontStyle.time">{{ cardData.time || formatNow() }}</span>
        </footer>
      </article>
    </div>
  </div>
</template>
