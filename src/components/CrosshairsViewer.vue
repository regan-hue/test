<template>
  <div class="crosshairs-viewer">
    <div class="viewport-container">
      <div ref="axialViewport" class="viewport">
        <div class="viewport-label">轴向 (Axial)</div>
      </div>
      <div ref="sagittalViewport" class="viewport">
        <div class="viewport-label">矢状 (Sagittal)</div>
      </div>
      <div ref="coronalViewport" class="viewport">
        <div class="viewport-label">冠状 (Coronal)</div>
      </div>
    </div>
    <!-- 保存/恢复控制按钮 -->
    <div class="control-panel" v-if="!loading && !error">
      <button 
        class="control-btn update-btn" 
        @click="updateMPRPosition"
        :disabled="!canSave"
        title="更新MPR到固定位置"
      >
        🔄 Update My Project
      </button>
      <button 
        class="control-btn save-btn" 
        @click="saveCurrentPosition"
        :disabled="!canSave"
        title="保存当前MPR位置"
      >
        💾 保存位置
      </button>
      <button 
        class="control-btn restore-btn" 
        @click="restoreSavedPosition"
        :disabled="!hasSavedPosition"
        title="恢复到保存的位置"
      >
        🔄 恢复位置
      </button>
      <div v-if="hasSavedPosition" class="save-info">
        已保存位置
      </div>
    </div>
    <!-- 数据探针面板 -->
    <div class="probe-panel" v-if="!loading && !error && cameraProbeData">
      <div class="probe-header">
        <h3>📊 数据探针</h3>
        <button class="probe-toggle-btn" @click="toggleProbePanel">
          {{ probePanelExpanded ? '−' : '+' }}
        </button>
      </div>
      <div v-if="probePanelExpanded" class="probe-content">
        <!-- 鼠标位置探针 -->
        <div class="probe-section-divider">
          <span>🖱️ 鼠标位置</span>
        </div>
        <div 
          v-for="(mouseData, viewName) in mouseProbeData" 
          :key="'mouse-' + viewName"
          class="probe-viewport-section mouse-probe-section"
        >
          <div class="probe-viewport-title">
            {{ getViewportDisplayName(viewName) }}
            <span v-if="mouseData && mouseData.isValid" class="probe-status-indicator">●</span>
            <span v-else class="probe-status-indicator inactive">○</span>
          </div>
          <div v-if="mouseData && mouseData.isValid" class="probe-data-grid">
            <div class="probe-data-item">
              <span class="probe-label">画布坐标:</span>
              <span class="probe-value">
                [{{ mouseData.canvasPoint[0] }}, {{ mouseData.canvasPoint[1] }}]
              </span>
            </div>
            <div class="probe-data-item">
              <span class="probe-label">世界坐标 (mm):</span>
              <span class="probe-value highlight">
                {{ formatVector(mouseData.worldPoint) }}
              </span>
            </div>
            <div class="probe-data-item">
              <span class="probe-label">图像坐标 (体素):</span>
              <span class="probe-value highlight">
                {{ formatImagePoint(mouseData.imagePoint) }}
              </span>
            </div>
            <div class="probe-data-item" v-if="mouseData.pixelValue !== null">
              <span class="probe-label">像素值 (HU):</span>
              <span class="probe-value highlight pixel-value">
                {{ mouseData.pixelValue }}
              </span>
            </div>
          </div>
          <div v-else class="probe-no-data">
            鼠标移入视口查看坐标信息
          </div>
        </div>
        
        <!-- 相机参数探针 -->
        <div class="probe-section-divider">
          <span>📷 相机参数</span>
        </div>
        <div 
          v-for="(data, viewName) in cameraProbeData" 
          :key="'camera-' + viewName"
          class="probe-viewport-section"
        >
          <div class="probe-viewport-title">
            {{ getViewportDisplayName(viewName) }}
          </div>
          <div class="probe-data-grid">
            <div class="probe-data-item">
              <span class="probe-label">相机位置 (Position):</span>
              <span class="probe-value">
                {{ formatVector(data.position) }}
              </span>
            </div>
            <div class="probe-data-item">
              <span class="probe-label">焦点 (Focal Point):</span>
              <span class="probe-value">
                {{ formatVector(data.focalPoint) }}
              </span>
            </div>
            <div class="probe-data-item">
              <span class="probe-label">视图向上 (View Up):</span>
              <span class="probe-value">
                {{ formatVector(data.viewUp) }}
              </span>
            </div>
            <div class="probe-data-item">
              <span class="probe-label">平行缩放 (Parallel Scale):</span>
              <span class="probe-value">
                {{ formatNumber(data.parallelScale) }} mm
              </span>
            </div>
            <div class="probe-data-item" v-if="data.viewAngle !== null && data.viewAngle !== undefined">
              <span class="probe-label">视角 (View Angle):</span>
              <span class="probe-value">
                {{ formatNumber(data.viewAngle) }}°
              </span>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div v-if="loading" class="loading">加载中...</div>
    <div v-if="error" class="error">{{ error }}</div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import * as cornerstone from '@cornerstonejs/core'
import {
  init as cornerstoneInit,
  imageLoader,
  getRenderingEngine,
  RenderingEngine,
  Enums,
  volumeLoader,
  cache,
} from '@cornerstonejs/core'
import cornerstoneDICOMImageLoader from '@cornerstonejs/dicom-image-loader'
import dicomParser from 'dicom-parser'
import { 
  init as cornerstoneToolsInit, 
  addTool, 
  ToolGroupManager, 
  CrosshairsTool,
  Enums as ToolsEnums 
} from '@cornerstonejs/tools'
import { cornerstoneStreamingImageVolumeLoader } from '@cornerstonejs/streaming-image-volume-loader'

