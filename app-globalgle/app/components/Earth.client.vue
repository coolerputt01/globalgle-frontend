<template>
  <canvas ref="c" style="position:fixed;top:0;left:0;width:100vw;height:100vh;z-index:2;pointer-events:none;" />
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'

const c = ref(null)
let animationId = null
let earthMesh = null
let lastScrollTime = 0

const setScrollProgress = (value) => {
  lastScrollTime = Date.now()
  if (earthMesh) earthMesh.rotation.y = value * Math.PI * 2
}

defineExpose({ setScrollProgress })

onMounted(() => {
  const canvas = c.value
  const W = window.innerWidth
  const H = window.innerHeight

  const renderer = new THREE.WebGLRenderer({ canvas, alpha: true, antialias: true })
  renderer.setSize(W, H)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  canvas.style.width = W + 'px'
  canvas.style.height = H + 'px'

  const scene = new THREE.Scene()
  const camera = new THREE.PerspectiveCamera(45, W / H, 0.1, 1000)
  camera.position.z = 5

  scene.add(new THREE.AmbientLight(0xffffff, 0.8))
  const sun = new THREE.DirectionalLight(0xffffff, 2.5)
  sun.position.set(5, 3, 5)
  scene.add(sun)

  // One sphere, no swap ever
  const geo = new THREE.SphereGeometry(1.5, 64, 64)
  const mat = new THREE.MeshStandardMaterial({ color: 0x2a6f8f })
  earthMesh = new THREE.Mesh(geo, mat)
  scene.add(earthMesh)

  // Load texture onto the sphere
  new THREE.TextureLoader().load('/texture.jpg', (texture) => {
    mat.map = texture
    mat.color.set(0xffffff)
    mat.needsUpdate = true
  })

  const loop = () => {
    animationId = requestAnimationFrame(loop)
    if (Date.now() - lastScrollTime > 1000) {
      earthMesh.rotation.y += 0.003
    }
    renderer.render(scene, camera)
  }
  loop()

  window.addEventListener('orientationchange', () => {
    setTimeout(() => {
      const nW = window.innerWidth
      const nH = window.innerHeight
      camera.aspect = nW / nH
      camera.updateProjectionMatrix()
      renderer.setSize(nW, nH)
      canvas.style.width = nW + 'px'
      canvas.style.height = nH + 'px'
    }, 300)
  })
})

onUnmounted(() => {
  if (animationId) cancelAnimationFrame(animationId)
})
</script>