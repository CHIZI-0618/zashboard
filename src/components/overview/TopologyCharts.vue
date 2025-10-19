<template>
  <div class="relative h-96 w-full overflow-hidden pt-12">
    <div
      ref="chart"
      class="h-full w-full"
    />
    <span
      class="border-base-content/30 text-base-content/10 bg-base-100/70 hidden"
      ref="colorRef"
    />
    <div
      v-if="sankeyData.nodes.length === 0"
      class="text-base-content/50 absolute inset-0 flex items-center justify-center"
    >
      <div class="text-center">
        <div>{{ t('noData') }}</div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { getIPLabelFromMap } from '@/helper/sourceip'
import { activeConnections } from '@/store/connections'
import { font, theme } from '@/store/settings'
import { useElementSize } from '@vueuse/core'
import { SankeyChart } from 'echarts/charts'
import { GridComponent, LegendComponent, TooltipComponent } from 'echarts/components'
import * as echarts from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { debounce } from 'lodash'
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'

echarts.use([SankeyChart, GridComponent, LegendComponent, TooltipComponent, CanvasRenderer])

const { t } = useI18n()
const colorRef = ref()
const chart = ref()
const colorSet = {
  baseContent10: '',
  baseContent30: '',
  baseContent: '',
  base70: '',
}

let fontFamily = ''

const updateColorSet = () => {
  const colorStyle = getComputedStyle(colorRef.value)

  colorSet.baseContent = colorStyle.getPropertyValue('--color-base-content').trim()
  colorSet.baseContent10 = colorStyle.color
  colorSet.baseContent30 = colorStyle.borderColor
  colorSet.base70 = colorStyle.backgroundColor
}

const updateFontFamily = () => {
  const baseColorStyle = getComputedStyle(colorRef.value)
  fontFamily = baseColorStyle.fontFamily
}

const sankeyData = computed(() => {
  const connections = activeConnections.value
  if (!connections || connections.length === 0) {
    return { nodes: [], links: [] }
  }

  const nodeMap = new Map<string, number>()
  const linkMap = new Map<string, number>()
  const layerMap = new Map<string, number>()
  const nodeTypeMap = new Map<string, string>()
  let nodeIndex = 0

  const addNode = (name: string, layer: number, type: string) => {
    if (!nodeMap.has(name)) {
      nodeMap.set(name, nodeIndex++)
      layerMap.set(name, layer)
      nodeTypeMap.set(name, type)
    }
    return nodeMap.get(name)!
  }

  connections.forEach((conn) => {
    const sourceIP = getIPLabelFromMap(conn.metadata.sourceIP)
    const rulePayload = conn.rulePayload ? `${conn.rule}: ${conn.rulePayload}` : conn.rule
    const chains = conn.chains || []

    if (chains.length === 0) return

    const chainLast = chains[chains.length - 1]
    const chainFirst = chains[0]

    const sourceNode = addNode(sourceIP, 0, t('sourceIPAddress'))
    const ruleNode = addNode(rulePayload, 1, t('ruleMatch'))

    if (chainFirst === chainLast) {
      const chainExitNode = addNode(chainFirst, 3, t('proxyChainExit'))

      const link1 = `${sourceNode}-${ruleNode}`
      const link2 = `${ruleNode}-${chainExitNode}`

      linkMap.set(link1, (linkMap.get(link1) || 0) + 1)
      linkMap.set(link2, (linkMap.get(link2) || 0) + 1)
    } else {
      const chainLastNode = addNode(chainLast, 2, t('proxyChainEntry'))
      const chainFirstNode = addNode(chainFirst, 3, t('proxyChainExit'))

      const link1 = `${sourceNode}-${ruleNode}`
      const link2 = `${ruleNode}-${chainLastNode}`
      const link3 = `${chainLastNode}-${chainFirstNode}`

      linkMap.set(link1, (linkMap.get(link1) || 0) + 1)
      linkMap.set(link2, (linkMap.get(link2) || 0) + 1)
      linkMap.set(link3, (linkMap.get(link3) || 0) + 1)
    }
  })

  const nodes = Array.from(nodeMap.entries()).map(([name, index]) => ({
    id: index,
    name: name,
    nodeType: nodeTypeMap.get(name) || t('unknown'),
    itemStyle: {
      color: layerColors[layerMap.get(name) || 0],
    },
  }))

  const links = Array.from(linkMap.entries()).map(([link, value]) => {
    const [source, target] = link.split('-').map(Number)
    return {
      source,
      target,
      value,
    }
  })

  return { nodes, links }
})