// Props 定义
const props = defineProps({
  studyInstanceUID: {
    type: String,
    required: true,
  },
  seriesInstanceUID: {
    type: String,
    required: true,
  },
})

const axialViewport = ref(null)
const sagittalViewport = ref(null)
const coronalViewport = ref(null)
const loading = ref(true)
const error = ref(null)

const renderingEngineId = 'myRenderingEngine'
const toolGroupId = 'myToolGroup'

// 保存的位置状态
const savedPosition = ref(null)
const hasSavedPosition = ref(false)
const canSave = ref(false)

// 数据探针相关状态
const cameraProbeData = ref(null)
const probePanelExpanded = ref(true)
let probeUpdateInterval = null

// 鼠标位置探针数据
const mouseProbeData = ref({
  axial: null,
  sagittal: null,
  coronal: null,
})

// 存储渲染引擎和viewport IDs的引用
let renderingEngineRef = null
let viewportIdsRef = null
let volumeRef = null

// DICOM 配置
// Orthanc服务器地址（根据Orthanc DICOMweb文档）
const orthancUrl = 'http://192.168.1.3:18997'

onMounted(async () => {
  try {
    // 初始化 Cornerstone
    await cornerstoneInit()
    await cornerstoneToolsInit()
    
    // 配置 DICOM 图像加载器
    // 设置 cornerstone 实例 - 这是必需的！
    cornerstoneDICOMImageLoader.external.cornerstone = cornerstone
    // 设置 dicomParser - 这也是必需的！
    cornerstoneDICOMImageLoader.external.dicomParser = dicomParser
    
    // 初始化 Web Worker 管理器
    const config = {
      maxWebWorkers: navigator.hardwareConcurrency || 4,
      startWebWorkersOnDemand: true,
      taskConfiguration: {
        decodeTask: {
          initializeCodecsOnStartup: false,
          usePDFJS: false,
          strict: false,
        },
      },
    }
    
    if (cornerstoneDICOMImageLoader.webWorkerManager) {
      cornerstoneDICOMImageLoader.webWorkerManager.initialize(config)
    }
    
    // 注册 DICOM 图像加载器
    // 使用 register 方法自动注册所有加载器
    if (cornerstoneDICOMImageLoader.register) {
      cornerstoneDICOMImageLoader.register(imageLoader)
    } else {
      // 如果没有 register 方法，手动注册
      if (cornerstoneDICOMImageLoader.wadouri && cornerstoneDICOMImageLoader.wadouri.loadImage) {
        imageLoader.registerImageLoader('dicomweb', cornerstoneDICOMImageLoader.wadouri.loadImage)
      } else if (cornerstoneDICOMImageLoader.loadImage) {
        imageLoader.registerImageLoader('dicomweb', cornerstoneDICOMImageLoader.loadImage)
      }
    }
    
    // 注册体积加载器
    volumeLoader.registerVolumeLoader('cornerstoneStreamingImageVolume', cornerstoneStreamingImageVolumeLoader)

    // 创建渲染引擎
    const renderingEngine = new RenderingEngine(renderingEngineId)
    renderingEngineRef = renderingEngine

    // 创建工具组
    const toolGroup = ToolGroupManager.createToolGroup(toolGroupId)

    // 添加 Crosshairs 工具
    addTool(CrosshairsTool)
    toolGroup.addTool(CrosshairsTool.toolName)
    
    // 配置 Crosshairs 工具以确保横截面正确显示
    toolGroup.setToolConfiguration(CrosshairsTool.toolName, {
      // 启用参考线显示
      getReferenceLineColor: () => [255, 255, 0], // 黄色参考线
      getReferenceLineControllable: () => true,
      getReferenceLineDraggableRotatable: () => true,
      // 确保在所有viewport之间同步
      autoPan: {
        enabled: false,
      },
    })
    
    toolGroup.setToolActive(CrosshairsTool.toolName, {
      bindings: [{ mouseButton: ToolsEnums.MouseBindings.Primary }],
    })

    // 添加其他必要的工具
    const { PanTool, WindowLevelTool, ZoomTool, StackScrollMouseWheelTool } = await import('@cornerstonejs/tools')
    addTool(PanTool)
    addTool(WindowLevelTool)
    addTool(ZoomTool)
    addTool(StackScrollMouseWheelTool)

    toolGroup.addTool(PanTool.toolName)
    toolGroup.addTool(WindowLevelTool.toolName)
    toolGroup.addTool(ZoomTool.toolName)
    toolGroup.addTool(StackScrollMouseWheelTool.toolName)

    toolGroup.setToolActive(PanTool.toolName, {
      bindings: [{ mouseButton: ToolsEnums.MouseBindings.Auxiliary }],
    })
    toolGroup.setToolActive(WindowLevelTool.toolName, {
      bindings: [{ mouseButton: ToolsEnums.MouseBindings.Secondary }],
    })
    toolGroup.setToolActive(ZoomTool.toolName, {
      bindings: [{ mouseButton: ToolsEnums.MouseBindings.Primary, modifierKey: ToolsEnums.KeyboardBindings.Shift }],
    })
    toolGroup.setToolActive(StackScrollMouseWheelTool.toolName, {
      bindings: [],
    })

    // 获取系列中的所有实例
    const imageIds = await fetchInstances()
    
    if (!imageIds || imageIds.length === 0) {
      throw new Error('未找到DICOM实例')
    }

    // 创建 viewport IDs
    const viewportIds = {
      axial: 'axial-viewport',
      sagittal: 'sagittal-viewport',
      coronal: 'coronal-viewport',
    }

    // 创建 Volume Viewports（用于多平面重建）
    const viewportInputArray = [
      {
        viewportId: viewportIds.axial,
        type: Enums.ViewportType.ORTHOGRAPHIC,
        element: axialViewport.value,
        defaultOptions: {
          orientation: Enums.OrientationAxis.AXIAL,
        },
      },
      {
        viewportId: viewportIds.sagittal,
        type: Enums.ViewportType.ORTHOGRAPHIC,
        element: sagittalViewport.value,
        defaultOptions: {
          orientation: Enums.OrientationAxis.SAGITTAL,
        },
      },
      {
        viewportId: viewportIds.coronal,
        type: Enums.ViewportType.ORTHOGRAPHIC,
        element: coronalViewport.value,
        defaultOptions: {
          orientation: Enums.OrientationAxis.CORONAL,
        },
      },
    ]
    console.log('x轴的方向:', Enums.OrientationAxis.AXIAL)

    renderingEngine.setViewports(viewportInputArray)

    // 将 viewports 添加到工具组
    toolGroup.addViewport(viewportIds.axial, renderingEngineId)
    toolGroup.addViewport(viewportIds.sagittal, renderingEngineId)
    toolGroup.addViewport(viewportIds.coronal, renderingEngineId)

    // 存储viewport IDs引用
    viewportIdsRef = viewportIds

    // 加载体积数据
    await loadVolume(renderingEngine, viewportIds, imageIds)

    // 启用保存功能
    canSave.value = true
    loading.value = false
  } catch (err) {
    console.error('初始化错误:', err)
    error.value = err.message || '初始化失败'
    loading.value = false
  }
})

