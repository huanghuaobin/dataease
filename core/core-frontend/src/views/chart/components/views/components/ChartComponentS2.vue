<script lang="ts" setup>
import {
  computed,
  inject,
  onBeforeUnmount,
  onMounted,
  PropType,
  reactive,
  ref,
  shallowRef,
  ShallowRef,
  toRaw,
  toRefs
} from 'vue'
import { getData } from '@/api/chart'
import chartViewManager from '@/views/chart/components/js/panel'
import { useAppStoreWithOut } from '@/store/modules/app'
import { dvMainStoreWithOut } from '@/store/modules/data-visualization/dvMain'
import ViewTrackBar from '@/components/visualization/ViewTrackBar.vue'
import { storeToRefs } from 'pinia'
import { S2ChartView } from '@/views/chart/components/js/panel/types/impl/s2'
import { ElPagination } from 'element-plus-secondary'
import ChartError from '@/views/chart/components/views/components/ChartError.vue'
import { defaultsDeep, cloneDeep } from 'lodash-es'
import { BASE_VIEW_CONFIG } from '../../editor/util/chart'
import { customAttrTrans, customStyleTrans, recursionTransObj } from '@/utils/canvasStyle'
import { deepCopy } from '@/utils/utils'
import { useEmitt } from '@/hooks/web/useEmitt'
import { trackBarStyleCheck } from '@/utils/canvasUtils'

const dvMainStore = dvMainStoreWithOut()
const { nowPanelTrackInfo, nowPanelJumpInfo, mobileInPc } = storeToRefs(dvMainStore)
const { emitter } = useEmitt()

const props = defineProps({
  element: {
    type: Object,
    default() {
      return {
        propValue: null
      }
    }
  },
  view: {
    type: Object as PropType<ChartObj>,
    default() {
      return {
        propValue: null
      }
    }
  },
  showPosition: {
    type: String,
    required: false,
    default: 'canvas'
  },
  scale: {
    type: Number,
    required: false,
    default: 1
  },
  terminal: {
    type: String,
    default: 'pc'
  }
})

const emit = defineEmits(['onChartClick', 'onDrillFilters', 'onJumpClick'])

const { view, showPosition, scale, terminal } = toRefs(props)

const isError = ref(false)
const errMsg = ref('')
const chartExtRequest = inject('chartExtRequest') as ShallowRef<object>

const state = reactive({
  trackBarStyle: {
    position: 'absolute',
    left: '50px',
    top: '50px'
  },
  linkageActiveParam: null,
  pointParam: null,
  loading: false,
  data: { fields: [] }, // 图表数据
  pageInfo: {
    total: 0,
    pageSize: 20,
    currentPage: 1
  },
  totalItems: 0,
  showPage: false
})
// 图表数据不用全响应式
let chartData = shallowRef<Partial<Chart['data']>>({
  fields: []
})

const containerId = 'container-' + showPosition.value + '-' + view.value.id
const viewTrack = ref(null)

