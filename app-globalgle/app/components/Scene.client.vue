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
  
  // Scene.client.vue – corrected decryption
  async function decryptFile(url, password) {
    // 1. fetch encrypted blob
    const response = await fetch(url);
    const encryptedData = await response.arrayBuffer();

    // 2. extract IV (first 16 bytes) and ciphertext
    const iv = new Uint8Array(encryptedData.slice(0, 16));
    const data = encryptedData.slice(16);

    // 3. derive AES key from password (same as decrypt.js)
    const passwordBuffer = new TextEncoder().encode(password);
    const hash = await crypto.subtle.digest('SHA-256', passwordBuffer);
    const rawKey = hash.slice(0, 32); // AES‑256 needs 32 bytes

    const cryptoKey = await crypto.subtle.importKey(
      'raw',
      rawKey,
      { name: 'AES-CBC' },
      false,
      ['decrypt']
    );

    // 4. decrypt
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
  let mixer = null
  let headBone = null
  let screenLight = null
  let mouse = { x: 0, y: 0 }
  const interpolation = { x: 0.18, y: 0.28, zoom: 0.05 }

let targetCameraZ = 24.7
  const scene = new THREE.Scene()
  const clock = new THREE.Clock()
  let camera = null
  
  function setLighting(scene) {
    const ambient = new THREE.AmbientLight(0x00FF88, 0)
    scene.add(ambient)
    const key = new THREE.DirectionalLight(0x00FF88, 0)
    key.position.set(5, 5, 5)
    scene.add(key)
    const fill = new THREE.DirectionalLight(0x00FF88, 0)
    fill.position.set(-5, 0, 3)
    scene.add(fill)
    const pointLight = new THREE.PointLight(0x00FF88, 0, 10)
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
    const isMobile = w < 768
camera = new THREE.PerspectiveCamera(45, w/h, 0.1, 1000)
camera.position.set(isMobile ? 0 : 2, 11, isMobile ? 32 : 34)
camera.zoom = isMobile ? 1.0 : 1.0
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
      const decrypted = await decryptFile('/models/character.enc', 'MyCharacter12');
// --- diagnostic ---
const magic = new Uint8Array(decrypted, 0, 4);
console.log('First 4 bytes:', Array.from(magic).map(b => b.toString(16)).join(' '));
// GLB magic should be '67 6C 54 46' (hex) which is 'glTF' in ASCII
const magicString = String.fromCharCode(...magic);
console.log('Magic string:', magicString);
if (magicString !== 'glTF') {
  console.error('Decryption failed – not a valid GLB file');
}
// -----------------
const blobUrl = URL.createObjectURL(new Blob([decrypted]));
  
      const gltf = await new Promise((resolve, reject) => {
        gltfLoader.load(blobUrl, resolve, undefined, reject)
      })
  
      const character = gltf.scene
  
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
      character.rotation.y = Math.PI * 0.25
      headBone    = character.getObjectByName('spine006') || null
      screenLight = character.getObjectByName('screenlight') || null
  
      if (gltf.animations?.length) {
  mixer = new THREE.AnimationMixer(character)

  // ── Intro animation — plays once then fades out ───────────────────────
  const introClip = gltf.animations.find(c => c.name === 'introAnimation')
  let introAction = null
  if (introClip) {
    introAction = mixer.clipAction(introClip)
    introAction.setLoop(THREE.LoopOnce, 1)
    introAction.clampWhenFinished = true
    introAction.play()
    // Fade it out once it finishes so the typing animation takes full effect
    mixer.addEventListener('finished', (e) => {
      if (e.action === introAction) introAction.fadeOut(1.2)
    })
  }

  // ── Typing key animations ─────────────────────────────────────────────
  const keyClips = ['key1', 'key2', 'key5', 'key6']
  keyClips.forEach(name => {
    const clip = THREE.AnimationClip.findByName(gltf.animations, name)
    if (clip) {
      const action = mixer.clipAction(clip)
      action.setLoop(THREE.LoopRepeat, Infinity)
      action.play()
      action.timeScale = 1.2
    }
  })

  // ── Typing bone animation ─────────────────────────────────────────────
  const typingBoneNames = [
    'indexFinger1_R','indexFinger2_R','indexFinger3_R',
    'middleFinger1_R','middleFinger2_R','middleFinger3_R',
    'indexFinger1_L','indexFinger2_L','indexFinger3_L',
    'middleFinger1_L','middleFinger2_L','middleFinger3_L',
    'hand_R','hand_L',
  ]
  const typingClip = THREE.AnimationClip.findByName(gltf.animations, 'typing')
  if (typingClip) {
    const filteredTracks = typingClip.tracks.filter(track =>
      typingBoneNames.some(bone => track.name.includes(bone))
    )
    // Fall back to full clip if bone name filter yields nothing
    const clip = filteredTracks.length > 0
      ? new THREE.AnimationClip('typing_filtered', typingClip.duration, filteredTracks)
      : typingClip
    const typingAction = mixer.clipAction(clip)
    typingAction.setLoop(THREE.LoopRepeat, Infinity)
    typingAction.setEffectiveWeight(1)
    typingAction.enabled = true
    typingAction.play()
    typingAction.timeScale = 1.2
  }

  // ── Blink once intro has settled ─────────────────────────────────────
  setTimeout(() => {
    const blinkClip = gltf.animations.find(c => c.name === 'Blink')
    if (blinkClip) {
      mixer.clipAction(blinkClip).play().fadeIn(0.5)
    }
  }, 5000)

  // ── Eyebrow hover on mouse enter/leave ────────────────────────────────
  const eyebrowBoneNames = [
    'browInnerUp_L','browInnerUp_R',
    'browDown_L','browDown_R',
    'browOuterUp_L','browOuterUp_R',
  ]
  const browClip = THREE.AnimationClip.findByName(gltf.animations, 'browup')
  if (browClip) {
    const filteredTracks = browClip.tracks.filter(track =>
      eyebrowBoneNames.some(bone => track.name.includes(bone))
    )
    const filteredBrowClip = new THREE.AnimationClip('browup_filtered', browClip.duration, filteredTracks)
    const browAction = mixer.clipAction(filteredBrowClip)
    browAction.setLoop(THREE.LoopOnce, 1)
    browAction.clampWhenFinished = true
    browAction.enabled = true

    let isHovering = false
    const onEnter = () => {
      if (!isHovering) {
        isHovering = true
        browAction.reset()
        browAction.enabled = true
        browAction.setEffectiveWeight(4)
        browAction.fadeIn(0.5).play()
      }
    }
    const onLeave = () => {
      if (isHovering) {
        isHovering = false
        browAction.fadeOut(0.6)
      }
    }
    canvasDiv.value.addEventListener('mouseenter', onEnter)
    canvasDiv.value.addEventListener('mouseleave', onLeave)
    cleanupFns.push(() => {
      canvasDiv.value?.removeEventListener('mouseenter', onEnter)
      canvasDiv.value?.removeEventListener('mouseleave', onLeave)
    })
  }
}
  
      URL.revokeObjectURL(blobUrl)
      dracoLoader.dispose()
      setTimeout(() => light.turnOnLights(), 2500)
    } catch (err) {
      console.error('Character failed to load:', err)
    }
  
  
    // ── Resize ───────────────────────────────────────────────────────────────────
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
      const delta = clock.getDelta()
      if (mixer) mixer.update(delta)
      if (headBone) {
        headBone.rotation.y = THREE.MathUtils.lerp(headBone.rotation.y, mouse.x * 0.4, interpolation.x)
        headBone.rotation.x = THREE.MathUtils.lerp(headBone.rotation.x, mouse.y * 0.2, interpolation.y)
        light.setPointLight(screenLight)
      }
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