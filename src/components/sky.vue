<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import img1Url from '../assets/showimg/1.jpg';
import img2Url from '../assets/showimg/2.jpg';
import img3Url from '../assets/showimg/3.jpg';
import img4Url from '../assets/showimg/4.jpg';

const canvasRef = ref(null);
let renderer, animationId;

// 创建立体展牌：底座 + 支柱 + 画框 + 图片面板 + 金色铭牌
function createStand(scene, texLoader, imgUrl, x, z, ry) {
  const woodMat = new THREE.MeshLambertMaterial({ color: 0x4a2510 });
  const goldMat = new THREE.MeshLambertMaterial({ color: 0xb8860b });
  const group = new THREE.Group();

  // 底座
  const base = new THREE.Mesh(new THREE.BoxGeometry(4.8, 0.3, 1.5), woodMat);
  base.position.y = 0.15;
  base.castShadow = true;
  group.add(base);

  // 中央支柱
  const pillar = new THREE.Mesh(new THREE.BoxGeometry(0.3, 2.8, 0.3), woodMat);
  pillar.position.y = 1.7;
  pillar.castShadow = true;
  group.add(pillar);

  // 画框（深木色立方体）
  const frame = new THREE.Mesh(new THREE.BoxGeometry(4.35, 5.0, 0.21), woodMat);
  frame.position.y = 4.95;
  frame.castShadow = true;
  group.add(frame);

  // 图片面板（贴在画框正面）
  const imgPanel = new THREE.Mesh(
    new THREE.PlaneGeometry(3.87, 4.54),
    new THREE.MeshLambertMaterial({ map: texLoader.load(imgUrl) })
  );
  imgPanel.position.set(0, 4.95, 0.123);
  group.add(imgPanel);

  // 金色铭牌
  const plaque = new THREE.Mesh(
    new THREE.BoxGeometry(2.4, 0.42, 0.11),
    goldMat
  );
  plaque.position.set(0, 2.55, 0.14);
  group.add(plaque);

  group.position.set(x, 0, z);
  group.rotation.y = ry;
  scene.add(group);

  // 展牌聚光灯（从正面斜上方打向画框）
  const frontX = Math.sin(ry),
    frontZ = Math.cos(ry);
  const spot = new THREE.SpotLight(0xfff8e0, 6.0, 20, Math.PI / 7, 0.4);
  spot.position.set(x + frontX * 3.5, 8.5, z + frontZ * 3.5);
  spot.target.position.set(x, 4.95, z);
  scene.add(spot);
  scene.add(spot.target);

  // 展牌脚下远距补光（温暖点光源）
  const fill = new THREE.PointLight(0xffe8c0, 1.8, 8);
  fill.position.set(x + frontX * 2.0, 1.2, z + frontZ * 2.0);
  scene.add(fill);
}