const calcData = (view: Chart, callback, resetPageInfo = true) => {
  if (view.tableId || view['dataFrom'] === 'template') {
    isError.value = false
    const v = JSON.parse(JSON.stringify(view))
    getData(v)
      .then(res => {
        if (res.code && res.code !== 0) {
          isError.value = true
          errMsg.value = res.msg
        } else {
          chartData.value = res?.data as Partial<Chart['data']>
          state.totalItems = res?.totalItems
          dvMainStore.setViewDataDetails(view.id, chartData.value)
          emit('onDrillFilters', res?.drillFilters)
          renderChart(res as unknown as Chart, resetPageInfo)
        }
        callback?.()
      })
      .catch(() => {
        callback?.()
      })
  } else {
    callback?.()
  }
}
// 图表对象不用响应式
let myChart = null
// 实际渲染的视图信息，适应缩放
let actualChart: ChartObj
const renderChartFromDialog = (viewInfo: Chart, chartDataInfo) => {
  chartData.value = chartDataInfo
  renderChart(viewInfo, false)
}
const renderChart = (viewInfo: Chart, resetPageInfo: boolean) => {
  if (!viewInfo) {
    return
  }
  // view 为引用对象 需要存库 view.data 直接赋值会导致保存不必要的数据
  actualChart = deepCopy({
    ...defaultsDeep(viewInfo, cloneDeep(BASE_VIEW_CONFIG)),
    data: chartData.value
  } as ChartObj)

  recursionTransObj(customAttrTrans, actualChart.customAttr, scale.value, terminal.value)
  recursionTransObj(customStyleTrans, actualChart.customStyle, scale.value, terminal.value)

  setupPage(actualChart, resetPageInfo)
  myChart?.facet.timer?.stop()
  myChart?.destroy()
  const chartView = chartViewManager.getChartView(
    viewInfo.render,
    viewInfo.type
  ) as S2ChartView<any>
  myChart = chartView.drawChart({
    container: containerId,
    chart: toRaw(actualChart),
    chartObj: myChart,
    pageInfo: state.pageInfo,
    action,
    resizeAction
  })
  myChart?.render()
  initScroll()
}

const setupPage = (chart: ChartObj, resetPageInfo?: boolean) => {
  const customAttr = chart.customAttr
  if (chart.type !== 'table-info' || customAttr.basicStyle.tablePageMode !== 'page') {
    state.showPage = false
    return
  }
  const pageInfo = state.pageInfo
  pageInfo.pageSize = customAttr.basicStyle.tablePageSize ?? 20
  if (state.totalItems > state.pageInfo.pageSize) {
    pageInfo.total = state.totalItems
    state.showPage = true
  } else {
    state.showPage = false
  }
  if (resetPageInfo) {
    state.pageInfo.currentPage = 1
  }
}

const initScroll = () => {
  // 首先回到最顶部，然后计算行高*行数作为top，最后判断：如果top<数据量*行高，继续滚动，否则回到顶部
  const customAttr = actualChart.customAttr
  const senior = actualChart.senior
  if (
    senior?.scrollCfg?.open &&
    chartData.value.tableRow?.length &&
    (view.value.type === 'table-normal' || (view.value.type === 'table-info' && !state.showPage))
  ) {
    // 防止多次渲染
    myChart.facet.timer?.stop()
    if (myChart.store.get('scrollY') !== 0) {
      myChart.store.set('scrollY', 0)
      myChart.render()
    }
    // 平滑滚动，兼容原有的滚动速率设置
    // 假设原设定为 2 行间隔 2 秒，换算公式为: 滚动到底部的时间 = 未展示部分行数 / 2行 * 2秒
    const offsetHeight = document.getElementById(containerId).offsetHeight
    // 没显示就不滚了
    if (!offsetHeight) {
      return
    }
    const rowHeight = customAttr.tableCell.tableItemHeight
    const headerHeight = customAttr.tableHeader.tableTitleHeight
    const scrollBarSize = myChart.theme.scrollBar.size
    const scrollHeight =
      rowHeight * chartData.value.tableRow.length + headerHeight - offsetHeight + scrollBarSize
    // 显示内容没撑满
    if (scrollHeight < scrollBarSize) {
      return
    }
    const viewHeight = offsetHeight - headerHeight - scrollBarSize
    const scrollViewCount = chartData.value.tableRow.length - viewHeight / rowHeight
    const duration = (scrollViewCount / senior.scrollCfg.row) * senior.scrollCfg.interval
    myChart.facet.scrollWithAnimation(
      { offsetY: { value: scrollHeight, animate: false } },
      duration,
      initScroll
    )
  }
}

const showPage = computed(() => {
  if (view.value.type !== 'table-info') {
    return false
  }
  return state.showPage
})

const handleCurrentChange = pageNum => {
  let extReq = { goPage: pageNum }
  if (chartExtRequest.value) {
    extReq = { ...extReq, ...chartExtRequest.value }
  }
  const chart = { ...view.value, chartExtRequest: extReq }
  calcData(chart, null, false)
}

