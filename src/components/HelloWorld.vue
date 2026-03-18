<script setup>
import { ref,nextTick } from "vue"
import * as THREE from "three"
import { ACESFilmicToneMapping, AmbientLight, DirectionalLight, EquirectangularReflectionMapping, MathUtils, PerspectiveCamera, Scene, Vector3, WebGLRenderer } from "three";
import { TilesRenderer } from "3d-tiles-renderer"
import { CesiumIonAuthPlugin } from "3d-tiles-renderer/core/plugins";
import { GLTFExtensionsPlugin, ReorientationPlugin, TileCompressionPlugin, TilesFadePlugin } from "3d-tiles-renderer/plugins";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js"
import { HDRLoader } from "three/examples/jsm/Addons.js"
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader.js";
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js'
import { playerController } from "three-player-controller"

const container = ref(null)

let scene
let camera
let renderer
let tiles
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
  reinstantiateTiles();

  await initPlayer();

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
            url: "./glb/person1.glb",
            scale: 0.01,
            idleAnim: "idle",
            walkAnim: "walk",
            runAnim: "run",
            jumpAnim: "jump",
            flyAnim: "flying",
            flyIdleAnim: "flyidle",
            enterCarAnim: "enterCar",
            exitCarAnim: "exitCar",
            headObjName: "mixamorigHead",
            rotateY: Math.PI,
        },
        initPos: new Vector3(100, 100, 100),
        minCamDistance: 50,
        maxCamDistance: 250,
        colliderMeshUrl: "./glb/EiffelCollider.glb",
        thirdMouseMode: 1,
        enableOverShoulderView: true,
    });

    await player.loadVehicleModel({
        url: "./glb/tesla.glb",
        scale: 0.9,
        position: new Vector3(80, 80, 80),
        wheelsNames: [
            "WHEEL_LF", // 前左
            "WHEEL_RF", // 前右
            "WHEEL_LR", // 后左
            "WHEEL_RR", // 后右
        ],
        animations: {
            openDoorAnim: "opendoor",
        },
        boardingPoint: new Vector3(1, 0, 1.9),
        seatOffset: new Vector3(0.1, 0.5, 0),
        chassisRatio: 0.4,
        suspensionRestLengthRatio: 0.2,
        followVehicleDirection: false,
        speedMultiplier: 1.5,
    });
}

function reinstantiateTiles() {
    tiles = new TilesRenderer();
    const apiToken =
        "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJmMzgwMGY3ZS1jOTMwLTQyNmQtOTkyNS03MDE4ZjlkYmY0MTYiLCJpZCI6MjIzMDk3LCJpYXQiOjE3MTg3NjgwNTN9.FcpK7jiFPzWZL8m6VxRbG7ly8LMecpXnDAMZJX_UehM";
    tiles.registerPlugin(
        new CesiumIonAuthPlugin({
            apiToken: apiToken,
            assetId: "2275207",
            autoRefreshToken: true,
        })
    );
    tiles.registerPlugin(new TileCompressionPlugin());
    tiles.registerPlugin(new TilesFadePlugin());
    tiles.registerPlugin(
        new GLTFExtensionsPlugin({
            dracoLoader: new DRACOLoader().setDecoderPath("https://unpkg.com/three@0.153.0/examples/jsm/libs/draco/gltf/"),
        })
    );
    tiles.registerPlugin(
        new ReorientationPlugin({
            lat: 48.8584 * MathUtils.DEG2RAD,
            lon: 2.2945 * MathUtils.DEG2RAD,
        })
    );

    scene.add(tiles.group);
    tiles.setResolutionFromRenderer(camera, renderer);
    tiles.setCamera(camera);
}

function animate() {
    if (!tiles) return;
    requestAnimationFrame(animate); 
    tiles.setResolutionFromRenderer(camera, renderer);
    tiles.setCamera(camera);
    camera.updateMatrixWorld();
    tiles.update();

    if (player) player.update();

    renderer.render(scene, camera);
}

function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    renderer.setSize(window.innerWidth, window.innerHeight);

    camera.updateProjectionMatrix();
    renderer.setPixelRatio(window.devicePixelRatio);
    tiles.setResolutionFromRenderer(camera, renderer);
}

</script>

<template>
  <div ref="container" style="width:100%;height:100vh;"></div>
</template>

<style scoped>
</style>