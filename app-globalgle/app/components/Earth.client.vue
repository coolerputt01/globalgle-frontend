<template>
  <canvas ref="threeCanvas" class="three-canvas"></canvas>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

const threeCanvas = ref(null)
let scene, camera, renderer, earthGroup, earthMesh, particleSystem
let particlesPositions = []
let randomOffsets = []
let progress = 0
let animationId = null
let isInitialized = false

const JOINED_THRESHOLD = 0.58

const setShatterProgress = (value) => {
  const clamped = Math.min(1, Math.max(0, value))
  // Only skip update if already joined and still above threshold (scrolling further down)
  if (progress >= JOINED_THRESHOLD && clamped >= JOINED_THRESHOLD) return
  progress = clamped
  updateParticles()
}

const setYOffset = (y) => {
  if (earthGroup) earthGroup.position.y = y
}

defineExpose({ setShatterProgress, setYOffset })

function updateParticles() {
  if (!particleSystem) return
  const positions = particleSystem.geometry.attributes.position.array
  const t = 0.2 - Math.pow(progress, 1.2)
  for (let i = 0; i < particlesPositions.length; i++) {
    const orig = particlesPositions[i]
    const offset = randomOffsets[i]
    positions[i*3]   = orig.x + offset.x * t * 2.5
    positions[i*3+1] = orig.y + offset.y * t * 2.5
    positions[i*3+2] = orig.z + offset.z * t * 2.5
  }
  particleSystem.geometry.attributes.position.needsUpdate = true
}

// Sample the texture on a CPU-side canvas to get per-particle colors
function sampleTextureColors(texture, particleCount, positions) {
  const image = texture.image
  const canvas = document.createElement('canvas')
  canvas.width = image.width
  canvas.height = image.height
  const ctx = canvas.getContext('2d')
  ctx.drawImage(image, 0, 0)
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
  const data = imageData.data

  const colors = new Float32Array(particleCount * 10)

  for (let i = 0; i < particleCount; i++) {
    const x = positions[i * 3]
    const y = positions[i * 3 + 1]
    const z = positions[i * 3 + 2]

    // Convert 3D sphere position → spherical UV coords
    const len = Math.sqrt(x*x + y*y + z*z)
    const nx = x / len, ny = y / len, nz = z / len
    const u = 0.5 + Math.atan2(nz, nx) / (2 * Math.PI)
    const v = 0.5 - Math.asin(ny) / Math.PI

    const px = Math.floor(u * (canvas.width  - 1))
    const py = Math.floor(v * (canvas.height - 1))
    const idx = (py * canvas.width + px) * 4

    colors[i * 3]     = data[idx]     / 255
    colors[i * 3 + 1] = data[idx + 1] / 255
    colors[i * 3 + 2] = data[idx + 2] / 255
  }

  return colors
}

