<template>
  <canvas ref="c" style="position:fixed;top:0;left:0;width:100vw;height:100vh;z-index:2;pointer-events:none;" />
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'

const c = ref(null)
let animationId = null
let fragments = []
let fragmentTargets = []   // resting (assembled) positions
let fragmentOrigins = []   // shattered (exploded) positions
let scrollProgress = 0
let lastScrollTime = 0

const FRAGMENT_COUNT = 180

const setScrollProgress = (value) => {
  lastScrollTime = Date.now()
  // Remap input range (0 → 0.35) to output (0 → 1) for faster assembly
  let remapped = value / 0.35
  scrollProgress = Math.max(0, Math.min(1, remapped))
}

defineExpose({ setScrollProgress })

function easeInOutCubic(t) {
  return t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2
}

onMounted(() => {
  const canvas = c.value
  const W = window.innerWidth
  const H = window.innerHeight

  const renderer = new THREE.WebGLRenderer({ canvas, alpha: true, antialias: true })
  renderer.setSize(W, H)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))

  const scene = new THREE.Scene()
  const camera = new THREE.PerspectiveCamera(45, W / H, 0.1, 1000)
  camera.position.z = 5

  scene.add(new THREE.AmbientLight(0xffffff, 0.7))
  const sun = new THREE.DirectionalLight(0xffffff, 2.5)
  sun.position.set(5, 3, 5)
  scene.add(sun)

  // Load texture
  const texture = new THREE.TextureLoader().load('/bg-image.JPG')

  // Build fragments — each is a small curved sphere patch
  const R = 1.5
  const rows = 12
  const cols = 15

  for (let r = 0; r < rows; r++) {
    for (let col = 0; col < cols; col++) {
      const phiStart   = (r / rows) * Math.PI
      const phiEnd     = ((r + 1) / rows) * Math.PI
      const thetaStart = (col / cols) * Math.PI * 2
      const thetaEnd   = ((col + 1) / cols) * Math.PI * 2

      const geo = new THREE.SphereGeometry(
        R, 4, 4,
        thetaStart, thetaEnd - thetaStart,
        phiStart,   phiEnd - phiStart
      )

      const mat = new THREE.MeshStandardMaterial({
        map: texture,
        side: THREE.FrontSide,
        roughness: 0.8,
      })

      const mesh = new THREE.Mesh(geo, mat)

      // Assembled position — stays at origin (sphere is built in place)
      const assembledPos = new THREE.Vector3(0, 0, 0)

      // Shattered position — explode outward from sphere surface midpoint
      const midPhi   = (phiStart + phiEnd) / 2
      const midTheta = (thetaStart + thetaEnd) / 2
      const nx = Math.sin(midPhi) * Math.cos(midTheta)
      const ny = Math.cos(midPhi)
      const nz = Math.sin(midPhi) * Math.sin(midTheta)
      const normal = new THREE.Vector3(nx, ny, nz)

      // Random explosion distance + lateral scatter
      const explodeDist = 2.5 + Math.random() * 4.5
      const lateralX = (Math.random() - 0.5) * 3
      const lateralY = (Math.random() - 0.5) * 3

      const shatteredPos = new THREE.Vector3(
        normal.x * explodeDist + lateralX,
        normal.y * explodeDist + lateralY,
        normal.z * explodeDist
      )

      // Random rotation when shattered
      const shatteredRot = new THREE.Euler(
        (Math.random() - 0.5) * Math.PI * 2,
        (Math.random() - 0.5) * Math.PI * 2,
        (Math.random() - 0.5) * Math.PI * 2
      )

      // Start shattered
      mesh.position.copy(shatteredPos)
      mesh.rotation.copy(shatteredRot)

      scene.add(mesh)
      fragments.push(mesh)
      fragmentTargets.push({ pos: assembledPos, rot: new THREE.Euler(0, 0, 0) })
      fragmentOrigins.push({ pos: shatteredPos.clone(), rot: shatteredRot.clone() })
    }
  }

  let autoRotY = 0

  const loop = () => {
    animationId = requestAnimationFrame(loop)

    const p = easeInOutCubic(scrollProgress)
    const idle = Date.now() - lastScrollTime > 800

    if (idle && scrollProgress >= 0.99) {
      autoRotY += 0.003
    }

    fragments.forEach((mesh, i) => {
      const origin = fragmentOrigins[i]

      // Interpolate position
      mesh.position.lerpVectors(origin.pos, fragmentTargets[i].pos, p)

      // Interpolate rotation back to zero
      mesh.rotation.x = origin.rot.x * (1 - p)
      mesh.rotation.y = origin.rot.y * (1 - p) + autoRotY
      mesh.rotation.z = origin.rot.z * (1 - p)
    })

    renderer.render(scene, camera)
  }
  loop()

  const onResize = () => {
    const nW = window.innerWidth
    const nH = window.innerHeight
    camera.aspect = nW / nH
    camera.updateProjectionMatrix()
    renderer.setSize(nW, nH)
  }
  window.addEventListener('resize', onResize)
  window.addEventListener('orientationchange', () => setTimeout(onResize, 300))
})

onUnmounted(() => {
  if (animationId) cancelAnimationFrame(animationId)
})
</script>