<template>
  <div ref="container" class="cylinder-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'

const container = ref(null)
let scene, camera, renderer, cylinder, controls

const init = () => {
  // 创建场景
  scene = new THREE.Scene()
  
  // 创建相机
  camera = new THREE.PerspectiveCamera(
    45,
    window.innerWidth / window.innerHeight,
    0.1,
    2000
  )
  camera.position.set(800, 800, 800)
  camera.lookAt(0, 0, 0)
  
  // 创建渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setClearColor(0xf5f5dc)
  container.value.appendChild(renderer.domElement)
  
  // 创建圆柱体
  const radius = 250 // 增加半径到250
  const geometry = new THREE.CylinderGeometry(radius, radius, 1000, 32)
  const material = new THREE.MeshPhongMaterial({
    color: 0xffff00, // 黄色侧面
    specular: 0x111111,
    shininess: 30
  })
  cylinder = new THREE.Mesh(geometry, material)
  cylinder.position.set(0, 0, 0) // 确保圆柱体在中心位置
  
  // 创建顶部和底部的圆形
  const circleGeometry = new THREE.CircleGeometry(radius, 32) // 使用相同的半径
  const circleMaterial = new THREE.MeshPhongMaterial({
    color: 0xff69b4, // 粉色
    side: THREE.DoubleSide
  })
  
  const topCircle = new THREE.Mesh(circleGeometry, circleMaterial)
  topCircle.position.y = 500
  topCircle.rotation.x = -Math.PI / 2
  
  const bottomCircle = new THREE.Mesh(circleGeometry, circleMaterial)
  bottomCircle.position.y = -500
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
  controls.enableZoom = false    // 保持禁用缩放
  controls.autoRotate = false    // 保持禁用自动旋转
  controls.target.set(0, 0, 0)   // 设置旋转中心点为圆柱体中心
  controls.minPolarAngle = 0     // 限制垂直旋转角度
  controls.maxPolarAngle = Math.PI // 限制垂直旋转角度
  
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
.cylinder-container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
}
</style> 