async function fetchInstances() {
  try {
    // Orthanc API: 先通过SeriesInstanceUID查询找到系列
    // 使用tools/find接口查询
    const findResponse = await fetch('/tools/find', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        Level: 'Series',
        Query: {
          SeriesInstanceUID: props.seriesInstanceUID,
        },
      }),
    })

    if (!findResponse.ok) {
      throw new Error(`查询系列失败: ${findResponse.status}`)
    }

    const seriesIds = await findResponse.json()

    if (!seriesIds || seriesIds.length === 0) {
      throw new Error('未找到指定的系列')
    }

    // 获取第一个系列的详细信息
    const seriesId = seriesIds[0]
    const seriesResponse = await fetch(`/series/${seriesId}`)
    
    if (!seriesResponse.ok) {
      throw new Error(`获取系列信息失败: ${seriesResponse.status}`)
    }

    const seriesData = await seriesResponse.json()
    
    // 获取实例列表
    const instanceIds = seriesData.Instances || []
    
    if (instanceIds.length === 0) {
      throw new Error('系列中没有找到实例')
    }

    // 获取每个实例的元数据并排序
    const instancesWithMetadata = await Promise.all(
      instanceIds.map(async (instanceId) => {
        try {
          // 获取实例的标签（tags）
          const tagsResponse = await fetch(`/instances/${instanceId}/tags?simplify`)
          if (!tagsResponse.ok) {
            console.warn(`无法获取实例 ${instanceId} 的标签`)
            return { instanceId, imageId: `wadouri:/instances/${instanceId}/file`, position: null, sliceLocation: null, instanceNumber: null }
          }
          
          const tags = await tagsResponse.json()
          
          // 提取关键排序字段
          // ImagePositionPatient (0020,0032) - 图像在患者坐标系中的位置
          // SliceLocation (0020,1041) - 切片位置
          // InstanceNumber (0020,0013) - 实例编号（备用）
          const imagePositionPatient = tags['0020,0032'] || tags['ImagePositionPatient']
          const sliceLocation = tags['0020,1041'] || tags['SliceLocation']
          const instanceNumber = tags['0020,0013'] || tags['InstanceNumber']
          
          // 解析 ImagePositionPatient（格式通常是 "x\\y\\z"）
          let position = null
          if (imagePositionPatient && typeof imagePositionPatient === 'string') {
            const coords = imagePositionPatient.split('\\').map(Number)
            if (coords.length >= 3 && !coords.some(isNaN)) {
              position = coords
            }
          }
          
          return {
            instanceId,
            imageId: `wadouri:/instances/${instanceId}/file`,
            position,
            sliceLocation: sliceLocation ? Number(sliceLocation) : null,
            instanceNumber: instanceNumber ? Number(instanceNumber) : null,
          }
        } catch (err) {
          console.warn(`处理实例 ${instanceId} 时出错:`, err)
          return { instanceId, imageId: `wadouri:/instances/${instanceId}/file`, position: null, sliceLocation: null, instanceNumber: null }
        }
      })
    )

    // 按照空间位置排序
    instancesWithMetadata.sort((a, b) => {
      // 优先使用 ImagePositionPatient 的 Z 坐标（通常是轴向位置）
      if (a.position && b.position && a.position.length >= 3 && b.position.length >= 3) {
        // 对于轴向视图，使用 Z 坐标；对于矢状视图，使用 X 坐标；对于冠状视图，使用 Y 坐标
        // 通常轴向视图使用 Z 坐标排序
        const zDiff = a.position[2] - b.position[2]
        if (Math.abs(zDiff) > 0.001) {
          return zDiff
        }
      }
      
      // 如果没有 ImagePositionPatient，使用 SliceLocation
      if (a.sliceLocation !== null && b.sliceLocation !== null) {
        const sliceDiff = a.sliceLocation - b.sliceLocation
        if (Math.abs(sliceDiff) > 0.001) {
          return sliceDiff
        }
      }
      
      // 最后使用 InstanceNumber 作为备用排序
      if (a.instanceNumber !== null && b.instanceNumber !== null) {
        return a.instanceNumber - b.instanceNumber
      }
      
      // 如果都没有，保持原顺序
      return 0
    })

    // 提取排序后的 imageIds
    const imageIds = instancesWithMetadata.map(item => item.imageId)
    
    console.log(`已加载 ${imageIds.length} 个实例，已按空间位置排序`)
    
    return imageIds
  } catch (err) {
    console.error('获取实例失败:', err)
    throw err
  }
}

