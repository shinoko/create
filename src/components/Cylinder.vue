<template>
  <div class="container">
    <div ref="container" class="cylinder-container"></div>
    <div class="button-group">
      <button class="control-btn" @click="toggleRotation">
        {{ isRotating ? '停止旋转' : '开始旋转' }}
      </button>
      <input
        type="file"
        ref="fileInput"
        accept=".xlsx,.xls"
        style="display: none"
        @change="handleFileSelect"
      />
      <button class="control-btn file-btn" @click="triggerFileSelect">
        选择Excel文件
      </button>
    </div>
    <div v-if="showTooltip" class="tooltip" :style="tooltipStyle">
      第 {{ currentRow + 1 }} 行
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import * as XLSX from 'xlsx'
import bgImg from './bg'
import highlight from './highlight'

const container = ref(null)
const fileInput = ref(null)
const isRotating = ref(false)
let scene, camera, renderer, cylinder, sphere, controls
let excelData = ref([])
let bgTexture = null
let gridMeshes = [] // 存储所有网格的引用
let cylinderRotationSpeed = 0.01
let sphereRotationSpeed = 0.02
const showTooltip = ref(false)
const tooltipStyle = ref({
  left: '0px',
  top: '0px'
})
const currentRow = ref(0)
const raycaster = new THREE.Raycaster()
const mouse = new THREE.Vector2()

// 触发文件选择
const triggerFileSelect = () => {
  fileInput.value.click()
}

// 处理文件选择
const handleFileSelect = async (event) => {
  const file = event.target.files[0]
  if (!file) return

  try {
    const arrayBuffer = await file.arrayBuffer()
    const workbook = XLSX.read(arrayBuffer, { type: 'array' })
    const firstSheetName = workbook.SheetNames[0]
    const worksheet = workbook.Sheets[firstSheetName]
    const data = XLSX.utils.sheet_to_json(worksheet, { header: 1 })

    // 确保数据是10行64列的二维数组
    const formattedData = data.slice(0, lineCount - TEXT_ARRAY.length).map(row => {
      const newRow = new Array(columnCount).fill('')
      row.forEach((cell, index) => {
        if (index < columnCount) {
          newRow[index] = cell || ''
        }
      })
      return newRow
    })

    excelData.value = formattedData
    console.log('Excel数据加载成功：', excelData.value)
    
    // 更新网格显示
    updateGridTextures()
  } catch (error) {
    console.error('读取Excel文件失败：', error)
  }
}

// 更新网格纹理
const updateGridTextures = () => {
  // 清除现有的网格
  gridMeshes.forEach(mesh => {
    cylinder.remove(mesh)
    mesh.geometry.dispose()
    mesh.material.dispose()
  })
  gridMeshes = []

  // 重新创建网格
  const gridSize = radius * 2 * Math.PI / 36
  const gridHeight = height / lineCount
  const gridMaterial = new THREE.MeshPhongMaterial({
    color: 0xffffff,
    transparent: true,
    opacity: 0.8,
    side: THREE.DoubleSide
  })

  for (let row = 0; row < lineCount; row++) {
    const colsInRow = gridRows[row]

    for (let col = 0; col < colsInRow; col++) {
      const textTexture = createTextTexture(row, col)
      if (textTexture) {
        const gridGeometry = new THREE.PlaneGeometry(gridSize, gridHeight)
        const gridMesh = new THREE.Mesh(gridGeometry, gridMaterial.clone())

        const angle = ((col + (col + 1)) * Math.PI) / colsInRow
        const x = radius * Math.cos(angle)
        const z = radius * Math.sin(angle)
        const y = -height / 2 + gridHeight / 2 + row * gridHeight
        gridMesh.position.set(x, y, z)

        gridMesh.lookAt(0, y, 0)
        gridMesh.rotateY(Math.PI)

        gridMesh.material.map = textTexture
        cylinder.add(gridMesh)
        gridMeshes.push(gridMesh)
      }
    }
  }
}

