<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import img1Url from '../assets/showimg/1.jpg';
import img2Url from '../assets/showimg/2.jpg';
import img3Url from '../assets/showimg/3.jpg';
import img4Url from '../assets/showimg/4.jpg';
import woodUrl from '../assets/showimg/木制.jpg';
import floorUrl from '../assets/showimg/southeast.jpg';
import topUrl from '../assets/top.jpg';

const canvasRef = ref(null);
const viewingStand = ref(false);
const isReturning = ref(false);
const showIntro = ref(true);
const introFading = ref(false);
let renderer, animationId;

function returnFromFocus() {
  isReturning.value = true;
}

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

  // 图片面板（MeshBasicMaterial 不受光照影响，原色显示）
  const imgPanel = new THREE.Mesh(
    new THREE.PlaneGeometry(3.87, 4.54),
    new THREE.MeshBasicMaterial({ map: texLoader.load(imgUrl) })
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
  const spot = new THREE.SpotLight(0xffffff, 12, 25, Math.PI / 6, 0.3);
  spot.position.set(x + frontX * 3.5, 8.5, z + frontZ * 3.5);
  spot.target.position.set(x, 4.95, z);
  scene.add(spot);
  scene.add(spot.target);

  // 展牌脚下远距补光（温暖点光源）
  const fill = new THREE.PointLight(0xffe8c0, 3.0, 10);
  fill.position.set(x + frontX * 2.0, 1.2, z + frontZ * 2.0);
  scene.add(fill);

  return group;
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
  // 每面墙单独创建材质，按实际尺寸设置纹理重复，避免拉伸
  const TILE = 3; // 每 3 个单位重复一次纹理
  const wallTexLoader = new THREE.TextureLoader();
  function makeWallMat(w, h) {
    const tex = wallTexLoader.load(woodUrl);
    tex.wrapS = THREE.RepeatWrapping;
    tex.wrapT = THREE.RepeatWrapping;
    tex.repeat.set(w / TILE, h / TILE);
    return new THREE.MeshLambertMaterial({ map: tex, side: THREE.FrontSide });
  }
  const floorTex = wallTexLoader.load(floorUrl);
  floorTex.wrapS = THREE.RepeatWrapping;
  floorTex.wrapT = THREE.RepeatWrapping;
  floorTex.repeat.set(ROOM_W / TILE, ROOM_D / TILE);
  const floorMat = new THREE.MeshLambertMaterial({ map: floorTex });
  const ceilingTex = wallTexLoader.load(topUrl);
  ceilingTex.wrapS = THREE.RepeatWrapping;
  ceilingTex.wrapT = THREE.RepeatWrapping;
  ceilingTex.repeat.set(ROOM_W / TILE, ROOM_D / TILE);
  const ceilingMat = new THREE.MeshLambertMaterial({ map: ceilingTex });

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

  // 四面墙（每面独立材质）
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
    const m = new THREE.Mesh(new THREE.PlaneGeometry(w, h), makeWallMat(w, h));
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
  const stands = [
    { img: img1Url, x: -8, z: -2, ry: 0 },
    { img: img2Url, x: 8, z: -2, ry: 0 },
    { img: img3Url, x: -8, z: -14, ry: 0 },
    { img: img4Url, x: 8, z: -14, ry: 0 },
  ].map(({ img, x, z, ry }) => {
    const group = createStand(scene, texLoader, img, x, z, ry);
    const fx = Math.sin(ry),
      fz = Math.cos(ry);
    return {
      group,
      focusPos: new THREE.Vector3(x + fx * 5, 4.0, z + fz * 5),
      focusLook: new THREE.Vector3(x, 4.95, z),
    };
  });
  // hover / 点击检测
  const raycaster = new THREE.Raycaster();
  const mouseNDC = new THREE.Vector2(9999, 9999);
  let hoveredStand = null;
  // 焦点模式相机目标
  const focusCamPos = new THREE.Vector3();
  const focusLookPt = new THREE.Vector3();
  const currentLook = new THREE.Vector3(0, 2, -1);

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

  // 鼠标左键拖拽旋转视角 + 鼠标位置用于 hover 检测
  const canvas = canvasRef.value;
  let isDragging = false;
  let lastMouseX = 0,
    lastMouseY = 0;
  let mouseDownX = 0,
    mouseDownY = 0,
    didDrag = false;

  const onMouseDown = (e) => {
    if (e.button === 0) {
      isDragging = true;
      lastMouseX = e.clientX;
      lastMouseY = e.clientY;
      mouseDownX = e.clientX;
      mouseDownY = e.clientY;
      didDrag = false;
    }
  };
  const onMouseUp = (e) => {
    if (e.button === 0 && !didDrag && !viewingStand.value) {
      // 纯点击：检测展牌
      raycaster.setFromCamera(mouseNDC, camera);
      const meshes = stands.flatMap((s) => {
        const a = [];
        s.group.traverse((c) => {
          if (c.isMesh) a.push(c);
        });
        return a;
      });
      const hits = raycaster.intersectObjects(meshes);
      if (hits.length > 0) {
        const hitObj = hits[0].object;
        const hit = stands.find((s) => {
          let o = hitObj;
          while (o) {
            if (o === s.group) return true;
            o = o.parent;
          }
          return false;
        });
        if (hit) {
          focusCamPos.copy(hit.focusPos);
          focusLookPt.copy(hit.focusLook);
          currentLook
            .copy(camera.position)
            .add(
              new THREE.Vector3(
                -Math.sin(yaw) * Math.cos(pitch),
                Math.sin(pitch),
                -Math.cos(yaw) * Math.cos(pitch)
              )
            );
          viewingStand.value = true;
        }
      }
    }
    isDragging = false;
  };
  const onMouseMove = (e) => {
    mouseNDC.x = (e.clientX / window.innerWidth) * 2 - 1;
    mouseNDC.y = -(e.clientY / window.innerHeight) * 2 + 1;
    if (!isDragging) return;
    const dx = e.clientX - mouseDownX,
      dy = e.clientY - mouseDownY;
    if (Math.sqrt(dx * dx + dy * dy) > 4) didDrag = true;
    if (!viewingStand.value) {
      yaw -= (e.clientX - lastMouseX) * 0.004;
      pitch -= (e.clientY - lastMouseY) * 0.004;
      pitch = Math.max(
        -Math.PI / 2 + 0.05,
        Math.min(Math.PI / 2 - 0.05, pitch)
      );
    }
    lastMouseX = e.clientX;
    lastMouseY = e.clientY;
  };
  canvas.addEventListener('mousedown', onMouseDown);
  window.addEventListener('mouseup', onMouseUp);
  window.addEventListener('mousemove', onMouseMove);

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

    // 相机更新
    if (viewingStand.value) {
      if (isReturning.value) {
        // 平滑归位回玩家视角
        const targetPos = new THREE.Vector3(
          playerPos.x,
          EYE_HEIGHT,
          playerPos.z
        );
        const returnLook = new THREE.Vector3(
          -Math.sin(yaw) * Math.cos(pitch),
          Math.sin(pitch),
          -Math.cos(yaw) * Math.cos(pitch)
        ).add(targetPos);
        camera.position.lerp(targetPos, 0.07);
        currentLook.lerp(returnLook, 0.07);
        camera.lookAt(currentLook);
        if (camera.position.distanceTo(targetPos) < 0.08) {
          isReturning.value = false;
          viewingStand.value = false;
        }
      } else {
        // 焦点模式：平滑移向展牌正前方
        camera.position.lerp(focusCamPos, 0.06);
        currentLook.lerp(focusLookPt, 0.06);
        camera.lookAt(currentLook);
      }
    } else {
      // 第一人称
      camera.position.set(playerPos.x, EYE_HEIGHT, playerPos.z);
      const lookDir = new THREE.Vector3(
        -Math.sin(yaw) * Math.cos(pitch),
        Math.sin(pitch),
        -Math.cos(yaw) * Math.cos(pitch)
      );
      currentLook.copy(camera.position).add(lookDir);
      camera.lookAt(currentLook);
    }

    // ── 展牌 hover 检测与动画 ──
    raycaster.setFromCamera(mouseNDC, camera);
    const allMeshes = stands.flatMap((s) => {
      const arr = [];
      s.group.traverse((child) => {
        if (child.isMesh) arr.push(child);
      });
      return arr;
    });
    const hits = raycaster.intersectObjects(allMeshes);
    let newHovered = null;
    if (hits.length > 0) {
      const hitObj = hits[0].object;
      newHovered = stands.find((s) => {
        let o = hitObj;
        while (o) {
          if (o === s.group) return true;
          o = o.parent;
        }
        return false;
      });
    }
    stands.forEach((s) => {
      const hovered = s === newHovered;
      const ts = hovered ? 1.08 : 1.0;
      s.group.scale.x += (ts - s.group.scale.x) * 0.1;
      s.group.scale.y += (ts - s.group.scale.y) * 0.1;
      s.group.scale.z += (ts - s.group.scale.z) * 0.1;
    });
    hoveredStand = newHovered;
    canvas.style.cursor = newHovered ? 'pointer' : 'default';

    renderer.render(scene, camera);
  };
  animate();

  // 启动动画：2s后开始淡出，再经 1s 过渡后移除
  setTimeout(() => {
    introFading.value = true;
  }, 2000);
  setTimeout(() => {
    showIntro.value = false;
  }, 3000);

  onBeforeUnmount(() => {
    window.removeEventListener('resize', onResize);
    window.removeEventListener('keydown', onKeyDown);
    window.removeEventListener('keyup', onKeyUp);
    canvas.removeEventListener('mousedown', onMouseDown);
    window.removeEventListener('mouseup', onMouseUp);
    window.removeEventListener('mousemove', onMouseMove);
    cancelAnimationFrame(animationId);
    renderer.dispose();
  });
});
</script>

