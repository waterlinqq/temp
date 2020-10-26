<template>
  <div class="h-100">
    <div style="position: absolute">
      <button @click="rotate('front')">front</button>
      <button @click="rotate('back')">back</button>
      <button @click="rotate('left')">left</button>
      <button @click="rotate('right')">right</button>
      <button @click="rotate('up')">up</button>
      <br>
      <button @click="model.getObjectByName('BLOGO').visible = true">show LOGO</button>
      <button @click="model.getObjectByName('BLOGO').visible = false">hide LOGO</button>
      <button @click="model.getObjectByName('mesh_15').material.color.set('white')">white</button>
      <button @click="model.getObjectByName('mesh_15').material.color.set('red')">red</button>
      <button @click="model.getObjectByName('mesh_15').material.color.set('black')">black</button>
      <button @click="model.getObjectByName('mesh_15').material.metalness = 1">metal</button>
      <button @click="model.getObjectByName('mesh_15').material.metalness = 0">plane</button>
      <br>
      <button @click="mixer.clipAction(animations[0]).play()">動畫1</button>
      <button @click="mixer.clipAction(animations[1]).play()">動畫2</button>
      <button @click="mixer.clipAction(animations[2]).play()">動畫3</button>

    </div>
    <div id="canvas-wrapper"></div>
    <!-- <canvas class="w-100 h-100" ref="canvas"></canvas> -->
  </div>
</template>
<script>
import axios from 'axios'
import * as THREE from 'three'
// import gsap from 'gsap'
// import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { GUI } from 'dat.gui'
// import TWEEN from '@tweenjs/tween.js'
// import { ObjectControls } from 'threejs-object-controls'

import CameraControls from 'camera-controls'
import { MeshBasicMaterial } from 'three'