const rowNumbers = Array.from({ length: 64 }, (_, i) => i + 1) // 生成1-64的数组
const TEXT_ARRAY = [
  ['阴', '阳'],
  ['天', '人', '地'],
  ['春', '夏', '秋', '冬'],
  ['木', '火', '水', '金', '土'],
  ['上', '下', '左', '右', '前', '后'],
  ['天枢', '天璇', '天机', '天权', '玉衡', '开阳', '瑶光'],
  ['乾', '兑', '离', '震', '巽', '坎', '艮', '坤'],
  ['甲', '乙', '丙', '丁', '戊', '己', '庚', '辛', '壬', '癸'],
  ['子', '丑', '寅', '卯', '辰', '巳', '午', '未', '申', '酉', '辛', '亥'],
  rowNumbers,
]


// 倒序获取每个子数组的长度
const lengthArray = TEXT_ARRAY.map(arr => arr.length).reverse()

// 加载背景图片
const loadBackgroundTexture = () => {
  return new Promise((resolve, reject) => {
    const textureLoader = new THREE.TextureLoader()
    textureLoader.load(
      // '/bg.jpg',
      bgImg,
      (texture) => {
        bgTexture = texture
        resolve(texture)
      },
      undefined,
      (error) => {
        console.error('加载背景图片失败：', error)
        reject(error)
      }
    )
  })
}

// 创建文字贴图
const createTextTexture = (row, col) => {
  let text = ''
  if (lineCount - row - 1 < TEXT_ARRAY.length) {
    const itemArray = TEXT_ARRAY[lineCount - row - 1]
    // 倒转文字顺序
    const reversedIndex = itemArray.length - 1 - col
    text = itemArray[reversedIndex]
  } else if (excelData.value.length > 0) {
    // 使用Excel数据
    const excelRow = lineCount - row - 1 - TEXT_ARRAY.length
    if (excelRow >= 0 && excelRow < excelData.value.length) {
      // 倒转Excel数据的列顺序
      const reversedCol = columnCount - 1 - col
      text = excelData.value[excelRow][reversedCol] || ''
    }
  }

  if (!text) return null

  const canvas = document.createElement('canvas')
  canvas.width = 256
  // canvas.height = 256
  canvas.height = height/lineCount
  const context = canvas.getContext('2d')

  const isLarge = lineCount - row - 1 < TEXT_ARRAY.length
  
  // 如果有背景图片，先绘制背景
  if (!isLarge && bgTexture) {
    const img = bgTexture.image
    // 计算网格大小
    const gridSize = radius * 2 * Math.PI / 80
    const gridHeight = height / lineCount

    // 设置绘制尺寸为网格大小
    let drawWidth = gridSize
    let drawHeight = gridHeight

    // 计算偏移量使图片居中
    let offsetX = (canvas.width - drawWidth) / 2
    let offsetY = (canvas.height - drawHeight) / 2
    
    // 绘制背景图片，确保填满整个画布
    context.drawImage(img, offsetX, offsetY, drawWidth, drawHeight)
  } else {
    // 如果没有背景图片，使用透明背景
    context.fillStyle = 'rgba(0, 0, 0, 0)'
    context.fillRect(0, 0, canvas.width, canvas.height)
  }
  
  
  // 设置文字样式
  context.font = isLarge ? 'bold 60px Arial' : 'bold 30px Arial'
  context.fillStyle = 'black'
  context.textAlign = 'center'
  context.textBaseline = 'middle'
  
  context.fillText(text, canvas.width / 2, canvas.height / 2)
  
  const texture = new THREE.CanvasTexture(canvas)
  texture.needsUpdate = true
  return texture
}

const toggleRotation = () => {
  isRotating.value = !isRotating.value
}

const lineCount = 32
const radius = 1800
const height = 6400
const columnCount = 64
// const gridRows = [36, 36, 12, 10, 8, 4, 2] // 每行的网格数量
const gridRows = Array(lineCount - TEXT_ARRAY.length).fill(columnCount).concat(lengthArray) // 每行的网格数量