<template>
  <canvas ref="canvasRef" class="three-canvas" />

  <Transition name="intro">
    <div
      v-if="showIntro"
      class="intro-overlay"
      :class="{ fading: introFading }"
    >
      <div class="intro-title">苏绣博物馆</div>
      <div class="intro-sub">周庄展览·斋甲天下</div>
    </div>
  </Transition>
  <button
    v-if="viewingStand && !isReturning"
    class="return-btn"
    @click="returnFromFocus"
  >
    ← 返回
  </button>
  <div v-if="!viewingStand" class="hint">
    WASD 移动 &nbsp;·&nbsp; 鼠标左键拖拽 旋转视角 &nbsp;·&nbsp; 点击展牌
    聚焦视角
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

.return-btn {
  position: fixed;
  bottom: 36px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(20, 12, 4, 0.88);
  color: #f0d890;
  border: 1.5px solid #b8860b;
  padding: 11px 32px;
  border-radius: 26px;
  font-size: 15px;
  letter-spacing: 2px;
  cursor: pointer;
  transition:
    background 0.2s,
    transform 0.15s;
  white-space: nowrap;
}
.return-btn:hover {
  background: rgba(60, 38, 8, 0.96);
  transform: translateX(-50%) scale(1.04);
}

/* 启动动画 */
.intro-overlay {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: #0a0600;
  transition: opacity 1s ease;
  opacity: 1;
}
.intro-overlay.fading {
  opacity: 0;
}
.intro-title {
  font-size: clamp(42px, 8vw, 88px);
  font-family: 'STKaiti', '楷体', 'serif';
  color: #f0d890;
  letter-spacing: 0.25em;
  text-shadow:
    0 0 40px rgba(200, 150, 0, 0.7),
    0 2px 8px rgba(0, 0, 0, 0.8);
  animation: titleIn 1.2s cubic-bezier(0.16, 1, 0.3, 1) both;
}
.intro-sub {
  margin-top: 18px;
  font-size: clamp(14px, 2vw, 20px);
  font-family: 'STKaiti', '楷体', serif;
  color: #c8a060;
  letter-spacing: 0.4em;
  opacity: 0.85;
  animation: titleIn 1.6s 0.3s cubic-bezier(0.16, 1, 0.3, 1) both;
}
@keyframes titleIn {
  from {
    opacity: 0;
    transform: translateY(24px) scale(0.96);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}
</style>
