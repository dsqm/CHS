<script setup lang="ts">
import { useEngine } from '../../composables/useEngine'
import html2canvas from 'html2canvas'

const props = withDefaults(defineProps<{ mode?: 'single' | 'upload' }>(), { mode: 'single' })

const { engine, toast } = useEngine()

async function captureElement(selector: string): Promise<HTMLCanvasElement | null> {
  const el = document.querySelector(selector) as HTMLElement | null
  if (!el) return null

  // 临时去掉 overflow 限制，让表格完整渲染
  const originalOverflow = el.style.overflow
  const originalOverflowX = el.style.overflowX
  el.style.overflow = 'visible'
  el.style.overflowX = 'visible'

  const canvas = await html2canvas(el, {
    scale: 2,
    useCORS: true,
    logging: false,
    backgroundColor: null,
    // 使用元素的完整 scrollWidth/scrollHeight
    width: el.scrollWidth,
    height: el.scrollHeight,
    windowWidth: el.scrollWidth + 200,
  })

  el.style.overflow = originalOverflow
  el.style.overflowX = originalOverflowX
  return canvas
}

async function exportEvaluateImage() {
  const config = engine.getConfig()
  const schemeName = config.meta.name || '未命名'
  const authorName = config.meta.author || '未知作者'
  const timestamp = new Date().toISOString().slice(0, 16).replace(/[-:]/g, '')

  const isUpload = props.mode === 'upload'
  const tableSelector = isUpload ? '.uploaded-main-content' : '.evaluate-main-content'
  const chartSelector = isUpload ? '.uploaded-candidates-chart' : '.code-candidates-chart'
  const heatmapSelector = isUpload ? '.uploaded-heatmap-wrapper' : '.heatmap-wrapper'

  try {
    const [tableCanvas, chartCanvas, heatmapCanvas] = await Promise.all([
      captureElement(tableSelector),
      captureElement(chartSelector),
      captureElement(heatmapSelector),
    ])

    if (!tableCanvas && !chartCanvas && !heatmapCanvas) {
      toast('未找到测评内容，请先运行测评', 'error')
      return
    }

    const canvases = [tableCanvas, chartCanvas, heatmapCanvas].filter(Boolean) as HTMLCanvasElement[]

    const scale = 2
    const headerHeight = 48
    const footerHeight = 40
    const gap = 16
    const padding = 24

    const maxWidth = Math.max(...canvases.map(c => c.width))
    const totalHeight = (headerHeight + footerHeight + padding * 2) * scale
      + canvases.reduce((sum, c) => sum + c.height + gap * scale, 0)

    const merged = document.createElement('canvas')
    merged.width = maxWidth + padding * 2 * scale
    merged.height = totalHeight
    const ctx = merged.getContext('2d')!

    const bgColor = getComputedStyle(document.documentElement).getPropertyValue('--bg2').trim() || '#ffffff'
    ctx.fillStyle = bgColor || '#ffffff'
    ctx.fillRect(0, 0, merged.width, merged.height)

    ctx.fillStyle = '#1f2937'
    ctx.font = `bold ${32 * scale}px "PingFang SC", "Microsoft YaHei", sans-serif`
    ctx.textAlign = 'center'
    ctx.fillText(`${authorName} · ${new Date().toLocaleDateString()} 测评报告`, merged.width / 2, (padding + 30) * scale)

    let y = (headerHeight + padding) * scale
    for (const c of canvases) {
      const x = (merged.width - c.width) / 2
      ctx.drawImage(c, x, y)
      y += c.height + gap * scale
    }

    ctx.fillStyle = '#9ca3af'
    ctx.font = `${11 * scale}px "PingFang SC", "Microsoft YaHei", sans-serif`
    ctx.textAlign = 'center'
    ctx.fillText(`生成时间: ${new Date().toLocaleString()} | chars-hijack 测评系统`, merged.width / 2, merged.height - padding * scale)

    const link = document.createElement('a')
    link.download = `${schemeName}_${authorName}_测评截图_${timestamp}.png`
    link.href = merged.toDataURL('image/png')
    link.click()

    toast('测评截图已保存')
  } catch (error) {
    console.error('导出测评截图失败:', error)
    toast('导出测评截图失败，请重试', 'error')
  }
}
</script>

<template>
  <button class="btn btn-sm btn-outline" @click="exportEvaluateImage">
    📷 截图分享
  </button>
</template>

<style scoped>
.btn-sm {
  padding: 6px 12px;
  font-size: 13px;
}
</style>