const init = () => {
  // 创建场景
  scene = new THREE.Scene()

  // 创建相机
  camera = new THREE.PerspectiveCamera(
    60,
    window.innerWidth / window.innerHeight,
    0.1,
    12800
  )
  camera.position.set(3500, 1500, 500)
  camera.lookAt(0, 0, 0)

  // 创建渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true, logarithmicDepthBuffer: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setClearColor(0xf5f5dc)
  container.value.appendChild(renderer.domElement)

  // 创建圆柱体
  // const radius = 800
  // const height = 1400
  const geometry = new THREE.CylinderGeometry(radius, radius, height, 36, 5, true)
  const material = new THREE.MeshPhongMaterial({
    color: 0xffdd77,
    specular: 0xeeeeee,
    shininess: 30,
    wireframe: false
  })
  cylinder = new THREE.Mesh(geometry, material)
  cylinder.position.set(0, 0, 0)

  // 创建红色球体
  const sphereRadius = radius *1.01
  const sphereGeometry = new THREE.SphereGeometry(sphereRadius, 32, 32)
  const sphereMaterial = new THREE.MeshPhongMaterial({
    color: 0xff0000,
    specular: 0x448811,
    shininess: 30,
    emissive: 0xa20000,
    emissiveIntensity: 0.2
  })
  
  sphere = new THREE.Mesh(sphereGeometry, sphereMaterial)
  // 调整球体位置，使其与圆柱体分开
  const sphereOffset = radius * 0.2 // 设置偏移距离
  sphere.position.set(0, height / 2 + sphereRadius + sphereOffset, 0)
  cylinder.add(sphere)

  // 创建自定义网格线
  const gridLines = new THREE.Group()

  // const lineCount = 7
  // const gridRows = [36, 36, 12, 10, 8, 4, 2] // 每行的网格数量

  // 创建水平线
  for (let i = 0; i <= lineCount; i++) {
    const y = (i * height / lineCount) - height / 2
    const circleGeometry = new THREE.CircleGeometry(radius, 36)
    const circleEdges = new THREE.EdgesGeometry(circleGeometry)
    const circleLine = new THREE.LineSegments(
      circleEdges,
      new THREE.LineBasicMaterial({ color: 0x000000 })
    )
    circleLine.rotation.x = Math.PI / 2
    circleLine.position.y = y
    gridLines.add(circleLine)
  }

  // 创建垂直线
  for (let row = 0; row < lineCount; row++) {
    const colsInRow = gridRows[row]
    const yStart = (row * height / lineCount) - height / 2
    const yEnd = ((row + 1) * height / lineCount) - height / 2

    for (let i = 0; i < colsInRow; i++) {
      const angle = (i * Math.PI * 2) / colsInRow
      const x = radius * Math.cos(angle)
      const z = radius * Math.sin(angle)

      const lineGeometry = new THREE.BufferGeometry().setFromPoints([
        new THREE.Vector3(x, yStart, z),
        new THREE.Vector3(x, yEnd, z)
      ])
      const line = new THREE.Line(
        lineGeometry,
        new THREE.LineBasicMaterial({ color: 0x000000 })
      )
      gridLines.add(line)
    }
  }

  cylinder.add(gridLines)

  // 创建网格材质
  const gridMaterial = new THREE.MeshPhongMaterial({
    color: 0xffffff,
    transparent: true,
    opacity: 0.8,
    side: THREE.DoubleSide
  })

  // 创建所有网格
  const gridSize = radius * 2 * Math.PI / 36
  const gridHeight = height / lineCount

  for (let row = 0; row < lineCount; row++) {
    const colsInRow = gridRows[row]

    for (let col = 0; col < colsInRow; col++) {
      const textTexture = createTextTexture(row, col)
      if (textTexture) {
        const gridGeometry = new THREE.PlaneGeometry(gridSize, gridHeight)
        const gridMesh = new THREE.Mesh(gridGeometry, gridMaterial.clone())

        const angle = ((col + (col + 1)) * Math.PI) / colsInRow
        const x = radius * Math.cos(angle)
        const z = radius * Math.sin(angle)
        const y = -height / 2 + gridHeight / 2 + row * gridHeight
        gridMesh.position.set(x, y, z)

        gridMesh.lookAt(0, y, 0)
        gridMesh.rotateY(Math.PI)

        gridMesh.material.map = textTexture
        cylinder.add(gridMesh)
        gridMeshes.push(gridMesh)
      }
    }
  }

  // 创建顶部和底部的圆形
  const circleGeometry = new THREE.CircleGeometry(radius, 32) // 使用相同的半径
  const circleMaterial = new THREE.MeshPhongMaterial({
    color: 0xffdd77,
    side: THREE.DoubleSide
  })

  const topCircle = new THREE.Mesh(circleGeometry, circleMaterial)
  topCircle.position.y = height / 2
  topCircle.rotation.x = -Math.PI / 2

  const bottomCircle = new THREE.Mesh(circleGeometry, circleMaterial)
  bottomCircle.position.y = -height / 2
  bottomCircle.rotation.x = Math.PI / 2

  // 添加光源
  const ambientLight = new THREE.AmbientLight(0xffffff, 1)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(1, 1, 1) // 调整光源位置
  scene.add(directionalLight)

  // 添加所有对象到场景
  scene.add(cylinder)
  scene.add(topCircle)
  scene.add(bottomCircle)

  // 添加轨道控制器
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.enableRotate = true   // 启用旋转
  controls.enablePan = true     // 启用平移
  controls.enableZoom = true     // 启用缩放
  controls.autoRotate = false    // 初始状态不自动旋转
  controls.autoRotateSpeed = -3.0 // 设置自动旋转速度
  controls.target.set(0, 0, 0)   // 设置旋转中心点为圆柱体中心
  controls.minPolarAngle = 0     // 限制垂直旋转角度
  controls.maxPolarAngle = Math.PI // 限制垂直旋转角度
  controls.minDistance = 1800     // 设置最小缩放距离
  controls.maxDistance = 5500    // 设置最大缩放距离
  controls.zoomSpeed = 1.0       // 设置缩放速度

  // 设置平移限制
  controls.minPolarAngle = 0
  controls.maxPolarAngle = Math.PI
  controls.screenSpacePanning = true // 使用屏幕空间平移
  controls.panSpeed = 1.0 // 设置平移速度
  controls.maxPolarAngle = Math.PI // 限制垂直旋转角度
  controls.minPolarAngle = 0 // 限制垂直旋转角度

  // 开始动画循环
  animate()
}