async function loadVolume(renderingEngine, viewportIds, imageIds) {
  try {
    // 定义体积ID
    const volumeId = `cornerstoneStreamingImageVolume:volume-${props.seriesInstanceUID}`
    
    // 创建体积加载器
    const volume = await volumeLoader.createAndCacheVolume(volumeId, {
      imageIds,
    })

    // 加载体积数据
    await volume.load()

    // 保存volume引用用于获取像素值
    volumeRef = volume

    // 设置每个viewport显示体积
    const axialViewport = renderingEngine.getViewport(viewportIds.axial)
    const sagittalViewport = renderingEngine.getViewport(viewportIds.sagittal)
    const coronalViewport = renderingEngine.getViewport(viewportIds.coronal)

    // 设置体积到各个viewport
    axialViewport.setVolumes([{ volumeId }])
    sagittalViewport.setVolumes([{ volumeId }])
    coronalViewport.setVolumes([{ volumeId }])
    
    // 渲染所有viewport（不设置任何相机位置，使用默认位置）
    renderingEngine.renderViewports([viewportIds.axial, viewportIds.sagittal, viewportIds.coronal])

    // 添加坐标输出功能
    setupCoordinateLogging(renderingEngine, viewportIds)
    
    // 启动数据探针
    startCameraProbe(renderingEngine, viewportIds)
  } catch (err) {
    console.error('加载体积失败:', err)
    throw err
  }
}

// 设置坐标日志记录
function setupCoordinateLogging(renderingEngine, viewportIds) {
  const viewports = {
    axial: renderingEngine.getViewport(viewportIds.axial),
    sagittal: renderingEngine.getViewport(viewportIds.sagittal),
    coronal: renderingEngine.getViewport(viewportIds.coronal),
  }

  const elements = {
    axial: axialViewport.value,
    sagittal: sagittalViewport.value,
    coronal: coronalViewport.value,
  }

  // 为每个viewport添加事件监听
  Object.keys(viewports).forEach((viewName) => {
    const element = elements[viewName]
    const viewport = viewports[viewName]

    // 鼠标移动事件 - 更新鼠标位置探针
    const handleMouseMove = (event) => {
      const rect = element.getBoundingClientRect()
      const mouseX = event.clientX - rect.left
      const mouseY = event.clientY - rect.top

      // 获取Canvas坐标
      const canvasPoint = [mouseX, mouseY]

      try {
        // 转换为世界坐标
        const worldPoint = viewport.canvasToWorld(canvasPoint)

        // 获取图像坐标（对于Volume viewport）
        let imagePoint = null
        let pixelValue = null
        
        try {
          // 对于Volume viewport，使用worldToIndex
          if (viewport.worldToIndex) {
            imagePoint = viewport.worldToIndex(worldPoint)
            
            // 获取像素值
            if (imagePoint && volumeRef) {
              pixelValue = getPixelValue(imagePoint, volumeRef)
            }
          }
        } catch (e) {
          // 如果转换失败，尝试其他方法
          try {
            if (viewport.getImageData) {
              const imageData = viewport.getImageData()
              // 可以在这里添加其他坐标转换逻辑
            }
          } catch (e2) {
            // 忽略错误
          }
        }

        // 更新鼠标位置探针数据
        updateMouseProbeData(viewName, {
          canvasPoint: [Math.round(mouseX), Math.round(mouseY)],
          worldPoint: worldPoint,
          imagePoint: imagePoint,
          pixelValue: pixelValue,
          isValid: true,
        })
      } catch (error) {
        // 如果坐标转换失败，标记为无效
        updateMouseProbeData(viewName, {
          canvasPoint: [Math.round(mouseX), Math.round(mouseY)],
          worldPoint: null,
          imagePoint: null,
          pixelValue: null,
          isValid: false,
        })
      }
    }

    // 鼠标离开视口时清除数据
    const handleMouseLeave = () => {
      updateMouseProbeData(viewName, null)
    }

    // 滚动事件 - 检测MPR滑动
    const handleWheel = (event) => {
      const rect = element.getBoundingClientRect()
      const mouseX = event.clientX - rect.left
      const mouseY = event.clientY - rect.top
      const deltaY = event.deltaY

      console.log(`[${viewName.toUpperCase()} Viewport] MPR滑动事件:`, {
        滚动方向: deltaY > 0 ? '向下' : '向上',
        滚动量: Math.abs(deltaY),
        鼠标位置: {
          x: Math.round(mouseX),
          y: Math.round(mouseY),
        },
      })

      // 延迟获取坐标，等待渲染完成
      setTimeout(() => {
        try {
          const canvasPoint = [mouseX, mouseY]
          const worldPoint = viewport.canvasToWorld(canvasPoint)

          let imagePoint = null
          if (viewport.getImageData && viewport.worldToIndex) {
            try {
              imagePoint = viewport.worldToIndex(worldPoint)
            } catch (e) {
              // 忽略错误
            }
          }

          console.log(`[${viewName.toUpperCase()} Viewport] 滑动后坐标:`, {
            鼠标位置: {
              x: Math.round(mouseX),
              y: Math.round(mouseY),
            },
            世界坐标: {
              x: worldPoint[0]?.toFixed(2),
              y: worldPoint[1]?.toFixed(2),
              z: worldPoint[2]?.toFixed(2),
            },
            图像坐标: imagePoint ? {
              i: Math.round(imagePoint[0]),
              j: Math.round(imagePoint[1]),
              k: imagePoint[2] ? Math.round(imagePoint[2]) : null,
            } : '无法获取',
          })
        } catch (error) {
          console.log(`[${viewName.toUpperCase()} Viewport] 滑动后鼠标位置:`, {
            x: Math.round(mouseX),
            y: Math.round(mouseY),
          })
        }
      }, 50)
    }

    // 添加事件监听器
    element.addEventListener('mousemove', handleMouseMove)
    element.addEventListener('mouseleave', handleMouseLeave)
    element.addEventListener('wheel', handleWheel, { passive: true })

    // 存储清理函数
    element._coordinateLoggingCleanup = () => {
      element.removeEventListener('mousemove', handleMouseMove)
      element.removeEventListener('mouseleave', handleMouseLeave)
      element.removeEventListener('wheel', handleWheel)
    }
  })
}

