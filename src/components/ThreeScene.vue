<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import pxUrl from '../assets/px.png';
import nxUrl from '../assets/nx.png';
import pyUrl from '../assets/py.png';
import nyUrl from '../assets/ny.png';
import pzUrl from '../assets/pz.png';
import nzUrl from '../assets/nz.png';

const canvasRef = ref(null);
let renderer, animationId;

onMounted(() => {
  const width = window.innerWidth;
  const height = window.innerHeight;

  // 场景
  const scene = new THREE.Scene();

  // 相机 —— 偏离原点，避免与 target 重合导致 OrbitControls 旋转失效
  const camera = new THREE.PerspectiveCamera(110, width / height, 1, 2000000);
  camera.position.set(0, 0, 100);

  // 渲染器
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.setSize(width, height);
  renderer.setPixelRatio(window.devicePixelRatio);

  // 全景拖拽控制
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.target.set(0, 0, 0);
  controls.enableZoom = true;
  controls.enablePan = true;
  controls.panSpeed = 500;
  controls.zoomSpeed = 300;
  controls.rotateSpeed = 0.5;
  controls.minDistance = 10;
  controls.maxDistance = 150000;
  controls.minPolarAngle = 0.05;
  controls.maxPolarAngle = Math.PI - 0.05;

  // 超大物理空间盒（500000 单位边长），内壁贴图，有真实体积感
  const texLoader = new THREE.TextureLoader();
  const skyMaterials = [
    new THREE.MeshBasicMaterial({
      map: texLoader.load(pxUrl),
      side: THREE.BackSide,
    }), // px 右
    new THREE.MeshBasicMaterial({
      map: texLoader.load(nxUrl),
      side: THREE.BackSide,
    }), // nx 左
    new THREE.MeshBasicMaterial({
      map: texLoader.load(pyUrl),
      side: THREE.BackSide,
    }), // py 上
    new THREE.MeshBasicMaterial({
      map: texLoader.load(nyUrl),
      side: THREE.BackSide,
    }), // ny 下
    new THREE.MeshBasicMaterial({
      map: texLoader.load(pzUrl),
      side: THREE.BackSide,
    }), // pz 前
    new THREE.MeshBasicMaterial({
      map: texLoader.load(nzUrl),
      side: THREE.BackSide,
    }), // nz 后
  ];
  const skyBox = new THREE.Mesh(
    new THREE.BoxGeometry(500000, 500000, 500000),
    skyMaterials
  );
  scene.add(skyBox);

  // 响应式窗口
  const onResize = () => {
    const w = window.innerWidth;
    const h = window.innerHeight;
    camera.aspect = w / h;
    camera.updateProjectionMatrix();
    renderer.setSize(w, h);
  };
  window.addEventListener('resize', onResize);

  // 动画循环
  const animate = () => {
    animationId = requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
  };
  animate();

  // 清理
  onBeforeUnmount(() => {
    window.removeEventListener('resize', onResize);
    cancelAnimationFrame(animationId);
    controls.dispose();
    renderer.dispose();
  });
});
</script>

<template>
  <canvas ref="canvasRef" class="three-canvas" />
</template>

<style scoped>
.three-canvas {
  display: block;
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
}
</style>
