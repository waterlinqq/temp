<template>
  <div class="h-100">
    <canvas class="w-100 h-100" ref="canvas"></canvas>
  </div>
</template>
<script>
import axios from 'axios'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { GUI } from 'dat.gui'

export default {
  data () {
    return {
      renderer: null,
      gui: null,
      model: null
    }
  },
  async created () {
    const { id: modelId } = this.$route.query
    if (modelId == null) return
    // 00kPnwTB1gO
    const resp = await axios.get(`https://web-ar-d9581.firebaseio.com//webar-items/${modelId}.json`)
    if (resp.status !== 200) return
    if (resp.data == null) return
    console.log(resp.data)
    const { gltfSrc } = resp.data
    this.load(gltfSrc)
    console.log(GUI)
  },
  methods: {
    load (gltfSrc) {
      const canvas = this.$refs.canvas
      const renderer = new THREE.WebGLRenderer({ canvas })
      const gui = new GUI()
      renderer.gammaFactor = 10
      // renderer.toneMappingExposure = 5
      renderer.setPixelRatio(window.devicePixelRatio)

      const fov = 45
      const aspect = 2 // the canvas default
      const near = 0.1
      const far = 100
      const camera = new THREE.PerspectiveCamera(fov, aspect, near, far)

      camera.position.set(0, 10, 20)

      const controls = new OrbitControls(camera, canvas)
      controls.target.set(0, 0, 0)
      controls.update()

      const scene = new THREE.Scene()
      scene.background = new THREE.Color('#DEFEFF')

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
        gui.add(light, 'intensity', 0, 10, 0.5).name('HemisphereLight')
      }
      {
        const color = 0xFFFFFF
        const intensity = 1
        const light = new THREE.AmbientLight(color, intensity)
        scene.add(light)
        gui.add(light, 'intensity', 0, 10, 0.5).name('AmbientLight')
      }

      {
        const color = 0xffffff
        const intensity = 1
        const light = new THREE.DirectionalLight(color, intensity)
        light.position.set(5, 10, 2)
        scene.add(light)
        scene.add(light.target)
        gui.add(light, 'intensity', 0, 10, 0.5).name('DirectionalLight')
      }

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

        // move the camera to a position distance units way from the center
        // in whatever direction the camera was from the center already
        camera.position.copy(
          direction.multiplyScalar(distance).add(boxCenter)
        )

        // pick some near and far values for the frustum that
        // will contain the box.
        camera.near = boxSize / 100
        camera.far = boxSize * 100

        camera.updateProjectionMatrix()

        // point the camera to look at the center of the box
        camera.lookAt(boxCenter.x, boxCenter.y, boxCenter.z)
      }

      {
        const gltfLoader = new GLTFLoader()
        gltfLoader.load(gltfSrc, (gltf) => {
          const root = gltf.scene
          gui.add(root.position, 'x', -10, 10, 1).name('px')
          gui.add(root.position, 'y', -10, 10, 1).name('py')
          gui.add(root.position, 'z', -10, 10, 1).name('pz')
          scene.add(root)

          gui.add(root.rotation, 'x', 0, 2 * Math.PI, 0.1).name('rx')
          window.r = root.rotation

          // compute the box that contains all the stuff
          // from root and below
          const box = new THREE.Box3().setFromObject(root)

          const boxSize = box.getSize(new THREE.Vector3()).length()
          const boxCenter = box.getCenter(new THREE.Vector3())

          // set the camera to frame the box
          frameArea(boxSize * 0.5, boxSize, boxCenter, camera)

          // update the Trackball controls to handle the new size
          controls.maxDistance = boxSize * 10
          controls.target.copy(boxCenter)
          controls.update()
        })
      }

      function resizeRendererToDisplaySize (renderer) {
        const canvas = renderer.domElement
        const width = canvas.clientWidth
        const height = canvas.clientHeight
        const needResize = canvas.width !== width || canvas.height !== height
        if (needResize) {
          renderer.setSize(width, height, false)
        }
        return needResize
      }

      function render () {
        if (resizeRendererToDisplaySize(renderer)) {
          const canvas = renderer.domElement
          camera.aspect = canvas.clientWidth / canvas.clientHeight
          camera.updateProjectionMatrix()
        }

        renderer.render(scene, camera)

        requestAnimationFrame(render)
      }

      requestAnimationFrame(render)
    }

  }
}
</script>