// 保存当前MPR位置
function saveCurrentPosition() {
  if (!renderingEngineRef || !viewportIdsRef) {
    console.error('渲染引擎未初始化')
    return
  }

  try {
    const viewports = {
      axial: renderingEngineRef.getViewport(viewportIdsRef.axial),
      sagittal: renderingEngineRef.getViewport(viewportIdsRef.sagittal),
      coronal: renderingEngineRef.getViewport(viewportIdsRef.coronal),
    }

    const savedState = {}

    // 保存每个viewport的相机状态
    Object.keys(viewports).forEach((viewName) => {
      const viewport = viewports[viewName]
      
      try {
        const camera = viewport.getCamera()
        
        // 保存相机状态（深拷贝所有重要属性）
        savedState[viewName] = {
          position: camera.position ? [...camera.position] : null,
          focalPoint: camera.focalPoint ? [...camera.focalPoint] : null,
          viewUp: camera.viewUp ? [...camera.viewUp] : null,
          parallelScale: camera.parallelScale !== undefined ? camera.parallelScale : null,
          viewAngle: camera.viewAngle !== undefined ? camera.viewAngle : null,
          parallelProjection: camera.parallelProjection !== undefined ? camera.parallelProjection : null,
        }
        
        // 输出保存的状态信息
        console.log(`[${viewName.toUpperCase()} Viewport] 保存的状态:`, savedState[viewName])
        console.log(`[${viewName.toUpperCase()} Viewport] 位置 (position):`, savedState[viewName].position)
        console.log(`[${viewName.toUpperCase()} Viewport] 焦点 (focalPoint):`, savedState[viewName].focalPoint)
        console.log(`[${viewName.toUpperCase()} Viewport] 视图向上 (viewUp):`, savedState[viewName].viewUp)
        console.log(`[${viewName.toUpperCase()} Viewport] 平行缩放 (parallelScale):`, savedState[viewName].parallelScale)
      } catch (err) {
        console.warn(`保存${viewName} viewport状态失败:`, err)
        savedState[viewName] = null
      }
    })

    savedPosition.value = savedState
    hasSavedPosition.value = true
    
    console.log('位置已保存:', savedState)
  } catch (err) {
    console.error('保存位置失败:', err)
    alert('保存位置失败: ' + err.message)
  }
}

