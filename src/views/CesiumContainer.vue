<!--
 * @version: 1.0.0
 * @Author: liubofang<421419567@qq.com>
 * @Date: 2021-06-15 15:08:22
 * @LastEditTime: 2021-06-18 08:55:24
-->
<!--
 eslint-disable vue/no-multiple-template-root
-->
<template>
  <div id="cesiumContainer"></div>
  <canvas id="canvas"></canvas>
</template>

<script>
import 'cesium/Build/Cesium/Widgets/widgets.css'
import { onMounted, reactive, toRefs } from 'vue'
import * as Cesium from 'cesium'
import * as BABYLON from 'babylonjs'
import 'babylonjs-loaders'
const _this = {}
let canvas = null

console.log(canvas)

Cesium.Ion.defaultAccessToken =
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiIyM2Q2Zjg5Ni05ZjRlLTQ2YjAtOWRkOC1hMmY1YjZjMjk2MDQiLCJpZCI6NzQ2LCJpYXQiOjE1MjU3NDU0Mjl9.LLnHUPCJUlPi0ndq_m1AXpnWBAnoZGO_Z9qyfmDFCAs'
export default {
  name: 'CesiumContainer',
  setup() {
    const data = reactive({
      LNG: -122.4175,
      LAT: 37.655
    })

    const methods = {
      initCesium() {
        const viewer = new Cesium.Viewer('cesiumContainer', {
          useDefaultRenderLoop: false
        })
        viewer.camera.flyTo({
          destination: Cesium.Cartesian3.fromDegrees(data.LNG, data.LAT, 300),
          orientation: {
            heading: Cesium.Math.toRadians(0.0),
            pitch: Cesium.Math.toRadians(-90.0)
          }
        })

        _this.viewer = viewer
        _this.base_point = methods.cart2vec(Cesium.Cartesian3.fromDegrees(data.LNG, data.LAT, 50))
      },

      initBabylon() {
        const engine = new BABYLON.Engine(canvas)
        const scene = (_this.scene = new BABYLON.Scene(engine))
        scene.clearColor = BABYLON.Color4(0, 0, 0, 0)
        const camera = new BABYLON.FreeCamera('camera', new BABYLON.Vector3(0, 0, -10), scene)

        var light = new BABYLON.HemisphericLight('light1', new BABYLON.Vector3(0, 1, 0), scene)
        light.intensity = 0.6
        light.specular = BABYLON.Color3.Black()
        // _this.root_node = new BABYLON.TransformNode('BaseNode', scene)
        // _this.root_node.lookAt(_this.base_point_up.subtract(_this.base_point));
        // _this.root_node.addRotation(1.2, 0, 0)

        methods.loadGlb()
        _this.engine = engine
        _this.scene = scene
        _this.camera = camera
      },

      moveBabylonCamera() {
        const fov = Cesium.Math.toDegrees(_this.viewer.camera.frustum.fovy)
        _this.camera.fov = (fov / 180) * Math.PI

        const civm = _this.viewer.camera.inverseViewMatrix
        const cameraMatrix = BABYLON.Matrix.FromValues(
          civm[0],
          civm[1],
          civm[2],
          civm[3],
          civm[4],
          civm[5],
          civm[6],
          civm[7],
          civm[8],
          civm[9],
          civm[10],
          civm[11],
          civm[12],
          civm[13],
          civm[14],
          civm[15]
        )

        const scaling = BABYLON.Vector3.Zero()
        const rotation = BABYLON.Vector3.Zero()
        const transform = BABYLON.Vector3.Zero()
        cameraMatrix.decompose(scaling, rotation, transform)
        const cameraPos = methods.cart2vec(transform)
        const cameraDirection = methods.cart2vec(_this.viewer.camera.direction)
        const cameraUp = methods.cart2vec(_this.viewer.camera.up)

        let rotationY = Math.atan(cameraDirection.z / cameraDirection.x)
        if (cameraDirection.x < 0) rotationY += Math.PI
        rotationY = Math.PI / 2 - rotationY
        const rotationX = Math.asin(-cameraDirection.y)
        const cameraUpBeforeRotatez = new BABYLON.Vector3(-Math.cos(rotationY), 0, Math.sin(rotationY))
        let rotationZ = Math.acos(cameraUp.x * cameraUpBeforeRotatez.x + cameraUp.y * cameraUpBeforeRotatez.y + cameraUp.z * cameraUpBeforeRotatez.z)
        rotationZ = Math.PI / 2 - rotationZ
        if (cameraUp.y < 0) rotationZ = Math.PI - rotationZ

        _this.camera.position.x = cameraPos.x - _this.base_point.x
        _this.camera.position.y = cameraPos.y - _this.base_point.y
        _this.camera.position.z = cameraPos.z - _this.base_point.z
        _this.camera.rotation.x = rotationX
        _this.camera.rotation.y = rotationY
        _this.camera.rotation.z = rotationZ
      },

      loadGlb() {
        BABYLON.SceneLoader.ImportMesh('', 'https://assets.babylonjs.com/meshes/', 'HVGirl.glb', _this.scene, function (newMeshes, particleSystems, skeletons, animationGroups) {
          // newMeshes[0].parent = _this.root_node
          newMeshes[0].rotationQuaternion = null // 某些网格模型无法旋转时 设置rotationQuaternion = null
          newMeshes[0].rotation.x = -0.9
          newMeshes[0].rotation.y = 0.6
          newMeshes[0].position.y = -82
          const sambaAnim = _this.scene.getAnimationGroupByName('Samba')
          // Play the Samba animation
          sambaAnim.start(true, 1.0, sambaAnim.from, sambaAnim.to, false)
        })
      },

      cart2vec(cart) {
        return new BABYLON.Vector3(cart.x, cart.z, cart.y)
      }
    }

    onMounted(() => {
      canvas = document.getElementsByTagName('canvas')[0]
      methods.initCesium()
      methods.initBabylon()
      _this.engine.runRenderLoop(() => {
        _this.viewer.render()
        methods.moveBabylonCamera()
        _this.scene.render()
      })
    })
    return {
      ...toRefs(data),
      methods
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#cesiumContainer {
  width: 100%;
  height: 100%;
}

#canvas {
  width: 100%;
  height: 100%;
  position: absolute;
  left: 0;
  top: 0;
  pointer-events: none;
}
</style>
