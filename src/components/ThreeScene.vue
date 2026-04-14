<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'

const canvasRef = ref(null)
let renderer, animationId

onMounted(() => {
  const width = canvasRef.value.clientWidth
  const height = canvasRef.value.clientHeight

  // 场景
  const scene = new THREE.Scene()
  scene.background = new THREE.Color(0x1a1a2e)

  // 相机
  const camera = new THREE.PerspectiveCamera(60, width / height, 0.1, 1000)
  camera.position.set(0, 1.5, 5)

  // 渲染器
  renderer = new THREE.WebGLRenderer({ canvas: canvasRef.value, antialias: true })
  renderer.setSize(width, height)
  renderer.setPixelRatio(window.devicePixelRatio)
  renderer.shadowMap.enabled = true

  // 环境光
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4)
  scene.add(ambientLight)

  // 点光源
  const pointLight = new THREE.PointLight(0x00d4ff, 120, 20)
  pointLight.position.set(3, 4, 3)
  pointLight.castShadow = true
  scene.add(pointLight)

  // 地面
  const planeGeo = new THREE.PlaneGeometry(12, 12)
  const planeMat = new THREE.MeshStandardMaterial({ color: 0x16213e })
  const plane = new THREE.Mesh(planeGeo, planeMat)
  plane.rotation.x = -Math.PI / 2
  plane.receiveShadow = true
  scene.add(plane)

  // 中心立方体
  const boxGeo = new THREE.BoxGeometry(1.2, 1.2, 1.2)
  const boxMat = new THREE.MeshStandardMaterial({ color: 0x0f3460, metalness: 0.5, roughness: 0.3 })
  const box = new THREE.Mesh(boxGeo, boxMat)
  box.position.set(0, 0.6, 0)
  box.castShadow = true
  scene.add(box)

  // 球体
  const sphereGeo = new THREE.SphereGeometry(0.6, 32, 32)
  const sphereMat = new THREE.MeshStandardMaterial({ color: 0xe94560, metalness: 0.3, roughness: 0.4 })
  const sphere = new THREE.Mesh(sphereGeo, sphereMat)
  sphere.position.set(2.2, 0.6, 0)
  sphere.castShadow = true
  scene.add(sphere)

  // 圆环
  const torusGeo = new THREE.TorusGeometry(0.5, 0.18, 16, 60)
  const torusMat = new THREE.MeshStandardMaterial({ color: 0x00d4ff, metalness: 0.6, roughness: 0.2 })
  const torus = new THREE.Mesh(torusGeo, torusMat)
  torus.position.set(-2.2, 0.7, 0)
  torus.castShadow = true
  scene.add(torus)

  // 网格辅助线
  const gridHelper = new THREE.GridHelper(12, 12, 0x333355, 0x222244)
  scene.add(gridHelper)

  // 响应式窗口大小
  const onResize = () => {
    const w = canvasRef.value.clientWidth
    const h = canvasRef.value.clientHeight
    camera.aspect = w / h
    camera.updateProjectionMatrix()
    renderer.setSize(w, h)
  }
  window.addEventListener('resize', onResize)

  // 动画循环
  const clock = new THREE.Clock()
  const animate = () => {
    animationId = requestAnimationFrame(animate)
    const t = clock.getElapsedTime()

    box.rotation.x = t * 0.6
    box.rotation.y = t * 0.8

    sphere.position.y = 0.6 + Math.sin(t * 1.5) * 0.4

    torus.rotation.x = t * 0.5
    torus.rotation.z = t * 0.7

    pointLight.position.x = Math.sin(t * 0.8) * 4
    pointLight.position.z = Math.cos(t * 0.8) * 4

    renderer.render(scene, camera)
  }
  animate()

  // 清理
  onBeforeUnmount(() => {
    window.removeEventListener('resize', onResize)
    cancelAnimationFrame(animationId)
    renderer.dispose()
  })
})
</script>

<template>
  <canvas ref="canvasRef" class="three-canvas" />
</template>

<style scoped>
.three-canvas {
  display: block;
  width: 100%;
  height: 100vh;
}
</style>