// 更新MPR到固定位置
function updateMPRPosition() {
  if (!renderingEngineRef || !viewportIdsRef) {
    console.error('渲染引擎未初始化')
    return
  }

  try {
    // 从measurement.json中获取的点数据
    const points = [
      [31.50214385986328, -140.07765197753906, 859.3524780273438],
      [34.58005028416463, -141.6882029085106, 862.7489221061926],
      [36.77678577345103, -145.1404212737562, 865.4343205806341],
      [38.366546372851644, -149.20827428314414, 867.5557586555454],
      [39.26782760679941, -153.7362960800528, 869.0106359482569],
      [39.20237766015661, -158.56664231634903, 869.4903750293994],
      [37.59967700679518, -163.00194336521466, 868.3201652943335],
      [34.94380470594756, -166.3533901795924, 865.9276285186534],
      [31.767717502508365, -168.36767019269055, 862.8399388423335],
      [28.45614694304884, -169.21134772247547, 859.4781115063338],
      [25.14387881924835, -168.12118008491524, 855.8959789984457],
      [22.270828347227596, -165.91052583559835, 852.6453456080102],
      [20.01450129732287, -162.56020942448947, 849.9093146053336],
      [18.86841368358383, -158.21362607475058, 848.2194210195215],
      [18.989928792934382, -153.37034421552553, 847.7966966023463],
      [19.91445219632709, -148.7308044605562, 848.2355767150335],
      [21.984458101638452, -144.59890671975083, 849.9279877365689],
      [24.84366132888451, -141.80627991623572, 852.5964689029747],
      [28.116225529894905, -140.1623257331496, 855.8267680664645],
      [31.50214385986328, -140.07765197753906, 859.3524780273438]
    ]

    // 计算这些点的中心点（平均值）
    const calculateCenterPoint = (points) => {
      if (!points || points.length === 0) {
        return [0, 0, 0]
      }
      
      const sum = points.reduce(
        (acc, point) => [
          acc[0] + point[0],
          acc[1] + point[1],
          acc[2] + point[2]
        ],
        [0, 0, 0]
      )
      
      return [
        sum[0] / points.length,
        sum[1] / points.length,
        sum[2] / points.length
      ]
    }

    // 计算中心点作为焦点
    const fixedFocalPoint = calculateCenterPoint(points)
    const normalVector = [0.720, -0.078, -0.690]
    
    console.log('计算的中心点（焦点）:', fixedFocalPoint)

    // 归一化法向量
    const normalize = (v) => {
      const length = Math.sqrt(v[0] * v[0] + v[1] * v[1] + v[2] * v[2])
      if (length < 0.0001) return v
      return [v[0] / length, v[1] / length, v[2] / length]
    }

    const normalizedNormal = normalize(normalVector)

    // 计算viewUp向量（选择一个与法向量垂直的向量）
    // 使用一个参考向量（通常是[0, 0, 1]或[0, 1, 0]）来计算viewUp
    const referenceUp = [0, 0, 1]
    
    // 计算viewUp = referenceUp - (referenceUp · normal) * normal
    const dotProduct = referenceUp[0] * normalizedNormal[0] + 
                       referenceUp[1] * normalizedNormal[1] + 
                       referenceUp[2] * normalizedNormal[2]
    
    let viewUp = [
      referenceUp[0] - dotProduct * normalizedNormal[0],
      referenceUp[1] - dotProduct * normalizedNormal[1],
      referenceUp[2] - dotProduct * normalizedNormal[2]
    ]
    
    // 归一化viewUp
    viewUp = normalize(viewUp)
    
    // 如果viewUp太小（几乎平行于法向量），使用另一个参考向量
    const viewUpLength = Math.sqrt(viewUp[0] * viewUp[0] + viewUp[1] * viewUp[1] + viewUp[2] * viewUp[2])
    if (viewUpLength < 0.1) {
      const referenceUp2 = [0, 1, 0]
      const dotProduct2 = referenceUp2[0] * normalizedNormal[0] + 
                          referenceUp2[1] * normalizedNormal[1] + 
                          referenceUp2[2] * normalizedNormal[2]
      viewUp = [
        referenceUp2[0] - dotProduct2 * normalizedNormal[0],
        referenceUp2[1] - dotProduct2 * normalizedNormal[1],
        referenceUp2[2] - dotProduct2 * normalizedNormal[2]
      ]
      viewUp = normalize(viewUp)
    }

    // 计算相机位置（在法向量方向上，距离焦点一定距离）
    // 相机位置 = 焦点 - 法向量 * 距离
    // 使用一个合理的距离（例如体积的边界框大小）
    const distance = 500 // 可以根据实际情况调整
    const cameraPosition = [
      fixedFocalPoint[0] - normalizedNormal[0] * distance,
      fixedFocalPoint[1] - normalizedNormal[1] * distance,
      fixedFocalPoint[2] - normalizedNormal[2] * distance
    ]

    const viewports = {
      axial: renderingEngineRef.getViewport(viewportIdsRef.axial),
      sagittal: renderingEngineRef.getViewport(viewportIdsRef.sagittal),
      coronal: renderingEngineRef.getViewport(viewportIdsRef.coronal),
    }

    // 更新所有viewport的相机状态
    // 注意：为了保持Crosshairs工具正常工作，我们只更新焦点位置，保持标准视图方向
    Object.keys(viewports).forEach((viewName) => {
      const viewport = viewports[viewName]
      
      try {
        const camera = viewport.getCamera()
        
        // 只更新焦点位置，保持原有的viewUp和position方向
        // 这样可以确保Crosshairs工具继续正常工作
        camera.focalPoint = [...fixedFocalPoint]
        
        // 可选：如果需要根据法向量调整视图，可以更新position和viewUp
        // 但这可能会影响Crosshairs的显示
        // camera.position = [...cameraPosition]
        // camera.viewUp = [...viewUp]
        
        // 保持其他参数不变（如parallelScale）
        
        // 应用相机设置
        viewport.setCamera(camera)
        
        console.log(`[${viewName.toUpperCase()} Viewport] MPR焦点已更新:`, {
          焦点: fixedFocalPoint,
          原始相机位置: camera.position,
          原始viewUp: camera.viewUp
        })
      } catch (err) {
        console.warn(`更新${viewName} viewport位置失败:`, err)
      }
    })

    // 重新渲染所有viewport
    renderingEngineRef.renderViewports([
      viewportIdsRef.axial,
      viewportIdsRef.sagittal,
      viewportIdsRef.coronal,
    ])

    // 确保Crosshairs工具同步更新
    // Crosshairs工具通过监听viewport的相机变化来更新，但我们需要确保它被触发
    try {
      const toolGroup = ToolGroupManager.getToolGroup(toolGroupId)
      if (toolGroup) {
        // 强制刷新Crosshairs工具
        // 通过重新设置工具配置来触发更新
        toolGroup.setToolConfiguration(CrosshairsTool.toolName, {
          focalPoint: fixedFocalPoint,
        })
        
        // 触发所有viewport的相机更新事件，让Crosshairs工具知道位置已改变
        Object.keys(viewports).forEach((viewName) => {
          const viewport = viewports[viewName]
          // 通过重新设置相机来触发更新事件
          const camera = viewport.getCamera()
          viewport.setCamera(camera)
        })
        
        // 再次渲染以确保Crosshairs显示
        renderingEngineRef.renderViewports([
          viewportIdsRef.axial,
          viewportIdsRef.sagittal,
          viewportIdsRef.coronal,
        ])
        
        console.log('Crosshairs工具已同步更新到新位置')
      }
    } catch (err) {
      console.warn('同步Crosshairs工具失败:', err)
    }

    console.log('MPR位置已更新到固定位置')
  } catch (err) {
    console.error('更新MPR位置失败:', err)
    alert('更新MPR位置失败: ' + err.message)
  }
}