async function initThree() {
  if (isInitialized) return
  if (!threeCanvas.value) {
    await nextTick()
    if (!threeCanvas.value) return
  }

  scene = new THREE.Scene()
  scene.background = null

  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 0, 8)

  try {
    renderer = new THREE.WebGLRenderer({
      canvas: threeCanvas.value,
      alpha: true,
      antialias: true,
      preserveDrawingBuffer: false,
      powerPreference: 'high-performance',
    })
    renderer.setSize(window.innerWidth, window.innerHeight)
    renderer.setPixelRatio(window.devicePixelRatio)
  } catch (err) {
    console.error('WebGL not supported:', err)
    return
  }

  // Lighting
  scene.add(new THREE.AmbientLight(0x404060))
  const dirLight = new THREE.DirectionalLight(0xffffff, 1)
  dirLight.position.set(1, 2, 1)
  scene.add(dirLight)
  const backLight = new THREE.PointLight(0x4466cc, 0.5)
  backLight.position.set(0, 0, -2)
  scene.add(backLight)

  // Stars
  const starGeo = new THREE.BufferGeometry()
  const starCount = 1500
  const starPos = new Float32Array(starCount * 3)
  for (let i = 0; i < starCount; i++) {
    starPos[i*3]   = (Math.random() - 0.5) * 2000
    starPos[i*3+1] = (Math.random() - 0.5) * 1000
    starPos[i*3+2] = (Math.random() - 0.5) * 200 - 50
  }
  starGeo.setAttribute('position', new THREE.BufferAttribute(starPos, 3))
  scene.add(new THREE.Points(starGeo, new THREE.PointsMaterial({ color: 0xffffff, size: 0.8,sizeAttenuation: false,  })))

  // Load GLTF + texture together so both are ready before we use them
  const loader = new GLTFLoader()
  const textureLoader = new THREE.TextureLoader()

  let earthTexture = null
  try {
    const [gltf, tex] = await Promise.all([
      loader.loadAsync('/earth.glb'),
      textureLoader.loadAsync('/earth-texture.jpg'),
    ])
    earthTexture = tex
    earthMesh = gltf.scene
    earthMesh.traverse((child) => {
      if (child.isMesh) {
        child.material = new THREE.MeshStandardMaterial({
          map: earthTexture,
          roughness: 0.8,
          metalness: 0.1,
        })
      }
    })
    earthMesh.scale.set(1.2, 1.2, 1.2)
  } catch (error) {
    console.error('Earth assets failed, using fallback sphere:', error)
    try {
      earthTexture = await textureLoader.loadAsync('/earth-texture.jpg')
    } catch (_) {}
    const geometry = new THREE.SphereGeometry(1, 64, 64)
    earthMesh = new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({
      map: earthTexture ?? null,
      color: earthTexture ? 0xffffff : 0x2c3e66,
      roughness: 0.8,
      metalness: 0.1,
    }))
    earthMesh.scale.set(1.2, 1.2, 1.2)
  }

  // ── Particles sampled from the texture ────────────────────────────────────
  const earthRadius = 1.0
  const particleCount = 4000
  const positionsArray = new Float32Array(particleCount * 10)

  for (let i = 0; i < particleCount; i++) {
    const theta = Math.random() * Math.PI * 2
    const phi   = Math.acos(2 * Math.random() - 1)
    const x = earthRadius * Math.sin(phi) * Math.cos(theta)
    const y = earthRadius * Math.sin(phi) * Math.sin(theta)
    const z = earthRadius * Math.cos(phi)
    particlesPositions.push(new THREE.Vector3(x, y, z))
    const dir = new THREE.Vector3(x, y, z).normalize()
    randomOffsets.push(dir.multiplyScalar(1.5 + Math.random() * 2.5))
    positionsArray[i*3]   = x
    positionsArray[i*3+1] = y
    positionsArray[i*3+2] = z
  }

  const particleGeo = new THREE.BufferGeometry()
  particleGeo.setAttribute('position', new THREE.BufferAttribute(positionsArray, 3))

  // If we have a texture, sample its colors per-particle; otherwise fall back to green
  let particleMat
  if (earthTexture && earthTexture.image) {
    const sampledColors = sampleTextureColors(earthTexture, particleCount, positionsArray)
    particleGeo.setAttribute('color', new THREE.BufferAttribute(sampledColors, 3))
    particleMat = new THREE.PointsMaterial({
      vertexColors: true,
      size: 0.12,
      transparent: true,
      opacity: 0.9,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })
  } else {
    particleMat = new THREE.PointsMaterial({
      color: 0x00ff88,
      size: 0.05,
      transparent: true,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })
  }

  particleSystem = new THREE.Points(particleGeo, particleMat)

  earthGroup = new THREE.Group()
  earthGroup.add(earthMesh)
  earthGroup.add(particleSystem)
  scene.add(earthGroup)

  updateParticles()

  function animate() {
    animationId = requestAnimationFrame(animate)
    if (earthMesh) earthMesh.rotation.y += 0.02
    if (particleSystem) particleSystem.rotation.y += 0.05
    if (renderer && scene && camera) renderer.render(scene, camera)
  }
  animate()

  const onWindowResize = () => {
    if (!camera || !renderer) return
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
  }
  window.addEventListener('resize', onWindowResize)
  cleanupFns.push(() => window.removeEventListener('resize', onWindowResize))

  isInitialized = true
}

const cleanupFns = []
onMounted(() => { initThree() })
onUnmounted(() => {
  if (animationId) cancelAnimationFrame(animationId)
  cleanupFns.forEach(fn => fn())
  if (renderer) { renderer.dispose(); renderer = null }
  isInitialized = false
})
</script>

<style scoped>
.three-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: block;
  z-index: 1;
}
</style>