const action = param => {
  // 下钻 联动 跳转
  state.pointParam = param
  if (trackMenu.value.length < 2) {
    // 只有一个事件直接调用
    trackClick(trackMenu.value[0])
  } else {
    // 图表关联多个事件
    const barStyleTemp = {
      left: param.x - 50,
      top: param.y + 10
    }
    trackBarStyleCheck(props.element, barStyleTemp, props.scale)
    state.trackBarStyle.left = barStyleTemp.left + 'px'
    state.trackBarStyle.top = barStyleTemp.top + 'px'
    viewTrack.value.trackButtonClick()
  }
}
const appStore = useAppStoreWithOut()

const isDataEaseBi = computed(() => appStore.getIsDataEaseBi)

const trackClick = trackAction => {
  const param = state.pointParam
  if (!param?.data?.dimensionList) {
    return
  }
  const linkageParam = {
    option: 'linkage',
    name: state.pointParam.data.name,
    viewId: view.value.id,
    dimensionList: state.pointParam.data.dimensionList,
    quotaList: state.pointParam.data.quotaList
  }
  const jumpParam = {
    option: 'jump',
    name: state.pointParam.data.name,
    viewId: view.value.id,
    dimensionList: state.pointParam.data.dimensionList,
    quotaList: state.pointParam.data.quotaList,
    sourceType: state.pointParam.data.sourceType
  }
  switch (trackAction) {
    case 'linkageAndDrill':
      dvMainStore.addViewTrackFilter(linkageParam)
      emit('onChartClick', param)
      break
    case 'drill':
      emit('onChartClick', param)
      break
    case 'linkage':
      dvMainStore.addViewTrackFilter(linkageParam)
      break
    case 'jump':
      if (isDataEaseBi.value || mobileInPc.value) return
      emit('onJumpClick', jumpParam)
      break
    default:
      break
  }
}

const trackMenu = computed(() => {
  let trackMenuInfo = []
  if (showPosition.value === 'viewDialog') {
    return trackMenuInfo
  }
  let linkageCount = 0
  let jumpCount = 0
  chartData.value?.fields?.forEach(item => {
    const sourceInfo = view.value.id + '#' + item.id
    if (nowPanelTrackInfo.value[sourceInfo]) {
      linkageCount++
    }
    if (nowPanelJumpInfo.value[sourceInfo]) {
      jumpCount++
    }
  })
  jumpCount &&
    view.value?.jumpActive &&
    !isDataEaseBi.value &&
    !mobileInPc.value &&
    trackMenuInfo.push('jump')
  linkageCount && view.value?.linkageActive && trackMenuInfo.push('linkage')
  view.value.drillFields.length && trackMenuInfo.push('drill')
  // 如果同时配置jump linkage drill 切配置联动时同时下钻 在实际只显示两个 '跳转' '联动和下钻'
  if (trackMenuInfo.length === 3 && props.element.actionSelection.linkageActive === 'auto') {
    trackMenuInfo = ['jump', 'linkageAndDrill']
  } else if (
    trackMenuInfo.length === 2 &&
    props.element.actionSelection.linkageActive === 'auto' &&
    !trackMenuInfo.includes('jump')
  ) {
    trackMenuInfo = ['linkageAndDrill']
  }
  return trackMenuInfo
})

const resizeAction = resizeColumn => {
  if (showPosition.value !== 'canvas') {
    return
  }
  const fieldId: string = resizeColumn.info.meta.field
  const { basicStyle } = view.value.customAttr
  const containerWidth = document.getElementById(containerId).offsetWidth
  const column = basicStyle.tableFieldWidth?.find(i => i.fieldId === fieldId)
  let tableWidth: ChartBasicStyle['tableFieldWidth']
  const width = parseFloat(((resizeColumn.info.resizedWidth / containerWidth) * 100).toFixed(2))
  if (column) {
    column.width = width
    tableWidth = [...basicStyle.tableFieldWidth]
  } else {
    const tmp = { fieldId, width }
    tableWidth = basicStyle.tableFieldWidth?.length ? [...basicStyle.tableFieldWidth, tmp] : [tmp]
  }
  emitter.emit('set-table-column-width', tableWidth)
}
defineExpose({
  calcData,
  renderChart,
  renderChartFromDialog,
  trackMenu
})