// 恢复到保存的位置
function restoreSavedPosition() {
  if (!renderingEngineRef || !viewportIdsRef || !savedPosition.value) {
    console.error('无法恢复: 没有保存的位置')
    return
  }

  try {
    const viewports = {
      axial: renderingEngineRef.getViewport(viewportIdsRef.axial),
      sagittal: renderingEngineRef.getViewport(viewportIdsRef.sagittal),
      coronal: renderingEngineRef.getViewport(viewportIdsRef.coronal),
    }

    // 恢复每个viewport的相机状态
    Object.keys(viewports).forEach((viewName) => {
      const viewport = viewports[viewName]
      const savedState = savedPosition.value[viewName]

      if (savedState && savedState !== null) {
        try {
          const camera = viewport.getCamera()
          
          // 恢复相机状态
          if (savedState.position && Array.isArray(savedState.position)) {
            camera.position = [...savedState.position]
          }
          if (savedState.focalPoint && Array.isArray(savedState.focalPoint)) {
            camera.focalPoint = [...savedState.focalPoint]
          }
          if (savedState.viewUp && Array.isArray(savedState.viewUp)) {
            camera.viewUp = [...savedState.viewUp]
          }
          if (savedState.parallelScale !== null && savedState.parallelScale !== undefined) {
            camera.parallelScale = savedState.parallelScale
          }
          if (savedState.viewAngle !== null && savedState.viewAngle !== undefined) {
            camera.viewAngle = savedState.viewAngle
          }
          if (savedState.parallelProjection !== null && savedState.parallelProjection !== undefined) {
            camera.parallelProjection = savedState.parallelProjection
          }

          // 应用相机设置
          viewport.setCamera(camera)
        } catch (err) {
          console.warn(`恢复${viewName} viewport状态失败:`, err)
        }
      }
    })

    // 重新渲染所有viewport
    renderingEngineRef.renderViewports([
      viewportIdsRef.axial,
      viewportIdsRef.sagittal,
      viewportIdsRef.coronal,
    ])

    console.log('位置已恢复')
  } catch (err) {
    console.error('恢复位置失败:', err)
    alert('恢复位置失败: ' + err.message)
  }
}

// 启动相机数据探针
function startCameraProbe(renderingEngine, viewportIds) {
  // 立即更新一次
  updateCameraProbe(renderingEngine, viewportIds)
  
  // 设置定时更新（每100ms更新一次，保证实时性）
  probeUpdateInterval = setInterval(() => {
    if (renderingEngineRef && viewportIdsRef) {
      updateCameraProbe(renderingEngineRef, viewportIdsRef)
    }
  }, 100)
}

// 更新相机探针数据
function updateCameraProbe(renderingEngine, viewportIds) {
  if (!renderingEngine || !viewportIds) {
    return
  }

  try {
    const viewports = {
      axial: renderingEngine.getViewport(viewportIds.axial),
      sagittal: renderingEngine.getViewport(viewportIds.sagittal),
      coronal: renderingEngine.getViewport(viewportIds.coronal),
    }

    const probeData = {}

    Object.keys(viewports).forEach((viewName) => {
      const viewport = viewports[viewName]
      
      try {
        const camera = viewport.getCamera()
        
        probeData[viewName] = {
          position: camera.position ? [...camera.position] : null,
          focalPoint: camera.focalPoint ? [...camera.focalPoint] : null,
          viewUp: camera.viewUp ? [...camera.viewUp] : null,
          parallelScale: camera.parallelScale !== undefined ? camera.parallelScale : null,
          viewAngle: camera.viewAngle !== undefined ? camera.viewAngle : null,
          parallelProjection: camera.parallelProjection !== undefined ? camera.parallelProjection : null,
        }
      } catch (err) {
        console.warn(`获取${viewName} viewport相机数据失败:`, err)
        probeData[viewName] = {
          position: null,
          focalPoint: null,
          viewUp: null,
          parallelScale: null,
          viewAngle: null,
          parallelProjection: null,
        }
      }
    })

    cameraProbeData.value = probeData
  } catch (err) {
    console.error('更新相机探针数据失败:', err)
  }
}

// 格式化向量显示
function formatVector(vector) {
  if (!vector || !Array.isArray(vector) || vector.length < 3) {
    return 'N/A'
  }
  return `[${vector[0]?.toFixed(2)}, ${vector[1]?.toFixed(2)}, ${vector[2]?.toFixed(2)}]`
}

// 格式化数字显示
function formatNumber(value) {
  if (value === null || value === undefined) {
    return 'N/A'
  }
  return value.toFixed(2)
}

// 获取视口显示名称
function getViewportDisplayName(viewName) {
  const names = {
    axial: '轴向 (Axial)',
    sagittal: '矢状 (Sagittal)',
    coronal: '冠状 (Coronal)',
  }
  return names[viewName] || viewName
}

// 切换探针面板展开/收起
function toggleProbePanel() {
  probePanelExpanded.value = !probePanelExpanded.value
}

// 更新鼠标位置探针数据
function updateMouseProbeData(viewName, data) {
  if (!mouseProbeData.value) {
    mouseProbeData.value = {
      axial: null,
      sagittal: null,
      coronal: null,
    }
  }
  mouseProbeData.value[viewName] = data
}

// 获取像素值
function getPixelValue(imagePoint, volume) {
  if (!imagePoint || !volume) {
    return null
  }

  try {
    const [i, j, k] = imagePoint.map(Math.round)
    
    // 获取volume的维度
    const dimensions = volume.dimensions
    if (!dimensions || dimensions.length < 3) {
      return null
    }

    // 检查索引是否在有效范围内
    if (i < 0 || i >= dimensions[0] || 
        j < 0 || j >= dimensions[1] || 
        k < 0 || k >= dimensions[2]) {
      return null
    }

    // 计算线性索引
    const index = k * dimensions[0] * dimensions[1] + j * dimensions[0] + i
    
    // 获取标量数据
    const scalarData = volume.getScalarData()
    if (!scalarData || index >= scalarData.length) {
      return null
    }

    return scalarData[index]
  } catch (error) {
    console.warn('获取像素值失败:', error)
    return null
  }
}

// 格式化图像坐标显示
function formatImagePoint(imagePoint) {
  if (!imagePoint || !Array.isArray(imagePoint)) {
    return 'N/A'
  }
  
  const formatted = imagePoint.map((val, idx) => {
    if (val === null || val === undefined) {
      return 'N/A'
    }
    return Math.round(val)
  })
  
  return `[${formatted.join(', ')}]`
}

