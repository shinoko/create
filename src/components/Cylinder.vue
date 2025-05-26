<template>
  <div class="container">
    <div ref="container" class="cylinder-container"></div>
    <button class="control-btn" @click="toggleRotation">
      {{ isRotating ? '停止旋转' : '开始旋转' }}
    </button>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'

const container = ref(null)
const isRotating = ref(false)
let scene, camera, renderer, cylinder, controls

// 创建文字贴图
const createTextTexture = (row, col) => {
  const canvas = document.createElement('canvas')
  canvas.width = 256
  canvas.height = 256
  const context = canvas.getContext('2d')
  
  // 设置背景为透明
  context.fillStyle = 'rgba(0, 0, 0, 0)'
  context.fillRect(0, 0, canvas.width, canvas.height)
  
  // 设置文字样式
  context.font = 'bold 40px Arial'
  context.fillStyle = 'black'
  context.textAlign = 'center'
  context.textBaseline = 'middle'
  
  let text = ''
  switch(row) {
    case 6:
      text = col === 0 ? '阴' : '阳'
      break
    case 5: 
      const seasons = ['春', '夏', '秋', '冬']
      text = seasons[col]
      break
    case 4: 
      const trigrams = ['乾', '兑', '离', '震', '巽', '坎', '艮', '坤']
      text = trigrams[col]
      break
    case 3:
      const stems = ['甲', '乙', '丙', '丁', '戊', '己', '庚', '辛', '壬', '癸']
      text = stems[col]
      break
    case 2: 
      const branches = ['子', '丑', '寅', '卯', '辰', '巳', '午', '未', '申', '酉', '辛', '亥']
      text = branches[col]
      break
    case 1: 
      text = String(col + 1).padStart(3, '0')
      break
    case 0: 
      text = `姓名${String(col + 1).padStart(2, '0')}`
      break
  }
  
  context.fillText(text, canvas.width / 2, canvas.height / 2)
  
  const texture = new THREE.CanvasTexture(canvas)
  texture.needsUpdate = true
  return texture
}

const toggleRotation = () => {
  isRotating.value = !isRotating.value
  controls.autoRotate = isRotating.value
}

const init = () => {
  // 创建场景
  scene = new THREE.Scene()

  // 创建相机
  camera = new THREE.PerspectiveCamera(
    45,
    window.innerWidth / window.innerHeight,
    0.1,
    5000
  )
  camera.position.set(1500, 500, 1500)
  camera.lookAt(0, 0, 0)

  // 创建渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setClearColor(0xf5f5dc)
  container.value.appendChild(renderer.domElement)

  // 创建圆柱体
  const radius = 800
  const height = 1400
  const geometry = new THREE.CylinderGeometry(radius, radius, height, 36, 5, true)
  const material = new THREE.MeshPhongMaterial({
    color: 0xffff00,
    specular: 0x111111,
    shininess: 30,
    wireframe: false
  })
  cylinder = new THREE.Mesh(geometry, material)
  cylinder.position.set(0, 0, 0)

  // 创建红色球体
  const sphereRadius = radius * 0.5
  const sphereGeometry = new THREE.SphereGeometry(sphereRadius, 32, 32)
  const sphereMaterial = new THREE.MeshPhongMaterial({
    color: 0xff0000,
    specular: 0x111111,
    shininess: 30
  })
  const sphere = new THREE.Mesh(sphereGeometry, sphereMaterial)
  // 将球体放置在圆柱体顶部，球体中心点位于圆柱体顶部上方 sphereRadius 的距离
  sphere.position.set(0, height / 2 + sphereRadius, 0)
  cylinder.add(sphere)

  // 创建自定义网格线
  const gridLines = new THREE.Group()

  const lineCount = 7
  // const gridRows = [2, 4, 8, 10, 12, 36, 36] // 每行的网格数量
  const gridRows = [36, 36, 12, 10, 8, 4, 2] // 每行的网格数量

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
  const gridSize = radius * 2 * Math.PI / 36 // 网格宽度
  const gridHeight = height / lineCount // 网格高度

  // 创建5行36列的网格
  for (let row = 0; row < lineCount; row++) {

    const colsInRow = gridRows[row]

    for (let col = 0; col < colsInRow; col++) {
      const gridGeometry = new THREE.PlaneGeometry(gridSize, gridHeight)
      const gridMesh = new THREE.Mesh(gridGeometry, gridMaterial.clone())

      // 计算网格位置，稍微调整以避免与分割线重叠
      //   const angle = (col * Math.PI * 2) / 36
      const angle = ((col + (col + 1)) * Math.PI) / colsInRow
      const x = radius * Math.cos(angle)
      const z = radius * Math.sin(angle)
      const y = -height / 2 + gridHeight / 2 + row * gridHeight
      gridMesh.position.set(x, y, z)

      // 使网格面向圆柱体中心，然后旋转使其背对中轴线
      gridMesh.lookAt(0, y, 0)
      gridMesh.rotateY(Math.PI)

      // 应用文字贴图
      const textTexture = createTextTexture(row, col)
      gridMesh.material.map = textTexture

      cylinder.add(gridMesh)
    }
  }

  // 创建顶部和底部的圆形
  const circleGeometry = new THREE.CircleGeometry(radius, 32) // 使用相同的半径
  const circleMaterial = new THREE.MeshPhongMaterial({
    color: 0xff69b4, // 粉色
    side: THREE.DoubleSide
  })

  const topCircle = new THREE.Mesh(circleGeometry, circleMaterial)
  topCircle.position.y = height / 2
  topCircle.rotation.x = -Math.PI / 2

  const bottomCircle = new THREE.Mesh(circleGeometry, circleMaterial)
  bottomCircle.position.y = -height / 2
  bottomCircle.rotation.x = Math.PI / 2

  // 添加光源
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
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
  controls.enablePan = false     // 保持禁用平移
  controls.enableZoom = true     // 启用缩放
  controls.autoRotate = false    // 初始状态不自动旋转
  controls.autoRotateSpeed = -2.0 // 设置自动旋转速度
  controls.target.set(0, 0, 0)   // 设置旋转中心点为圆柱体中心
  controls.minPolarAngle = 0     // 限制垂直旋转角度
  controls.maxPolarAngle = Math.PI // 限制垂直旋转角度
  controls.minDistance = 400     // 设置最小缩放距离
  controls.maxDistance = 2500    // 设置最大缩放距离
  controls.zoomSpeed = 1.0       // 设置缩放速度

  // 开始动画循环
  animate()
}

const animate = () => {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

const handleResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}

onMounted(() => {
  init()
  window.addEventListener('resize', handleResize)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  if (renderer) {
    renderer.dispose()
  }
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

.control-btn {
  position: absolute;
  top: 20px;
  right: 20px;
  padding: 10px 20px;
  font-size: 16px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
  z-index: 1000;
}

.control-btn:hover {
  background-color: #45a049;
}

.control-btn:active {
  background-color: #3d8b40;
}
</style>