const layerColors = ['#6a6fc5', '#a8d4a0', '#fddb8a', '#f2a0a0']

const options = computed(() => ({
  backgroundColor: 'transparent',
  textStyle: {
    fontFamily: fontFamily || 'inherit',
    color: colorSet.baseContent,
  },
  tooltip: {
    trigger: 'item',
    triggerOn: 'mousemove',
    backgroundColor: colorSet.base70,
    borderColor: colorSet.baseContent30,
    textStyle: {
      color: colorSet.baseContent,
    },
    formatter: (params: {
      dataType: string
      data: {
        name: string
        nodeType?: string
        source: number
        target: number
        value: number
      }
    }) => {
      if (params.dataType === 'node') {
        return `${params.data.name}<br/>${t('nodeType')}: ${params.data.nodeType || t('unknown')}`
      } else if (params.dataType === 'edge') {
        const sourceNode = sankeyData.value.nodes.find((n) => n.id === params.data.source)
        const targetNode = sankeyData.value.nodes.find((n) => n.id === params.data.target)
        if (sourceNode && targetNode) {
          return `${sourceNode.name} → ${targetNode.name}<br/>${t('connectionCount')}: ${params.data.value}`
        }
        return `${t('connectionCount')}: ${params.data.value}`
      }
      return ''
    },
  },
  series: [
    {
      id: 'sankey',
      type: 'sankey',
      layout: 'none',
      data: sankeyData.value.nodes,
      links: sankeyData.value.links,
      emphasis: {
        focus: 'adjacency',
      },
      lineStyle: {
        color: 'gradient',
        curveness: 0.5,
      },
      itemStyle: {
        borderWidth: 1,
        borderColor: colorSet.baseContent30,
      },
      label: {
        color: colorSet.baseContent,
        fontSize: 12,
        formatter: (params: { name: string }) => {
          const name = params.name
          return name.length > 25 ? name.substring(0, 25) + '...' : name
        },
      },
      nodeGap: 4,
      nodeWidth: 15,
      nodeAlign: 'left',
      animation: true,
      animationDuration: 1000,
      animationEasing: 'cubicOut',
      animationDelay: (idx: number) => idx * 50,
    },
  ],
}))

onMounted(() => {
  updateColorSet()
  updateFontFamily()

  watch(theme, updateColorSet)
  watch(font, updateFontFamily)

  const myChart = echarts.init(chart.value)
  myChart.setOption(options.value)

  const updateChartData = debounce((newData: typeof sankeyData.value) => {
    if (myChart && newData.nodes.length > 0) {
      myChart.setOption(options.value)
    } else if (myChart && newData.nodes.length === 0) {
      myChart.clear()
    }
  }, 300)

  watch(sankeyData, updateChartData, { deep: true })

  watch([theme, font], () => {
    if (myChart) {
      myChart.setOption(options.value)
    }
  })

  const { width } = useElementSize(chart)
  const resize = debounce(() => {
    myChart.resize()
  }, 100)

  watch(width, resize)
})

onUnmounted(() => {
  if (chart.value) {
    const myChart = echarts.getInstanceByDom(chart.value)
    if (myChart) {
      myChart.dispose()
    }
  }
})
</script>