let timer
const resize = (width, height) => {
  if (timer) {
    clearTimeout(timer)
  }
  timer = setTimeout(() => {
    myChart?.changeSheetSize(width, height)
    myChart?.facet.timer?.stop()
    myChart?.render()
    initScroll()
  }, 500)
}
const preSize = [0, 0]
const TOLERANCE = 1
let resizeObserver: ResizeObserver
onMounted(() => {
  resizeObserver = new ResizeObserver(([entry] = []) => {
    const [size] = entry.borderBoxSize || []
    // 拖动的时候宽高重新计算，误差范围内不重绘，误差先设置为1
    if (!(preSize[0] || preSize[1])) {
      preSize[0] = size.inlineSize
      preSize[1] = size.blockSize
    }
    const widthOffset = Math.abs(size.inlineSize - preSize[0])
    const heightOffset = Math.abs(size.blockSize - preSize[1])
    if (widthOffset < TOLERANCE && heightOffset < TOLERANCE) {
      return
    }
    preSize[0] = size.inlineSize
    preSize[1] = size.blockSize
    resize(size.inlineSize, size.blockSize)
  })

  resizeObserver.observe(document.getElementById(containerId))
})
onBeforeUnmount(() => {
  myChart?.facet.timer?.stop()
  myChart?.destroy()
  resizeObserver?.disconnect()
})

const autoStyle = computed(() => {
  return {
    height: 100 / scale.value + '%!important',
    width: 100 / scale.value + '%!important',
    left: 50 * (1 - 1 / scale.value) + '%', // 放大余量 除以 2
    top: 50 * (1 - 1 / scale.value) + '%', // 放大余量 除以 2
    transform: 'scale(' + scale.value + ')'
  }
})

const autoHeightStyle = computed(() => {
  return {
    height: 20 * scale.value + 8 + 'px'
  }
})
</script>

<template>
  <div class="canvas-area">
    <view-track-bar
      ref="viewTrack"
      :track-menu="trackMenu"
      class="track-bar"
      :style="state.trackBarStyle"
      @trackClick="trackClick"
    />
    <div v-if="!isError" class="canvas-content">
      <div style="position: relative; height: 100%" :id="containerId"></div>
    </div>
    <el-row :style="autoHeightStyle" v-if="showPage && !isError">
      <div :style="autoStyle" class="table-page-info">
        <div>共{{ state.pageInfo.total }}条</div>
        <el-pagination
          class="table-page-content"
          layout="prev, pager, next"
          v-model:page-size="state.pageInfo.pageSize"
          v-model:current-page="state.pageInfo.currentPage"
          :pager-count="5"
          :total="state.pageInfo.total"
          @update:current-page="handleCurrentChange"
        />
      </div>
    </el-row>
    <chart-error v-if="isError" :err-msg="errMsg" />
  </div>
</template>

<style lang="less" scoped>
.canvas-area {
  z-index: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
  width: 100%;
  height: 100%;
  .canvas-content {
    flex: 1;
    width: 100%;
    overflow: hidden;
  }
}
.table-page-info {
  position: relative;
  padding-left: 4px;
  margin: 4px;
  height: 20px;
  display: flex;
  width: 100%;
  justify-content: space-between;
  color: grey;
  :deep(.table-page-content) {
    button,
    button[disabled] {
      color: grey;
      background: transparent !important;
    }
    ul li {
      &:not(.is-active) {
        color: grey;
      }
      background: transparent !important;
    }
  }
}
</style>
