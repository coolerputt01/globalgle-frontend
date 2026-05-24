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
  
  async function decryptFile(url, key) {
    const res = await fetch(url)
    const encryptedBuffer = await res.arrayBuffer()
    const keyMaterial = await crypto.subtle.importKey(
      'raw',
      new TextEncoder().encode(key),
      { name: 'PBKDF2' },
      false,
      ['deriveKey']
    )
    const cryptoKey = await crypto.subtle.deriveKey(
      { name: 'PBKDF2', salt: new TextEncoder().encode('salt'), iterations: 100000, hash: 'SHA-256' },
      keyMaterial,
      { name: 'AES-GCM', length: 256 },
      false,
      ['decrypt']
    )
    const iv = encryptedBuffer.slice(0, 12)
    const ciphertext = encryptedBuffer.slice(12)
    const decrypted = await crypto.subtle.decrypt({ name: 'AES-GCM', iv }, cryptoKey, ciphertext)
    return decrypted
  }
  
  const canvasDiv = ref(null)
  
  let renderer = null
  let animationId = null
  let mixer = null
  let headBone = null
  let screenLight = null
  let mouse = { x: 0, y: 0 }
  const interpolation = { x: 0.1, y: 0.2 }
  const scene = new THREE.Scene()
  const clock = new THREE.Clock()
  let camera = null
  
  function setLighting(scene) {
    const ambient = new THREE.AmbientLight(0xffffff, 0)
    scene.add(ambient)
    const key = new THREE.DirectionalLight(0xffffff, 0)
    key.position.set(5, 5, 5)
    scene.add(key)
    const fill = new THREE.DirectionalLight(0x8888ff, 0)
    fill.position.set(-5, 0, 3)
    scene.add(fill)
    const pointLight = new THREE.PointLight(0x4488ff, 0, 10)
    scene.add(pointLight)
  
    function turnOnLights() {
      ambient.intensity = 0.6
      key.intensity = 1.2
      fill.intensity = 0.4
    }
    function setPointLight(obj) {
      if (obj) {
        const pos = new THREE.Vector3()
        obj.getWorldPosition(pos)
        pointLight.position.copy(pos)
        pointLight.intensity = 1.5
      }
    }
    return { turnOnLights, setPointLight }
  }
  
  const cleanupFns = []
  
  onMounted(async () => {
    if (!canvasDiv.value) return
  
    // Wait a tick so the DOM has settled and getBoundingClientRect is accurate
    await new Promise(r => setTimeout(r, 50))
  
    const rect = canvasDiv.value.getBoundingClientRect()
    const w = rect.width  || window.innerWidth
    const h = rect.height || window.innerHeight
  
    // ── Renderer ────────────────────────────────────────────────────────────────
    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true })
    renderer.setSize(w, h)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
    renderer.toneMapping = THREE.ACESFilmicToneMapping
    renderer.toneMappingExposure = 1
    // Make sure the canvas sits at the back of the container stacking context
    renderer.domElement.style.position = 'absolute'
    renderer.domElement.style.inset = '0'
    renderer.domElement.style.zIndex = '1'
    canvasDiv.value.appendChild(renderer.domElement)
  
    // ── Camera ──────────────────────────────────────────────────────────────────
    camera = new THREE.PerspectiveCamera(14.5, w / h, 0.1, 1000)
    camera.position.set(0, 13.1, 24.7)
    camera.zoom = 1.1
    camera.updateProjectionMatrix()
  
    // ── Lighting ─────────────────────────────────────────────────────────────────
    const light = setLighting(scene)
  
    // ── HDR Environment ──────────────────────────────────────────────────────────
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
      console.warn('HDR load failed, continuing without env map:', e)
    }
  
    // ── Character ────────────────────────────────────────────────────────────────
    const gltfLoader = new GLTFLoader()
    const dracoLoader = new DRACOLoader()
    dracoLoader.setDecoderPath('/draco/')
    gltfLoader.setDRACOLoader(dracoLoader)
  
    try {
      const decrypted = await decryptFile('/models/character.enc', 'MyCharacter12')
      const blobUrl = URL.createObjectURL(new Blob([decrypted]))
  
      const gltf = await new Promise((resolve, reject) => {
        gltfLoader.load(blobUrl, resolve, undefined, reject)
      })
  
      const character = gltf.scene
      await renderer.compileAsync(character, camera, scene)
  
      character.traverse((child) => {
        if (child.isMesh) {
          if (child.name === 'BODY.SHIRT') {
            const mat = child.material.clone()
            mat.color = new THREE.Color('#8B4513')
            child.material = mat
          } else if (child.name === 'Pant') {
            const mat = child.material.clone()
            mat.color = new THREE.Color('#000000')
            child.material = mat
          }
          child.castShadow = true
          child.receiveShadow = true
          child.frustumCulled = true
        }
      })
  
      const footR = character.getObjectByName('footR')
      const footL = character.getObjectByName('footL')
      if (footR) footR.position.y = 3.36
      if (footL) footL.position.y = 3.36
  
      scene.add(character)
      headBone    = character.getObjectByName('spine006') || null
      screenLight = character.getObjectByName('screenlight') || null
  
      if (gltf.animations?.length) {
        mixer = new THREE.AnimationMixer(character)
        mixer.clipAction(gltf.animations[0]).play()
      }
  
      URL.revokeObjectURL(blobUrl)
      dracoLoader.dispose()
      setTimeout(() => light.turnOnLights(), 2500)
    } catch (err) {
      console.error('Character failed to load:', err)
    }
  
    // ── Mouse tracking ───────────────────────────────────────────────────────────
    const onMouseMove = (e) => {
      mouse = {
        x:  (e.clientX / window.innerWidth)  * 2 - 1,
        y: -(e.clientY / window.innerHeight) * 2 + 1,
      }
    }
    document.addEventListener('mousemove', onMouseMove)
    cleanupFns.push(() => document.removeEventListener('mousemove', onMouseMove))
  
    // ── Resize ───────────────────────────────────────────────────────────────────
    const onResize = () => {
      if (!canvasDiv.value || !renderer || !camera) return
      const r = canvasDiv.value.getBoundingClientRect()
      const rw = r.width  || window.innerWidth
      const rh = r.height || window.innerHeight
      camera.aspect = rw / rh
      camera.updateProjectionMatrix()
      renderer.setSize(rw, rh)
    }
    window.addEventListener('resize', onResize)
    cleanupFns.push(() => window.removeEventListener('resize', onResize))
  
    // ── Render loop ──────────────────────────────────────────────────────────────
    function animate() {
      animationId = requestAnimationFrame(animate)
      if (headBone) {
        headBone.rotation.y = THREE.MathUtils.lerp(headBone.rotation.y, mouse.x * 0.4, interpolation.x)
        headBone.rotation.x = THREE.MathUtils.lerp(headBone.rotation.x, mouse.y * 0.2, interpolation.y)
        light.setPointLight(screenLight)
      }
      const delta = clock.getDelta()
      if (mixer) mixer.update(delta)
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
    /* establish stacking context so z-index on canvas works */
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