onMounted(() => {
  const width = window.innerWidth;
  const height = window.innerHeight;

  // 场景
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);

  // 博物馆尺寸
  const ROOM_W = 50,
    ROOM_D = 40,
    ROOM_H = 9;

  // 相机
  const camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);

  // 渲染器
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.setSize(width, height);
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;

  // ── 博物馆建筑 ──
  const wallMat = new THREE.MeshLambertMaterial({
    color: 0xf2ede2,
    side: THREE.FrontSide,
  });
  const floorMat = new THREE.MeshLambertMaterial({ color: 0xddd5c4 });
  const ceilingMat = new THREE.MeshLambertMaterial({ color: 0xfafafa });

  // 地面
  const floor = new THREE.Mesh(
    new THREE.PlaneGeometry(ROOM_W, ROOM_D),
    floorMat
  );
  floor.rotation.x = -Math.PI / 2;
  floor.receiveShadow = true;
  scene.add(floor);

  // 天花板
  const ceiling = new THREE.Mesh(
    new THREE.PlaneGeometry(ROOM_W, ROOM_D),
    ceilingMat
  );
  ceiling.rotation.x = Math.PI / 2;
  ceiling.position.y = ROOM_H;
  scene.add(ceiling);

  // 四面墙
  [
    { w: ROOM_W, h: ROOM_H, pos: [0, ROOM_H / 2, -ROOM_D / 2], ry: 0 },
    { w: ROOM_W, h: ROOM_H, pos: [0, ROOM_H / 2, ROOM_D / 2], ry: Math.PI },
    {
      w: ROOM_D,
      h: ROOM_H,
      pos: [ROOM_W / 2, ROOM_H / 2, 0],
      ry: -Math.PI / 2,
    },
    {
      w: ROOM_D,
      h: ROOM_H,
      pos: [-ROOM_W / 2, ROOM_H / 2, 0],
      ry: Math.PI / 2,
    },
  ].forEach(({ w, h, pos, ry }) => {
    const m = new THREE.Mesh(new THREE.PlaneGeometry(w, h), wallMat);
    m.position.set(...pos);
    m.rotation.y = ry;
    scene.add(m);
  });

  // 踢脚线
  const baseboardMat = new THREE.MeshLambertMaterial({ color: 0xbdb0a0 });
  [
    { size: [ROOM_W, 0.25, 0.05], pos: [0, 0.125, -ROOM_D / 2 + 0.03], ry: 0 },
    { size: [ROOM_W, 0.25, 0.05], pos: [0, 0.125, ROOM_D / 2 - 0.03], ry: 0 },
    {
      size: [ROOM_D, 0.25, 0.05],
      pos: [ROOM_W / 2 - 0.03, 0.125, 0],
      ry: Math.PI / 2,
    },
    {
      size: [ROOM_D, 0.25, 0.05],
      pos: [-ROOM_W / 2 + 0.03, 0.125, 0],
      ry: Math.PI / 2,
    },
  ].forEach(({ size, pos, ry }) => {
    const m = new THREE.Mesh(new THREE.BoxGeometry(...size), baseboardMat);
    m.position.set(...pos);
    m.rotation.y = ry;
    scene.add(m);
  });

  // ── 光照 ──
  scene.add(new THREE.AmbientLight(0xfff8f0, 1.2));

  // 天花板灯具（4盏）
  [
    [-12, -10],
    [12, -10],
    [-12, 10],
    [12, 10],
  ].forEach(([x, z]) => {
    const pl = new THREE.PointLight(0xfff5e0, 2.5, 60);
    pl.position.set(x, ROOM_H - 0.3, z);
    scene.add(pl);
    const lamp = new THREE.Mesh(
      new THREE.CylinderGeometry(0.3, 0.5, 0.4, 16),
      new THREE.MeshLambertMaterial({
        color: 0xc8a000,
        emissive: 0xffeeaa,
        emissiveIntensity: 0.6,
      })
    );
    lamp.position.set(x, ROOM_H - 0.2, z);
    scene.add(lamp);
  });

  // ── 四个立体展牌（两排正对玩家，ry=0 画面朝 +Z 方向）──
  const texLoader = new THREE.TextureLoader();
  // 前排（靠近玩家出发点）
  createStand(scene, texLoader, img1Url, -8, -2, 0);
  createStand(scene, texLoader, img2Url, 8, -2, 0);
  // 后排（靠近北墙）
  createStand(scene, texLoader, img3Url, -8, -14, 0);
  createStand(scene, texLoader, img4Url, 8, -14, 0);

  // ── 第一人称控制参数 ──
  const EYE_HEIGHT = 1.7;
  const MOVE_SPEED = 0.08;
  const BOUND_X = ROOM_W / 2 - 1;
  const BOUND_Z = ROOM_D / 2 - 1;

  const playerPos = new THREE.Vector3(0, 0, 8);

  let yaw = 0; // 水平视角
  let pitch = 0; // 垂直视角（俯仰）

  // 键盘
  const keys = {};
  const onKeyDown = (e) => {
    keys[e.code] = true;
  };
  const onKeyUp = (e) => {
    keys[e.code] = false;
  };
  window.addEventListener('keydown', onKeyDown);
  window.addEventListener('keyup', onKeyUp);

  // 指针锁定：点击画布后锁定鼠标，移动鼠标控制视角
  const canvas = canvasRef.value;
  const onClick = () => {
    canvas.requestPointerLock();
  };
  canvas.addEventListener('click', onClick);

  const onMouseMove = (e) => {
    if (document.pointerLockElement !== canvas) return;
    yaw -= e.movementX * 0.002;
    pitch -= e.movementY * 0.002;
    pitch = Math.max(-Math.PI / 2 + 0.05, Math.min(Math.PI / 2 - 0.05, pitch));
  };
  document.addEventListener('mousemove', onMouseMove);

  // 响应窗口大小
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

    // 水平移动方向（基于 yaw，忽略 pitch）
    const forward = new THREE.Vector3(-Math.sin(yaw), 0, -Math.cos(yaw));
    const right = new THREE.Vector3(Math.cos(yaw), 0, -Math.sin(yaw));

    if (keys['KeyW'] || keys['ArrowUp'])
      playerPos.addScaledVector(forward, MOVE_SPEED);
    if (keys['KeyS'] || keys['ArrowDown'])
      playerPos.addScaledVector(forward, -MOVE_SPEED);
    if (keys['KeyA'] || keys['ArrowLeft'])
      playerPos.addScaledVector(right, -MOVE_SPEED);
    if (keys['KeyD'] || keys['ArrowRight'])
      playerPos.addScaledVector(right, MOVE_SPEED);

    // 边界限制（防止穿墙）
    playerPos.x = Math.max(-BOUND_X, Math.min(BOUND_X, playerPos.x));
    playerPos.z = Math.max(-BOUND_Z, Math.min(BOUND_Z, playerPos.z));

    // 相机置于眼睛位置
    camera.position.set(playerPos.x, EYE_HEIGHT, playerPos.z);

    // 根据 yaw + pitch 计算朝向
    const lookDir = new THREE.Vector3(
      -Math.sin(yaw) * Math.cos(pitch),
      Math.sin(pitch),
      -Math.cos(yaw) * Math.cos(pitch)
    );
    camera.lookAt(camera.position.clone().add(lookDir));

    renderer.render(scene, camera);
  };
  animate();

  onBeforeUnmount(() => {
    window.removeEventListener('resize', onResize);
    window.removeEventListener('keydown', onKeyDown);
    window.removeEventListener('keyup', onKeyUp);
    canvas.removeEventListener('click', onClick);
    document.removeEventListener('mousemove', onMouseMove);
    cancelAnimationFrame(animationId);
    renderer.dispose();
  });
});
</script>

<template>
  <canvas ref="canvasRef" class="three-canvas" />
  <div class="hint">
    点击画面锁定鼠标 &nbsp;·&nbsp; WASD 移动 &nbsp;·&nbsp; 鼠标 旋转视角
    &nbsp;·&nbsp; ESC 解锁
  </div>
</template>

<style scoped>
.three-canvas {
  display: block;
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
}

.hint {
  position: fixed;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.45);
  color: #fff;
  padding: 7px 18px;
  border-radius: 20px;
  font-size: 13px;
  pointer-events: none;
  white-space: nowrap;
}
</style>
