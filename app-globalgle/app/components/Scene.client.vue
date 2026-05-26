<template>
  <div class="character-container">
    <div class="character-model" ref="canvasDiv">
      <div class="character-rim"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js'

async function decryptFile(url, password) {
  const response = await fetch(url);
  const encryptedData = await response.arrayBuffer();
  const iv = new Uint8Array(encryptedData.slice(0, 16));
  const data = encryptedData.slice(16);
  const passwordBuffer = new TextEncoder().encode(password);
  const hash = await crypto.subtle.digest('SHA-256', passwordBuffer);
  const rawKey = hash.slice(0, 32);
  const cryptoKey = await crypto.subtle.importKey(
    'raw',
    rawKey,
    { name: 'AES-CBC' },
    false,
    ['decrypt']
  );
  const decrypted = await crypto.subtle.decrypt(
    { name: 'AES-CBC', iv },
    cryptoKey,
    data
  );
  return decrypted;
}

const canvasDiv = ref(null)
let renderer = null
let animationId = null
let scene = new THREE.Scene()
let camera = null
let mixer = null          // will remain null – animations disabled
let screenLight = null    // will be hidden
let mouse = { x: 0, y: 0 }
const interpolation = { x: 0.18, y: 0.28 }

function setLighting(scene) {
  const ambient = new THREE.AmbientLight(0xffffff, 0.8)
  scene.add(ambient)
  const key = new THREE.DirectionalLight(0xffffff, 1.2)
  key.position.set(5, 8, 5)
  scene.add(key)
  const fill = new THREE.DirectionalLight(0x88aaff, 0.6)
  fill.position.set(-4, 2, 6)
  scene.add(fill)
  const back = new THREE.DirectionalLight(0xffaa66, 0.4)
  back.position.set(2, 4, -5)
  scene.add(back)
  const rim = new THREE.PointLight(0x00ff88, 0.5)
  rim.position.set(3, 5, 4)
  scene.add(rim)

  function setPointLight(obj) {
    if (obj) {
      const pos = new THREE.Vector3()
      obj.getWorldPosition(pos)
      rim.position.copy(pos)
      rim.intensity = 0.9
    }
  }
  return { setPointLight }
}

const cleanupFns = []

