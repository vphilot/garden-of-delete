<script setup lang="ts">
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { GLTFExporter, GLTFExporterOptions } from 'three/examples/jsm/exporters/GLTFExporter'
import { AnimationClip, AnimationMixer, BoxGeometry, Clock, Color, DoubleSide, GridHelper, Group, Mesh, MeshBasicMaterial, Object3D, PCFSoftShadowMap, PerspectiveCamera, Plane, PlaneGeometry, PointLight, RectAreaLight, Scene, Vector2, Vector3, WebGLRenderer, sRGBEncoding } from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { RectAreaLightUniformsLib } from 'three/examples/jsm/lights/RectAreaLightUniformsLib.js'
import type { Ref } from 'vue'

import bodyModel from '../assets/models/body_model.glb?url'

interface LoadGLTFParams {
  url: string
  name: string
  rigged?: boolean
  basePoses?: boolean
  visible?: boolean
}

defineOptions({
  name: 'IndexPage',
})

const container: Ref<HTMLDivElement | undefined> = ref()
const viewer: Ref<HTMLDivElement | undefined> = ref()

// scene
const scene = new Scene()
const sceneBackground = new Color(0xAAAAAA)
scene.background = sceneBackground

// renderer
const renderer = new WebGLRenderer({ antialias: true, alpha: true, preserveDrawingBuffer: true, powerPreference: 'high-performance' })
renderer.outputEncoding = sRGBEncoding
renderer.info.autoReset = true
renderer.shadowMap.enabled = true
renderer.shadowMap.type = PCFSoftShadowMap
renderer.localClippingEnabled = true

// camera
const camera = new PerspectiveCamera(50, 1, 0.1, 1000)
camera.position.set(10, 10, 10)
const orbitControls = new OrbitControls(camera, renderer.domElement)
orbitControls.enableRotate = true
// orbitControls.enableDamping = true
// orbitControls.enablePan = false
// orbitControls.minPolarAngle = 0.3
// orbitControls.maxPolarAngle = 1.65
// orbitControls.minDistance = 1.7
// orbitControls.maxDistance = 3.3

// lights
const rectlight = new RectAreaLight(0xFFFFFF, 2, 10, 4)
rectlight.position.set(0, 2, 1.65)
RectAreaLightUniformsLib.init()

const pl1 = new PointLight(0xFFFFFF, 5, 100)
pl1.position.set(0, 10, 5)
pl1.castShadow = true
const pl2 = new PointLight(0xFFFFFF, 5, 100)
pl2.position.set(0, 10, -5)
pl2.castShadow = true

const lightsPivot = new Object3D()
scene.add(lightsPivot)
lightsPivot.add(rectlight, pl1, pl2)

const loadGLTF = async ({ url, name, rigged = true, basePoses = false, visible = true }: LoadGLTFParams): Promise<boolean> => {
  if (scene.getObjectByName(name))
    return new Promise(resolve => resolve(true))

  const loader = new GLTFLoader()
  return new Promise((resolve, reject) => {
    loader.load(
      url,
      (gltf) => {
        gltf.scene.name = name
        gltf.scene.visible = visible

        // if (rigged) {
        // // set an individual animation mixer for every poseable object
        //   animationMixers.value.push({
        //     name,
        //     mixer: new AnimationMixer(gltf.scene),
        //   })

        //   if (basePoses && gltf.animations && gltf.animations.length) {
        //     gltf.animations.map((animation) => {
        //       animationClips.value[animation.name.toLowerCase()] = Object.assign({}, animation, { name: animation.name.toLowerCase() })
        //     })
        //   }
        // }

        scene.add(gltf.scene)

        scene.traverse((child) => {
        // workaround for snikked mesh clipping
          child.frustumCulled = false
        })

        // loadedModels.value.push(name)
        resolve(true)
      },

      () => {
      // handle progress
      },

      (error) => {
        reject(error)
      },
    )
  })
}

const resizeEvent = () => {
  const width = container.value?.clientWidth
  const height = container.value?.clientHeight
  if (width && height) {
    const aspectRatio = width / height
    camera.aspect = aspectRatio
    camera.updateProjectionMatrix()
    renderer.setSize(width, height)
  }
}

const refresh = async () => {
  orbitControls.update()

  // rotate objects along with camera
  // lightsPivot.rotation.y = orbitControls.getAzimuthalAngle()

  requestAnimationFrame(refresh)
  renderer.render(scene, camera)
  // const delta = clock.getDelta()
  // animationMixers.value.forEach(item => {
  //   item.mixer.update(delta)
  // })

  // debug
  // stats.value?.update()
}

onMounted(async () => {
  (viewer.value as HTMLElement).appendChild(renderer.domElement)

  resizeEvent()
  window.addEventListener('resize', resizeEvent)

  // const targetPos = fitCameraToObject(camera, avatarBaseMesh as unknown as Group, 1.25, undefined, 0, false)

  // override camera.far since we need to show the full env
  // camera.far = 100
  // camera.position.set(targetPos.x, targetPos.y, 3.3)
  // camera.updateProjectionMatrix()
  // orbitControls.target = new Vector3(0, targetPos.y, 0)

  // await loadPoses()
  await loadGLTF({ url: bodyModel, name: 'body' })
  const body = scene.getObjectByName('body')

  scene.add(new GridHelper(20, 20))
  const geometry = new PlaneGeometry(20, 20)
  const material = new MeshBasicMaterial({ color: 'black' })
  const plane = new Mesh(geometry, material)
  plane.rotateOnAxis(new Vector3(-1, 0, 0), Math.PI / 2)
  scene.add(plane)

  await refresh()

  // toggleOrbitControls()
  // isReady.value = true
  // emit('ready', isReady.value)
})
</script>

<template>
  <!-- <a :href="bodyModel" download>download</a> -->
  <div ref="container" class="w-screen h-screen">
    <div ref="viewer" />
  </div>
</template>
