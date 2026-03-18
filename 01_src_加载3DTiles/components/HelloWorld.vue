<script setup>
import { ref,nextTick } from "vue"
import * as THREE from "three"
import { TilesRenderer } from "3d-tiles-renderer"
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js"
import { HDRLoader } from "three/examples/jsm/Addons.js"
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
import { playerController } from "three-player-controller"

const container = ref(null)

let scene
let camera
let renderer
let tilesRenderer
let controls
let player
let initPos

init();

async function init() {
  // 等待 DOM 挂载
  await nextTick()
  
  scene = new THREE.Scene()
  
  // 加载 HDR - 使用 RGBELoader 替代 HDRLoader
  const rgbeLoader = new RGBELoader()
  const envMap = await rgbeLoader.loadAsync('./hdr/indoor.hdr')
  envMap.mapping = THREE.EquirectangularReflectionMapping
  scene.environment = envMap
  scene.background = envMap

  camera = new THREE.PerspectiveCamera(
    60,
    container.value.clientWidth / container.value.clientHeight,
    0.1,
    100000
  )
  camera.position.set(0, 0, 0)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(
    container.value.clientWidth,
    container.value.clientHeight
  )
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.0

  container.value.appendChild(renderer.domElement)
  controls = new OrbitControls(camera, renderer.domElement)
  // 3DTiles
  tilesRenderer = new TilesRenderer("./models/out/tileset.json")

  // 注册 gltf loader
  tilesRenderer.manager.addHandler(
    /\.gltf$/,
    new GLTFLoader()
  )

  // 更新矩阵并设置相机位置
  let loadedTileSetHandled = false
  // tiles 加载完成
  tilesRenderer.addEventListener("load-root-tileset", () => {
    if (loadedTileSetHandled) return;
    loadedTileSetHandled = true;

    const sphere = new THREE.Sphere();
    tilesRenderer.getBoundingSphere(sphere);
    const center = sphere.center.clone(); // 获取包围球中心
    const radius = sphere.radius; // 获取包围球半径
    console.log("包围球中心：",center)
    controls.target.copy(center); // 把控制器目标设为包围球中心 

    const offset = new THREE.Vector3(radius * 2, radius, 0); // 给相机一个偏移
    initPos = center.clone().add(offset)
    camera.position.copy(center).add(offset); // 设置相机位置

    const m = (tilesRenderer).root.transform; // 获取原始矩阵
    const rotationMat3 = new THREE.Matrix3().set(m[0], m[1], m[2], m[4], m[5], m[6], m[8], m[9], m[10]); // 取出旋转部分
    rotationMat3.transpose(); // 逆旋转
    const rotationMat4 = new THREE.Matrix4().setFromMatrix3(rotationMat3); // 转回Matrix4以便应用
    const rotX90 = new THREE.Matrix4().makeRotationX((90 * Math.PI) / 180); // x轴旋转90度矩阵
    rotationMat4.multiply(rotX90); // 合并矩阵（由z轴向上坐标系 转为 y轴向上坐标系）
    const translationMatrix1 = new THREE.Matrix4().makeTranslation(center.x, center.y, center.z); // T(center)
    const translationMatrix2 = new THREE.Matrix4().makeTranslation(-center.x, -center.y, -center.z); // T(-center)
    const finalMatrix = new THREE.Matrix4().multiplyMatrices(translationMatrix1, rotationMat4).multiply(translationMatrix2); // 最终矩阵 = T(center) * R⁻¹ * T(-center)

    tilesRenderer.group.matrix.copy(finalMatrix); // 设置矩阵
    tilesRenderer.group.matrixAutoUpdate = false; // 禁止自动更新矩阵
    tilesRenderer.group.updateMatrixWorld(true); // 更新矩阵
  })

  // tilesRenderer.addEventListener("load-root-tileset", () => {
  //   const sphere = new THREE.Sphere();
  //   tilesRenderer.getBoundingSphere(sphere);

  //   const center = sphere.center.clone();
  //   const radius = sphere.radius;

  //   console.log("模型中心:", center);
  //   console.log("模型半径:", radius);

  //   // ✅ 把模型移动到原点
  //   tilesRenderer.group.position.copy(center.clone().multiplyScalar(-1));

  //   // ✅ 更新矩阵
  //   tilesRenderer.group.updateMatrixWorld(true);

  //   // ✅ 相机看原点
  //   const offset = new THREE.Vector3(radius * 2, radius, 0);
  //   initPos = center.clone().add(offset);

  //   camera.position.copy(initPos);
  //   camera.lookAt(center);

  //   controls.target.set(0, 0, 0);
  //   controls.update();

  //   // ✅ 坐标轴放原点
  //   const helper = new THREE.AxesHelper(radius * 0.3);
  //   scene.add(helper);
  // });

  scene.add(tilesRenderer.group)
  // tilesRenderer.setResolutionFromRenderer(camera, renderer)
  // tilesRenderer.setCamera(camera)

  await initPlayer()

  // 窗口大小监听
  onWindowResize();
  window.addEventListener("resize",onWindowResize,false);
  animate()
}

async function initPlayer() {
    player = playerController();
    renderer.render(scene, camera);
    await player.init({
        scene,
        camera,
        controls,
        playerModel: {
            url: "./glb/person2.glb",
            scale: 0.001,
            idleAnim: "idle",
            walkAnim: "walk",
            runAnim: "run",
            jumpAnim: "jump",
        },
        initPos: initPos,
    });
}


function animate() {
  tilesRenderer.setResolutionFromRenderer(camera,renderer);
  tilesRenderer.setCamera(camera);
  requestAnimationFrame(animate)
  camera.updateMatrixWorld()
  controls.update()
  tilesRenderer.update()

  if (player) player.update()

  renderer.render(scene, camera)
}

function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    renderer.setSize(window.innerWidth, window.innerHeight);

    camera.updateProjectionMatrix();
    renderer.setPixelRatio(window.devicePixelRatio);
    tilesRenderer.setResolutionFromRenderer(camera, renderer);
}

</script>

<template>
  <div ref="container" style="width:100%;height:100vh;"></div>
</template>

<style scoped>
</style>