onMounted(async () => {
  if (!canvasDiv.value) return
  await new Promise(r => setTimeout(r, 50))

  const rect = canvasDiv.value.getBoundingClientRect()
  const w = rect.width || window.innerWidth
  const h = rect.height || window.innerHeight

  renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true })
  renderer.setSize(w, h)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.2
  renderer.domElement.style.position = 'absolute'
  renderer.domElement.style.inset = '0'
  renderer.domElement.style.zIndex = '1'
  canvasDiv.value.appendChild(renderer.domElement)

  // ── Camera – zoomed in ──────────────────────────────────────────────
  const isMobile = w < 768
  camera = new THREE.PerspectiveCamera(16, w / h, 0.1, 1000)
  if (isMobile) {
    camera.position.set(0, 6, 18)
  } else {
    // Closer framing: lower height, shorter distance
    camera.position.set(0.5,25, 14)
  }
  camera.lookAt(0, 5, -10)
  camera.updateProjectionMatrix()

  const light = setLighting(scene)

  // HDR environment (keep for reflections)
  try {
    const { RGBELoader } = await import('three/examples/jsm/loaders/RGBELoader.js')
    const pmrem = new THREE.PMREMGenerator(renderer)
    pmrem.compileEquirectangularShader()
    const hdrLoader = new RGBELoader()
    const hdrTexture = await hdrLoader.loadAsync('/models/char-enviorment.hdr')
    const envMap = pmrem.fromEquirectangular(hdrTexture).texture
    scene.environment = envMap
    hdrTexture.dispose()
    pmrem.dispose()
  } catch (e) {
    console.warn('HDR load failed:', e)
  }

  // ── Load character (without animations) ─────────────────────────────
  const gltfLoader = new GLTFLoader()
  const dracoLoader = new DRACOLoader()
  dracoLoader.setDecoderPath('/draco/')
  gltfLoader.setDRACOLoader(dracoLoader)

  try {
    const decrypted = await decryptFile('/models/character.enc', 'MyCharacter12');
    const blobUrl = URL.createObjectURL(new Blob([decrypted]));

    const gltf = await new Promise((resolve, reject) => {
      gltfLoader.load(blobUrl, resolve, undefined, reject)
    })

    const character = gltf.scene

    // Customize materials
    character.traverse((child) => {
      if (child.isMesh) {
        if (child.name === 'BODY.SHIRT') {
          const mat = child.material.clone()
          mat.color = new THREE.Color('#8B4513')
          child.material = mat
        } else if (child.name === 'Pant') {
          const mat = child.material.clone()
          mat.color = new THREE.Color('#1a1a1a')
          child.material = mat
        }
        child.castShadow = true
        child.receiveShadow = true
        child.frustumCulled = true
      }
    })

    // Remove the computer (screenlight object)
    const computerObj = character.getObjectByName('screenlight')
    if (computerObj) {
      computerObj.visible = false
      // optionally remove from scene
      computerObj.parent?.remove(computerObj)
    }

    // Adjust foot positions (keep as original)
    const footR = character.getObjectByName('footR')
    const footL = character.getObjectByName('footL')
    if (footR) footR.position.y = 3.36
    if (footL) footL.position.y = 3.36

    scene.add(character)
    character.rotation.y = Math.PI * 0.2  // slightly turned

    // Store head bone for later if needed (but no animation)
    const headBone = character.getObjectByName('spine006') || null

    // ── NO ANIMATIONS ─────────────────────────────────────────────────
    // We deliberately do NOT create any AnimationMixer or play clips.
    // All animations (intro, typing, blink, eyebrows) are removed.
    mixer = null

    // Clean up
    URL.revokeObjectURL(blobUrl)
    dracoLoader.dispose()
  } catch (err) {
    console.error('Character failed to load:', err)
  }

  // ── Mouse tracking (kept for possible future use, but no head movement) ──
  const onMouseMove = (e) => {
    mouse = {
      x: (e.clientX / window.innerWidth) * 2 - 1,
      y: -(e.clientY / window.innerHeight) * 2 + 1,
    }
  }
  document.addEventListener('mousemove', onMouseMove)
  cleanupFns.push(() => document.removeEventListener('mousemove', onMouseMove))

  // ── Resize handler ──────────────────────────────────────────────────
  const onResize = () => {
    if (!canvasDiv.value || !renderer || !camera) return
    const r = canvasDiv.value.getBoundingClientRect()
    const rw = r.width || window.innerWidth
    const rh = r.height || window.innerHeight
    camera.aspect = rw / rh
    camera.updateProjectionMatrix()
    renderer.setSize(rw, rh)
  }
  window.addEventListener('resize', onResize)
  cleanupFns.push(() => window.removeEventListener('resize', onResize))

  // ── Simple render loop (no animation updates) ──────────────────────
  function animate() {
    animationId = requestAnimationFrame(animate)
    // No mixer update – character is static
    renderer.render(scene, camera)
  }
  animate()
})

onUnmounted(() => {
  if (animationId) cancelAnimationFrame(animationId)
  cleanupFns.forEach(fn => fn())
  scene.clear()
  if (renderer) {
    if (canvasDiv.value && renderer.domElement.parentNode === canvasDiv.value) {
      canvasDiv.value.removeChild(renderer.domElement)
    }
    renderer.dispose()
    renderer = null
  }
})
</script>

<style scoped>
.character-container {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.character-model {
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
  isolation: isolate;
}

.character-rim {
  position: absolute;
  inset: 0;
  pointer-events: none;
  box-shadow: inset 0 0 80px rgba(0, 0, 0, 0.4);
  z-index: 2;
}
</style>