CameraControls.install({ THREE: THREE })
export default {
  async mounted () {
    const { id: modelId } = this.$route.query
    let gltfSrc
    if (modelId == null) return
    // dN53AKsFkbU
    if (modelId === '07m9bHwS4C6') {
      gltfSrc = 'https://poly.googleusercontent.com/downloads/c/fp/1599822236116571/07m9bHwS4C6/8mxkdQAxJod/RobotExpressive.gltf'
    } else {
      // dLHpzNdygsg
      const resp = await axios.get(`https://web-ar-d9581.firebaseio.com//webar-items/${modelId}.json`)
      if (resp.status !== 200) return
      if (resp.data == null) return
      gltfSrc = resp.data.gltfSrc
    }
    this.init()
    this.addLights()
    this.setSize()
    this.render()
    this.listenEvents()
    this.addTags()
    this.addHelpers()
    await this.load(gltfSrc)
    window.v = this
    window.THREE = THREE
  },
  methods: {
    init () {
      this.renderer = new THREE.WebGLRenderer()
      this.gui = new GUI()
      this.camera = new THREE.PerspectiveCamera()
      this.clock = new THREE.Clock()
      this.scene = new THREE.Scene()
      this.rafId = -1
      this.model = null
      this.animations = null
      this.mixer = null
      setCamera(this.camera)
      setRenderer(this.renderer)
      setScene(this.scene)
      this.cameraControls = new CameraControls(this.camera, this.renderer.domElement)
      // this.cameraControls.update(0)
      function setCamera (camera) {
        camera.fov = 45
        camera.aspect = 2 // the canvas default
        camera.near = 0.01
        camera.far = 500
        // camera.position.set(0, 30, 20)
      }
      function setRenderer (renderer) {
        renderer.setPixelRatio(window.devicePixelRatio)
        renderer.setSize(window.innerWidth, window.innerHeight)
        document.getElementById('canvas-wrapper').appendChild(renderer.domElement)
      }
      function setScene (scene) {
        scene.background = new THREE.Color('#DEFEFF')
      }
    },
    addLights () {
      const { scene, gui } = this
      const lightFolder = gui.addFolder('light')
      {
        const skyColor = 0xffffff // light blue
        const groundColor = 0xffffff // brownish orange
        const intensity = 1
        const light = new THREE.HemisphereLight(
          skyColor,
          groundColor,
          intensity
        )
        scene.add(light)
        lightFolder.add(light, 'intensity', 0, 10, 0.5).name('HemisphereLight')
      }

      {
        const color = 0xFFFFFF
        const intensity = 1
        const light = new THREE.AmbientLight(color, intensity)
        scene.add(light)
        lightFolder.add(light, 'intensity', 0, 10, 0.5).name('AmbientLight')
      }

      {
        const color = 0xffffff
        const intensity = 1
        const light = new THREE.DirectionalLight(color, intensity)
        light.position.set(5, 10, 2)
        scene.add(light)
        scene.add(light.target)
        lightFolder.add(light, 'intensity', 0, 10, 0.5).name('DirectionalLight')
      }
    },
    setSize () {
      const { camera, renderer, cameraControls } = this
      const width = window.innerWidth
      const height = window.innerHeight
      const viewport = new THREE.Vector4(0, 0, width, height)
      camera.aspect = width / height
      camera.updateProjectionMatrix()
      renderer.setSize(width, height)
      renderer.setViewport(viewport)
      cameraControls.setViewport(viewport)
      // }
    },
    render () {
      const { clock, cameraControls, renderer, scene, camera, mixer } = this
      const delta = clock.getDelta()
      // eslint-disable-next-line no-unused-vars
      const elapsed = clock.getElapsedTime()
      const updated = cameraControls.update(delta)

      // if (elapsed > 30) {
      //   return
      // }

      this.rafId = requestAnimationFrame(this.render)

      if (updated) {
        renderer.render(scene, camera)
        if (mixer) {
          mixer.update(clock.getDelta())
        }
      }
    },
    listenEvents () {
      window.addEventListener('resize', this.setSize)
    },
    load (gltfSrc) {
      let finishLoad
      const { camera, scene, cameraControls } = this
      const gltfLoader = new GLTFLoader()
      gltfLoader.load(gltfSrc, (gltf) => {
        const model = gltf.scene
        const animations = gltf.animations
        this.animations = animations
        this.model = model
        this.mixer = new THREE.AnimationMixer(model)
        scene.add(model)

        const mesh = new THREE.Mesh(
          new THREE.BoxGeometry(0.1, 0.1, 0.1),
          new MeshBasicMaterial({ color: 'red', wireframe: true })
        )
        scene.add(mesh)

        // compute the box that contains all the stuff
        // from root and below
        const box = new THREE.Box3().setFromObject(model)

        const boxSize = box.getSize(new THREE.Vector3()).length()
        const boxCenter = box.getCenter(new THREE.Vector3())
        // model.position.multiplyVectors(
        //   new THREE.Vector3(-1, -0.5, -1),
        //   boxCenter
        // )
        model.position.set(...boxCenter.multiplyScalar(-1).toArray())

        // set the camera to frame the box
        // frameArea(boxSize * 0.5, boxSize, boxCenter, camera)
        console.log(frameArea, boxSize * 0.5, boxSize, boxCenter, camera)
        console.log('boxsize', boxSize)
        cameraControls.setPosition(
          ...new THREE.Vector3(0, 1.5, 3)
            .normalize()
            .multiplyScalar(boxSize)
            .toArray()
        )
        cameraControls.truck(0, -0.1 * boxSize)
        cameraControls.maxDistance = boxSize * 1.5
        cameraControls.minDistance = boxSize / 2 + boxSize / 100
        console.log(boxSize)
        camera.near = boxSize / 100
        camera.far = boxSize * 10
        finishLoad()
        // update the Trackball controls to handle the new size
        // controls.maxDistance = boxSize * 10
        // controls.target.copy(boxCenter)
        // controls.update()
      })
      function frameArea (sizeToFitOnScreen, boxSize, boxCenter, camera) {
        const halfSizeToFitOnScreen = sizeToFitOnScreen * 0.5
        const halfFovY = THREE.MathUtils.degToRad(camera.fov * 0.5)
        const distance = halfSizeToFitOnScreen / Math.tan(halfFovY)
        // compute a unit vector that points in the direction the camera is now
        // in the xz plane from the center of the box
        const direction = new THREE.Vector3()
          .subVectors(camera.position, boxCenter)
          .multiply(new THREE.Vector3(1, 0, 1))
          .normalize()
        console.log(direction, distance, boxCenter)
        // move the camera to a position distance units way from the center
        // in whatever direction the camera was from the center already
        // camera.position.copy(
        //   direction.multiplyScalar(distance).add(boxCenter)
        // )
        cameraControls.setPosition(
          ...direction
            .multiplyScalar(distance * 1.01)
            .add(boxCenter)
            .toArray()
        )

        // cameraControls.setPosition(

        // )

        // pick some near and far values for the frustum that
        // will contain the box.
        camera.near = boxSize / 100
        camera.far = boxSize * 1000

        camera.updateProjectionMatrix()

        // point the camera to look at the center of the box
        camera.lookAt(boxCenter.x, boxCenter.y, boxCenter.z)
      }
      return new Promise(function (resolve) {
        finishLoad = resolve
      })
      // function resizeRendererToDisplaySize (renderer) {
      //   const canvas = renderer.domElement
      //   const width = canvas.clientWidth
      //   const height = canvas.clientHeight
      //   const needResize = canvas.width !== width || canvas.height !== height
      //   if (needResize) {
      //     renderer.setSize(width, height, false)
      //   }
      //   return needResize
      // }

      // function render () {
      //   if (resizeRendererToDisplaySize(renderer)) {
      //     const canvas = renderer.domElement
      //     camera.aspect = canvas.clientWidth / canvas.clientHeight
      //     camera.updateProjectionMatrix()
      //   }

      //   renderer.render(scene, camera)

      //   requestAnimationFrame(render)
      // }

      // requestAnimationFrame(render)
    },
    addTags () {
      // "0.006749352405163555m 0.19659507278139957m -0.1272738747457698m"
      const { scene } = this
      const mesh = new THREE.Mesh(
        new THREE.BoxGeometry(0.06, 0.05, 0.05),
        new MeshBasicMaterial({ color: 'red', wireframe: true })
      )
      mesh.position.set(
        0.006749352405163555,
        0.19659507278139957,
        -0.1272738747457698
      )

      scene.add(mesh)
    },
    rotate (side) {
      const { renderer, cameraControls, camera } = this
      const DEG90 = Math.PI * 0.5
      const DEG180 = Math.PI
      console.log(rotateTo, paddingInCssPixel)
      cameraControls.normalizeRotations()
      rotateTo(side)
      function rotateTo (side) {
        switch (side) {
          case 'front':
            cameraControls.rotateTo(0, DEG90, true)
            break

          case 'back':
            cameraControls.rotateTo(DEG180, DEG90, true)
            break

          case 'up':
            cameraControls.rotateTo(0, 0, true)
            break

          case 'down':
            cameraControls.rotateTo(0, DEG180, true)
            break

          case 'right':
            cameraControls.rotateTo(DEG90, DEG90, true)
            break

          case 'left':
            cameraControls.rotateTo(-DEG90, DEG90, true)
            break
        }
      }

      function paddingInCssPixel (top, right, bottom, left, mesh) {
        const fov = camera.fov * THREE.MathUtils.DEG2RAD
        const rendererHeight = renderer.getSize(new THREE.Vector2()).height

        const boundingBox = new THREE.Box3().setFromObject(mesh)
        const size = boundingBox.getSize(new THREE.Vector3())
        const boundingWidth = size.x
        const boundingHeight = size.y
        const boundingDepth = size.z

        var distanceToFit = cameraControls.getDistanceToFitBox(boundingWidth, boundingHeight, boundingDepth)
        var paddingTop = 0
        var paddingBottom = 0
        var paddingLeft = 0
        var paddingRight = 0

        // loop to find almost convergence points
        for (var i = 0; i < 10; i++) {
          const depthAt = distanceToFit - boundingDepth * 0.5
          const cssPixelToUnit = (2 * Math.tan(fov * 0.5) * Math.abs(depthAt)) / rendererHeight
          paddingTop = top * cssPixelToUnit
          paddingBottom = bottom * cssPixelToUnit
          paddingLeft = left * cssPixelToUnit
          paddingRight = right * cssPixelToUnit

          distanceToFit = cameraControls.getDistanceToFitBox(
            boundingWidth + paddingLeft + paddingRight,
            boundingHeight + paddingTop + paddingBottom,
            boundingDepth
          )
        }

        cameraControls.fitToBox(mesh, true, { paddingLeft: paddingLeft, paddingRight: paddingRight, paddingBottom: paddingBottom, paddingTop: paddingTop })
      }
    },
    addHelpers () {
      const { scene } = this
      const gridHelper = new THREE.GridHelper(1, 10)
      scene.add(gridHelper)
    }
  }

}
</script>