onUnmounted(() => {
  // 清理数据探针定时器
  if (probeUpdateInterval) {
    clearInterval(probeUpdateInterval)
    probeUpdateInterval = null
  }

  // 清理事件监听器
  try {
    const elements = [axialViewport.value, sagittalViewport.value, coronalViewport.value]
    elements.forEach((element) => {
      if (element && element._coordinateLoggingCleanup) {
        element._coordinateLoggingCleanup()
      }
    })
  } catch (err) {
    console.error('清理事件监听器失败:', err)
  }

  // 清理资源
  try {
    const renderingEngine = getRenderingEngine(renderingEngineId)
    if (renderingEngine) {
      renderingEngine.destroy()
    }
    ToolGroupManager.destroyToolGroup(toolGroupId)
    
    // 清理缓存
    cache.purgeCache()
  } catch (err) {
    console.error('清理资源失败:', err)
  }
})
</script>

<style scoped>
.crosshairs-viewer {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.viewport-container {
  flex: 1;
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-template-rows: repeat(2, 1fr);
  gap: 4px;
  padding: 4px;
}

.viewport {
  width: 100%;
  height: 100%;
  background-color: #000;
  position: relative;
}

.viewport:nth-child(1) {
  grid-column: 1 / 2;
  grid-row: 1 / 2;
}

.viewport:nth-child(2) {
  grid-column: 2 / 3;
  grid-row: 1 / 2;
}

.viewport:nth-child(3) {
  grid-column: 1 / 3;
  grid-row: 2 / 3;
}

.loading,
.error {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 20px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  border-radius: 8px;
  z-index: 1000;
}

.error {
  background: rgba(200, 0, 0, 0.8);
}

.viewport-label {
  position: absolute;
  top: 10px;
  left: 10px;
  color: white;
  background: rgba(0, 0, 0, 0.6);
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  z-index: 10;
  pointer-events: none;
}

.control-panel {
  position: absolute;
  top: 10px;
  right: 10px;
  display: flex;
  flex-direction: column;
  gap: 8px;
  z-index: 100;
  background: rgba(0, 0, 0, 0.7);
  padding: 12px;
  border-radius: 8px;
  backdrop-filter: blur(4px);
}

.control-btn {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  color: white;
  min-width: 120px;
}

.save-btn {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

.save-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #5568d3 0%, #653a8f 100%);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.restore-btn {
  background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
}

.restore-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #e081eb 0%, #e4465b 100%);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(245, 87, 108, 0.4);
}

.update-btn {
  background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
}

.update-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, #3e9cee 0%, #00e2ee 100%);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(79, 172, 254, 0.4);
}

.control-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  background: #666;
}

.save-info {
  font-size: 12px;
  color: #4ade80;
  text-align: center;
  padding: 4px 0;
  font-weight: 500;
}

/* 数据探针面板样式 */
.probe-panel {
  position: absolute;
  bottom: 10px;
  left: 10px;
  z-index: 100;
  background: rgba(0, 0, 0, 0.85);
  border-radius: 8px;
  backdrop-filter: blur(8px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  min-width: 400px;
  max-width: 600px;
  max-height: 70vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5);
}

.probe-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.3) 0%, rgba(118, 75, 162, 0.3) 100%);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.probe-header h3 {
  margin: 0;
  font-size: 14px;
  font-weight: 600;
  color: white;
}

.probe-toggle-btn {
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: white;
  width: 24px;
  height: 24px;
  border-radius: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  font-weight: bold;
  transition: all 0.2s ease;
}

.probe-toggle-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: scale(1.1);
}

.probe-content {
  padding: 12px;
  overflow-y: auto;
  max-height: calc(70vh - 60px);
}

.probe-content::-webkit-scrollbar {
  width: 6px;
}

.probe-content::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.05);
  border-radius: 3px;
}

.probe-content::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3px;
}

.probe-content::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.3);
}

.probe-viewport-section {
  margin-bottom: 16px;
  padding: 12px;
  background: rgba(255, 255, 255, 0.03);
  border-radius: 6px;
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.probe-viewport-section:last-child {
  margin-bottom: 0;
}

.probe-viewport-title {
  font-size: 13px;
  font-weight: 600;
  color: #60a5fa;
  margin-bottom: 10px;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(96, 165, 250, 0.2);
  display: flex;
  align-items: center;
  gap: 8px;
}

.probe-status-indicator {
  color: #4ade80;
  font-size: 10px;
  animation: pulse 2s ease-in-out infinite;
}

.probe-status-indicator.inactive {
  color: rgba(255, 255, 255, 0.3);
  animation: none;
}

@keyframes pulse {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

.probe-section-divider {
  margin: 16px 0 12px 0;
  padding: 8px 12px;
  background: rgba(102, 126, 234, 0.15);
  border-left: 3px solid #667eea;
  font-size: 12px;
  font-weight: 600;
  color: #a5b4fc;
  border-radius: 4px;
}

.probe-section-divider:first-child {
  margin-top: 0;
}

.mouse-probe-section {
  border-left: 2px solid rgba(74, 222, 128, 0.3);
}

.probe-no-data {
  padding: 12px;
  text-align: center;
  color: rgba(255, 255, 255, 0.4);
  font-size: 11px;
  font-style: italic;
}

.probe-data-grid {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.probe-data-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
  padding: 6px 0;
}

.probe-label {
  font-size: 11px;
  color: rgba(255, 255, 255, 0.7);
  font-weight: 500;
}

.probe-value {
  font-size: 12px;
  color: #4ade80;
  font-family: 'Courier New', monospace;
  font-weight: 600;
  padding-left: 8px;
  word-break: break-all;
}

.probe-value.highlight {
  color: #60a5fa;
  font-weight: 700;
}

.probe-value.pixel-value {
  color: #fbbf24;
  font-size: 13px;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .probe-panel {
    min-width: 300px;
    max-width: calc(100vw - 20px);
  }
}
</style>

