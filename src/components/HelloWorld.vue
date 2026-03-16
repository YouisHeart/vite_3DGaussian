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

  // tiles 加载完成
  tilesRenderer.addEventListener("load-root-tileset", () => {

    const sphere = new THREE.Sphere()
    tilesRenderer.getBoundingSphere(sphere)

    // 模型移动到原点
    tilesRenderer.group.position
      .copy(sphere.center)
      .multiplyScalar(-1)

    // 控制器中心点
    controls.target.copy(new THREE.Vector3(0,0,0))
    controls.update()
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