const animate = () => {
  requestAnimationFrame(animate)
  
  if (isRotating.value) {
    cylinder.rotation.y += cylinderRotationSpeed
    sphere.rotation.y += sphereRotationSpeed
  }
  
  controls.update()
  renderer.render(scene, camera)
}

const handleResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}

// 修改鼠标移动事件处理
const onMouseMove = (event) => {
  // 计算鼠标在归一化设备坐标中的位置
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1

  // 更新射线
  raycaster.setFromCamera(mouse, camera)

  // 检查射线与圆柱体的相交
  const intersects = raycaster.intersectObject(cylinder)

  if (intersects.length > 0) {
    const intersect = intersects[0]
    const point = intersect.point
    // 计算行号
    const row = Math.floor((point.y + height / 2) / (height / lineCount))
    
    // 确保行号在有效范围内
    if (row >= 0 && row < lineCount-TEXT_ARRAY.length) {
      currentRow.value = lineCount - TEXT_ARRAY.length - row - 1
      showTooltip.value = true
      
      // 更新tooltip位置
      tooltipStyle.value = {
        left: `${event.clientX + 10}px`,
        top: `${event.clientY + 10}px`
      }
    } else {
      showTooltip.value = false
    }
  } else {
    showTooltip.value = false
  }
}

onMounted(async () => {
  try {
    await Promise.all([
      loadBackgroundTexture()
    ])
  } catch (error) {
    console.error('初始化失败：', error)
  }
  window.addEventListener('resize', handleResize)
  window.addEventListener('mousemove', onMouseMove)
  init()
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  window.removeEventListener('mousemove', onMouseMove)
  if (renderer) {
    renderer.dispose()
  }
  gridMeshes.forEach(mesh => {
    mesh.geometry.dispose()
    mesh.material.dispose()
  })
})
</script>

<style scoped>
.container {
  position: relative;
  width: 100%;
  height: 100vh;
}

.cylinder-container {
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.button-group {
  position: absolute;
  top: 20px;
  right: 20px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  z-index: 1000;
}

.control-btn {
  padding: 10px 20px;
  font-size: 16px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.control-btn:hover {
  background-color: #45a049;
}

.control-btn:active {
  background-color: #3d8b40;
}

.file-btn {
  background-color: #2196F3;
}

.file-btn:hover {
  background-color: #1976D2;
}

.file-btn:active {
  background-color: #1565C0;
}

.tooltip {
  position: fixed;
  background-color: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 8px 12px;
  border-radius: 4px;
  font-size: 14px;
  pointer-events: none;
  z-index: 1000;
  transition: opacity 0.2s;
}
</style>