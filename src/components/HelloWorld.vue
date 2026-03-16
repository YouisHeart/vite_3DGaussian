<script setup>
import { ref,nextTick } from "vue"
import * as THREE from "three"
import { TilesRenderer } from "3d-tiles-renderer"
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js"
import { HDRLoader } from "three/examples/jsm/Addons.js"
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'

const container = ref(null)

let scene
let camera
let renderer
let tilesRenderer
let controls

init();

async function init() {
  // 等待 DOM 挂载
  await nextTick()
  
  scene = new THREE.Scene()
  
  // 加载 HDR - 使用 RGBELoader 替代 HDRLoader
  const rgbeLoader = new RGBELoader()
  const envMap = await rgbeLoader.loadAsync('./hdr/suburban_garden_2k.hdr')
  envMap.mapping = THREE.EquirectangularReflectionMapping
  scene.environment = envMap
  scene.background = envMap

  camera = new THREE.PerspectiveCamera(
    60,
    container.value.clientWidth / container.value.clientHeight,
    0.1,
    100000
  )
  camera.position.set(20, 20, 20)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(
    container.value.clientWidth,
    container.value.clientHeight
  )
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.0

  container.value.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05

  // 3DTiles
  tilesRenderer = new TilesRenderer("./models/out/tileset.json")

  // 注册 gltf loader
  tilesRenderer.manager.addHandler(
    /\.gltf$/,
    new GLTFLoader()
  )

  tilesRenderer.setCamera(camera)
  tilesRenderer.setResolutionFromRenderer(camera, renderer)

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
    controls.target.copy(center); // 把控制器目标设为包围球中心 
    const offset = new THREE.Vector3(radius * 2, radius, 0); // 给相机一个偏移
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

  scene.add(tilesRenderer.group)
  animate()
}

function animate() {
  requestAnimationFrame(animate)
  camera.updateMatrixWorld()
  controls.update()
  tilesRenderer.update()
  renderer.render(scene, camera)

}
</script>

<template>
  <div ref="container" style="width:100%;height:100vh;"></div>
</template>

<style scoped>
</style>