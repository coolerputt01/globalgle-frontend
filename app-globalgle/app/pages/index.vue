<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import Earth from '~/components/Earth.client.vue'
import Scene from '~/components/Scene.client.vue'

const threeEarth = ref(null)
const cleanup = []


onMounted(async () => {
  const { gsap } = await import('gsap')
  const { ScrollTrigger } = await import('gsap/ScrollTrigger')
  gsap.registerPlugin(ScrollTrigger)

  


  // ─── NAVBAR ────────────────────────────────────────────────────────────────
  let lastScroll = 0
  const navbar = document.getElementById('navbar')
  const onScroll = () => {
    const cur = window.scrollY
    if (cur > 80) {
      navbar.classList.add('scrolled')
    } else {
      navbar.classList.remove('scrolled')
    }
    lastScroll = cur
  }
  window.addEventListener('scroll', onScroll, { passive: true })
  cleanup.push(() => window.removeEventListener('scroll', onScroll))

  // ─── HERO ─────────────────────────────────────────────────────────────────
  ScrollTrigger.create({
    trigger: '#hero',
    start: 'top top',
    end: 'bottom top',
    scrub: 1,
    onUpdate: (self) => {
  if (!threeEarth.value) return
  const p = self.progress

  // fade out
  const earthCanvas = document.querySelector('.three-canvas')
  if (earthCanvas) {
    const op = Math.max(0, 1 - p * 3)
    earthCanvas.style.opacity = op
    earthCanvas.style.visibility = op === 0 ? 'hidden' : 'visible'
    earthCanvas.style.pointerEvents = op === 0 ? 'none' : 'auto'
  }

  // stop doing anything once faded
  if (p > 0.35) return

  if (threeEarth.value.setScrollProgress) {
    threeEarth.value.setScrollProgress(p)
  }

  const heroContent = document.querySelector('.hero-content')
  if (heroContent) gsap.set(heroContent, { opacity: 1 - p * 2.2 })
}
  })

  gsap.to('#hero-title', { opacity: 1, scale: 1, duration: 1.4, ease: 'power3.out', delay: 0.4 })
  gsap.to('.hero-btns', { opacity: 1, duration: 1, ease: 'power2.out', delay: 1.1 })

  // ─── G SECTION ─────────────────────────────────────────────────────────────
  const gTl = gsap.timeline({ paused: true })
  gTl.fromTo('#big-g',
    { opacity: 0, scale: 0.4, y: 80 },
    { opacity: 1, scale: 1, y: 0, duration: 0.4, ease: 'back.out(2)' }, 0)
  gTl.to('#big-g', { y: -120, scale: 1.6, duration: 0.35, ease: 'power2.inOut' }, 0.4)
  gTl.fromTo('#circle-text-svg',
    { opacity: 0 },
    { opacity: 1, duration: 0.45, ease: 'power2.out' }, 0.55)

  // CHANGE 4: Use the taller hero scroll range to drive the G animation,
  // giving the G section full play time while hero fades
  ScrollTrigger.create({
    trigger: '#hero',
    start: 'top top',
    end: 'bottom top',
    scrub: 1.2,
    onUpdate(self) { gTl.progress(self.progress) },
  })

  ScrollTrigger.create({
    trigger: '#numbers-section',
    start: 'top bottom',
    end: 'top top',
    scrub: 1,
    onUpdate(self) { gsap.set('#g-section', { opacity: 1 - self.progress }) },
  })

  document.getElementById('circle-text-svg').style.animation = 'spinCircle 22s linear infinite'

// ─── NUMBERS ───────────────────────────────────────────────────────────────
const numberItems = gsap.utils.toArray('.number-item')

numberItems.forEach((el, i) => {
  ScrollTrigger.create({
    trigger: '#numbers-section',
    start: 'top 80%',
    end: 'bottom 20%',
    scrub: 1.5,
    onUpdate(self) {
      const p = self.progress
      const stagger = 0.07
      const duration = 0.28

      const itemStart = i * stagger
      const itemPeak  = itemStart + duration
      const itemEnd   = itemPeak + duration

      let local = 0
      if (p < itemStart) {
        local = 0
      } else if (p < itemPeak) {
        local = (p - itemStart) / duration
      } else if (p < itemEnd) {
        local = 1 - (p - itemPeak) / duration
      }

      if (p < itemPeak) {
        // ENTERING: from bottom-right diagonal
        gsap.set(el, {
          opacity: local,
          x: (1 - local) * 200,   // from right
          y: (1 - local) * 200,   // from bottom
        })
      } else {
        // EXITING: to top-left diagonal
        gsap.set(el, {
          opacity: local,
          x: -(1 - local) * 200,  // to left
          y: -(1 - local) * 200,  // to top
        })
      }
    },
  })
})

// Number layer parallax — gentle drift
gsap.utils.toArray('.number-layer').forEach((el, i) => {
  ScrollTrigger.create({
    trigger: '#numbers-section',
    start: 'top bottom',
    end: 'bottom top',
    scrub: 1,
    onUpdate(self) {
      gsap.set(el, { x: (i % 2 === 0 ? -1 : 1) * 30 * self.progress })
    },
  })
})

  // ─── SPIRAL ────────────────────────────────────────────────────────────────
  ;(function () {
    const svg = document.getElementById('spiral-svg')
    const pathEl = document.getElementById('spiral-path')
    const dotEl = document.getElementById('spiral-dot')

  function buildSpiralPath() {
  const W = svg.clientWidth || window.innerWidth
  // Reach into the bigtext section below
  const bigtextSection = document.getElementById('bigtext-section')
  const spiralSection  = document.getElementById('spiral-section')
  const extraReach = bigtextSection ? bigtextSection.offsetTop - spiralSection.offsetTop + bigtextSection.offsetHeight * 0.5 : 0
  const H = Math.max(spiralSection.offsetHeight, extraReach)

  const cx = W * 0.5
  const points = []

  for (let i = 0; i <= 300; i++) {
    const t = i / 300
    // Smooth spiral — no sharp initial jump, eases into curves
    const angle = -Math.PI / 2 + t * Math.PI * 7
    const radius = (80 + 120 * (1 - t * 0.6)) * Math.sin(t * Math.PI * 0.5 + 0.1)
    points.push([
      cx + radius * Math.cos(angle),
      H * 0.04 + t * H * 0.96 + radius * 0.18 * Math.sin(angle * 0.5),
    ])
  }

  const d = 'M ' + points.map(p => p[0].toFixed(1) + ' ' + p[1].toFixed(1)).join(' L ')
  pathEl.setAttribute('d', d)

  // Make the svg tall enough to reach bigtext
  svg.style.height = H + 'px'
  return pathEl.getTotalLength()
}

    let len = buildSpiralPath()
    pathEl.style.strokeDasharray = len
    pathEl.style.strokeDashoffset = len
    dotEl.setAttribute('opacity', '1')

    ScrollTrigger.create({
      trigger: '#spiral-section',
      start: 'top bottom',
      end: 'bottom top',
      scrub: 1,
      onUpdate(self) {
        pathEl.style.strokeDashoffset = len * (1 - self.progress)
        const pt = pathEl.getPointAtLength(len * self.progress)
        dotEl.setAttribute('cx', pt.x)
        dotEl.setAttribute('cy', pt.y)
      },
    })

    window.addEventListener('resize', () => { len = buildSpiralPath() })
  })()

  gsap.utils.toArray('.spiral-para').forEach((el) => {
    ScrollTrigger.create({
      trigger: el,
      start: 'top 85%',
      end: 'top 40%',
      scrub: 1,
      onUpdate(self) { gsap.set(el, { opacity: self.progress, y: (1 - self.progress) * 24 }) },
    })
  })

  // ─── BIG TEXT ──────────────────────────────────────────────────────────────
  ScrollTrigger.create({
    trigger: '#bigtext-section',
    start: 'top 80%',
    end: 'top 30%',
    scrub: 1,
    onUpdate(self) { gsap.set('#big-header', { opacity: self.progress, y: (1 - self.progress) * 40 }) },
  })
  ScrollTrigger.create({
    trigger: '#bigtext-section',
    start: 'top 75%',
    end: 'top 25%',
    scrub: 1,
    onUpdate(self) { gsap.set('.desc', { opacity: self.progress, y: (1 - self.progress) * 24 }) },
  })

  // ─── RUBIK ─────────────────────────────────────────────────────────────────
  ;(function () {
    const canvas = document.getElementById('rubik-canvas')
  const ctx = canvas.getContext('2d')
  canvas.width = 420
  canvas.height = 420
  let rotX = 0.42, autoRot = 0, rafId

    const FACE_COLORS = {
      front:  ['#7c3aed','#3b82f6','#00ff88','#7c3aed','#00ff88','#3b82f6','#00ff88','#3b82f6','#7c3aed'],
      back:   ['#3b82f6','#7c3aed','#00ff88','#00ff88','#3b82f6','#7c3aed','#7c3aed','#00ff88','#3b82f6'],
      top:    ['#00ff88','#7c3aed','#3b82f6','#3b82f6','#00ff88','#7c3aed','#7c3aed','#3b82f6','#00ff88'],
      bottom: ['#7c3aed','#3b82f6','#00ff88','#00ff88','#7c3aed','#3b82f6','#3b82f6','#00ff88','#7c3aed'],
      left:   ['#3b82f6','#00ff88','#7c3aed','#7c3aed','#3b82f6','#00ff88','#00ff88','#7c3aed','#3b82f6'],
      right:  ['#00ff88','#7c3aed','#3b82f6','#3b82f6','#00ff88','#7c3aed','#7c3aed','#3b82f6','#00ff88'],
    }

    const ICONS = {
      bitcoin(c, x, y, s) {
        c.save(); c.translate(x, y)
        c.beginPath(); c.arc(0, 0, s * 0.35, 0, Math.PI * 2)
        c.fillStyle = 'rgba(255,255,255,0.85)'; c.fill()
        c.fillStyle = '#f7931a'; c.font = `bold ${s * 0.38}px serif`
        c.textAlign = 'center'; c.textBaseline = 'middle'; c.fillText('₿', 0, 1); c.restore()
      },
      crypto(c, x, y, s) {
        c.save(); c.translate(x, y)
        c.beginPath(); c.arc(0, 0, s * 0.35, 0, Math.PI * 2)
        c.fillStyle = 'rgba(255,255,255,0.85)'; c.fill()
        c.fillStyle = '#627eea'; c.font = `bold ${s * 0.36}px serif`
        c.textAlign = 'center'; c.textBaseline = 'middle'; c.fillText('Ξ', 0, 0); c.restore()
      },
      call(c, x, y, s) {
        c.save(); c.translate(x, y)
        c.beginPath(); c.arc(0, 0, s * 0.35, 0, Math.PI * 2)
        c.fillStyle = 'rgba(255,255,255,0.85)'; c.fill()
        c.strokeStyle = '#00c853'; c.lineWidth = s * 0.07; c.lineCap = 'round'
        c.beginPath(); c.arc(-s * 0.06, -s * 0.06, s * 0.13, Math.PI, 0); c.stroke()
        c.beginPath(); c.moveTo(-s*.19,-s*.06); c.lineTo(-s*.22,s*.08); c.lineTo(-s*.12,s*.18); c.stroke()
        c.beginPath(); c.moveTo(s*.07,-s*.06); c.lineTo(s*.2,0); c.lineTo(s*.12,s*.18); c.stroke()
        c.restore()
      },
    }
    const ICON_KEYS = ['bitcoin', 'crypto', 'call']

    function mat4() { return new Float32Array([1,0,0,0,0,1,0,0,0,0,1,0,0,0,0,1]) }
    function mul(a, b) {
      const r = mat4()
      for (let i=0;i<4;i++) for (let j=0;j<4;j++) { r[i*4+j]=0; for(let k=0;k<4;k++) r[i*4+j]+=a[i*4+k]*b[k*4+j] }
      return r
    }
    function rx(a){const c=Math.cos(a),s=Math.sin(a),m=mat4();m[5]=c;m[6]=-s;m[9]=s;m[10]=c;return m}
    function ry(a){const c=Math.cos(a),s=Math.sin(a),m=mat4();m[0]=c;m[2]=s;m[8]=-s;m[10]=c;return m}
    function proj(x,y,z,mat,ox,oy,sc){
      const tx=mat[0]*x+mat[4]*y+mat[8]*z
      const ty=mat[1]*x+mat[5]*y+mat[9]*z
      const tz=mat[2]*x+mat[6]*y+mat[10]*z
      const d=4/(4+tz*0.6)
      return[ox+tx*sc*d, oy+ty*sc*d, tz]
    }

    function drawFace(pts, colors) {
      const s = Math.sqrt((pts[1][0]-pts[0][0])**2+(pts[1][1]-pts[0][1])**2)*0.31
      for (let i=0;i<9;i++){
        const row=Math.floor(i/3),col=i%3,t=1/3
        const mx=pts[0][0]+(pts[1][0]-pts[0][0])*(col*t+t*.5)+(pts[3][0]-pts[0][0])*(row*t+t*.5)
        const my=pts[0][1]+(pts[1][1]-pts[0][1])*(col*t+t*.5)+(pts[3][1]-pts[0][1])*(row*t+t*.5)
        const sx=(pts[1][0]-pts[0][0])*t*.88, sy=(pts[1][1]-pts[0][1])*t*.88
        const rx_=(pts[3][0]-pts[0][0])*t*.88, ry_=(pts[3][1]-pts[0][1])*t*.88
        ctx.beginPath()
        ctx.moveTo(mx-sx/2-rx_/2,my-sy/2-ry_/2)
        ctx.lineTo(mx+sx/2-rx_/2,my+sy/2-ry_/2)
        ctx.lineTo(mx+sx/2+rx_/2,my+sy/2+ry_/2)
        ctx.lineTo(mx-sx/2+rx_/2,my-sy/2+ry_/2)
        ctx.closePath()
        ctx.fillStyle=colors[i]; ctx.fill()
        ctx.strokeStyle='rgba(0,0,0,0.5)'; ctx.lineWidth=1; ctx.stroke()
        ICONS[ICON_KEYS[i%3]](ctx,mx,my,s)
      }
    }

    function normal(pts){
      return (pts[1][0]-pts[0][0])*(pts[3][1]-pts[0][1])-(pts[1][1]-pts[0][1])*(pts[3][0]-pts[0][0])
    }

    function drawCube() {
      ctx.clearRect(0,0,420,420)
      const mat=mul(rx(rotX),ry(autoRot))
      const ox=210,oy=210,sc=130,sz=0.72
      const verts=[[-sz,-sz,-sz],[sz,-sz,-sz],[sz,sz,-sz],[-sz,sz,-sz],[-sz,-sz,sz],[sz,-sz,sz],[sz,sz,sz],[-sz,sz,sz]]
      const p=verts.map(v=>proj(v[0],v[1],v[2],mat,ox,oy,sc))
      const faces=[
        {pts:[p[4],p[5],p[6],p[7]],key:'front', z:(p[4][2]+p[5][2]+p[6][2]+p[7][2])/4},
        {pts:[p[0],p[1],p[2],p[3]],key:'back',  z:(p[0][2]+p[1][2]+p[2][2]+p[3][2])/4},
        {pts:[p[3],p[2],p[6],p[7]],key:'bottom',z:(p[3][2]+p[2][2]+p[6][2]+p[7][2])/4},
        {pts:[p[0],p[1],p[5],p[4]],key:'top',   z:(p[0][2]+p[1][2]+p[5][2]+p[4][2])/4},
        {pts:[p[1],p[2],p[6],p[5]],key:'right', z:(p[1][2]+p[2][2]+p[6][2]+p[5][2])/4},
        {pts:[p[0],p[3],p[7],p[4]],key:'left',  z:(p[0][2]+p[3][2]+p[7][2]+p[4][2])/4},
      ]
      faces.sort((a,b)=>a.z-b.z)
      faces.forEach(f => { drawFace(f.pts, FACE_COLORS[f.key]) })
    }
    function loop(){autoRot+=0.030; rotX=0.42+Math.sin(Date.now()/2800)*0.22; drawCube(); rafId=requestAnimationFrame(loop)}
  loop()
  cleanup.push(()=>cancelAnimationFrame(rafId))

  const section = document.getElementById('rubik-section')
  const sectionW = () => section.offsetWidth

ScrollTrigger.create({
  trigger: '#rubik-section',
  start: 'top bottom',
  end: 'bottom top',
  scrub: 1,
  onUpdate(self) {
    const p = self.progress
    const W = sectionW()
    const center = (W - 420) / 2
    const amplitude = (W * 0.28) * (1 - p * 0.8)
    const sineX = Math.sin(p * Math.PI * 2) * amplitude;
    const x = center + sineX
    const y = -320 + p * 400
    canvas.style.left = x + 'px'
    canvas.style.top  = y + 'px'
  },
})

// 2. Fade out the cube during the last part of its journey
ScrollTrigger.create({
  trigger: '#rubik-section',
  start: '60% bottom',
  end: 'bottom top',
  scrub: 1,
  onUpdate(self) {
    // Starts fading when scroll is beyond 60% of the section
    const p = Math.min(1, Math.max(0, (self.progress - 0.6) / 0.4))
    canvas.style.opacity = 1 - p
  },
})

ScrollTrigger.create({
  trigger: '#rubik-section',
  start: '33% 60%',
  end: '40% 40%',
  scrub: 1,
  onUpdate(self) {
    gsap.set('#rt-left', { opacity: 1 - self.progress })
  },
})

// Text 2 — right side, cube at center
ScrollTrigger.create({
  trigger: '#rubik-section',
  start: '35% 60%',
  end: '50% 40%',
  scrub: 1,
  onUpdate(self) {
    gsap.set('#rt-right', { opacity: self.progress, x: (1 - self.progress) * 60 })
  },
})

// Text 2 — fade out as cube exits
ScrollTrigger.create({
  trigger: '#rubik-section',
  start: '60% 60%',
  end: '70% 40%',
  scrub: 1,
  onUpdate(self) {
    gsap.set('#rt-right', { opacity: 1 - self.progress })
  },
})

// Text 3 — left side, cube exiting
ScrollTrigger.create({
  trigger: '#rubik-section',
  start: '65% 60%',
  end: '85% 40%',
  scrub: 1,
  onUpdate(self) {
    gsap.set('#rt-bottom', { opacity: self.progress, x: (1 - self.progress) * 80 })
  },
})

ScrollTrigger.create({
  trigger: '#rubik-section',
  start: '70% 60%',
  end: '100% 40%',
  scrub: 1,
  onUpdate(self) {
    gsap.set('#rt-bottom', { opacity: self.progress, x: (1 - self.progress) * 80 })
  },
})
  })()

  // ─── FAQ ───────────────────────────────────────────────────────────────────
  ScrollTrigger.create({
    trigger: '#faq-section',
    start: 'top 80%',
    end: 'top 45%',
    scrub: 1,
    onUpdate(self) { gsap.set('#faq-label', { opacity: self.progress, y: (1 - self.progress) * 20 }) },
  })
  ScrollTrigger.create({
    trigger: '#faq-section',
    start: 'top 78%',
    end: 'top 40%',
    scrub: 1,
    onUpdate(self) { gsap.set('#faq-head', { opacity: self.progress, y: (1 - self.progress) * 30 }) },
  })

  gsap.utils.toArray('.faq-item').forEach((el, i) => {
    ScrollTrigger.create({
      trigger: '#faq-list',
      start: 'top 85%',
      end: 'top 30%',
      scrub: 1,
      onUpdate(self) {
        const s = i * 0.12
        const p = Math.min(Math.max((self.progress - s) / 0.45, 0), 1)
        gsap.set(el, { opacity: p, y: (1 - p) * 20 })
      },
    })

    const btn = el.querySelector('.faq-q')
    const handler = () => {
      const isOpen = el.classList.contains('open')
      document.querySelectorAll('.faq-item').forEach(x => x.classList.remove('open'))
      if (!isOpen) el.classList.add('open')
    }
    btn.addEventListener('click', handler)
    cleanup.push(() => btn.removeEventListener('click', handler))
  })

  // ─── GLE ───────────────────────────────────────────────────────────────────
  gsap.utils.toArray('.gle-word').forEach((el, i) => {
    ScrollTrigger.create({
      trigger: '#gle-section',
      start: 'top 85%',
      end: 'top 40%',
      scrub: 1,
      onUpdate(self) {
        const s = i * 0.2
        const p = Math.min(Math.max((self.progress - s) / 0.6, 0), 1)
        gsap.set(el, { opacity: p, y: (1 - p) * 60 })
      },
    })
    const t = gsap.to(el, { y: -28, duration: 0.55 + i * 0.08, ease: 'power2.inOut', yoyo: true, repeat: -1, delay: i * 0.22 })
    cleanup.push(() => t.kill())
  })

  // ─── W + FOOTER REVEAL ─────────────────────────────────────────────────────
  ScrollTrigger.create({
    trigger: '#gle-section',
    start: 'top 70%',
    end: 'bottom 30%',
    scrub: 1,
    onUpdate(self) {
      gsap.set('#w-overlay', { opacity: self.progress, scale: 0.3 + self.progress * 2.7 })
    },
  })

  ScrollTrigger.create({
    trigger: '#footer',
    start: 'top 95%',
    end: 'top 0%',
    scrub: 1.2,
    onUpdate(self) {
      const p = self.progress
      const r = p * 160
      gsap.set('#footer', {
        clipPath: `circle(${r}% at 50% 0%)`,
        opacity: p > 0 ? 1 : 0,
      })
      gsap.set('#w-overlay', { opacity: Math.max(0, 1 - p * 2.5) })
    },
  })
})

onUnmounted(() => {
  cleanup.forEach(fn => fn())
  cleanup.length = 0
})
</script>

<template>
  <main>
    <!-- ── NAVBAR ─────────────────────────────────────────────────────────── -->
    <NavBar/>

    <!-- HERO: CHANGE 1 — background image placeholder + taller section -->
    <section id="hero">
      <!-- 🖼️ ADD YOUR IMAGE PATH HERE -->
      <div class="hero-bg-image" style="background-image: url('/bg_image.PNG')"></div>
      <div class="hero-bg-overlay"></div>

      <ClientOnly>
        <Earth ref="threeEarth" />
      </ClientOnly>
      <div class="hero-content">
        <div class="hero-btns">
          <button class="btn-start">Get Started</button>
          <button class="btn-login">Login</button>
        </div>
        <h1 id="hero-title">GlobalGLE</h1>
      </div>
    </section>

    <!-- G SECTION: CHANGE 4 — taller so animation has full room to breathe -->
    <section id="g-section">
      <div id="big-g">G</div>
      <svg id="circle-text-svg" viewBox="0 0 500 500" xmlns="http://www.w3.org/2000/svg">
        <defs>
          <path id="circle-path" d="M250,250 m-190,0 a190,190 0 1,1 380,0 a190,190 0 1,1 -380,0" />
        </defs>
        <text font-family="Urbanist,sans-serif" font-size="28" font-weight="700"
              fill="rgba(255,255,255,0.8)" letter-spacing="12">
          <textPath href="#circle-path">GLOBALGLE · GLOBALGLE · GLOBALGLE ·</textPath>
        </text>
      </svg>
    </section>

    <!-- NUMBERS: CHANGE 3 — wrapped in screen/box container -->
    <section id="numbers-section">
      <div class="numbers-screen">
        <!-- Decorative screen chrome -->
        <div class="screen-chrome">
          <span class="chrome-dot"></span>
          <span class="chrome-dot"></span>
          <span class="chrome-dot"></span>
        </div>
        <div class="number-layer nl-top">
          <span class="number-item">$2.4B+</span>
          <span class="number-item">120+</span>
          <span class="number-item">40+</span>
          <span class="number-item">24/7</span>
        </div>
        <div class="number-layer nl-mid">
          <span class="number-item">120+</span>
          <span class="number-item">24/7</span>
          <span class="number-item">$2.4B+</span>
        </div>
        <div class="number-layer nl-bot">
          <span class="number-item">24/7</span>
          <span class="number-item">120+</span>
          <span class="number-item">40+</span>
          <span class="number-item">$2.4B+</span>
        </div>
        <!-- Screen corner scan-line overlay -->
        <div class="screen-scanlines" aria-hidden="true"></div>
        <div class="screen-vignette" aria-hidden="true"></div>
      </div>
    </section>

    <!-- SPIRAL + TEXT -->
    <section id="spiral-section">
      <svg id="spiral-svg">
        <path id="spiral-path" fill="none" stroke="#00ff88" stroke-width="2" stroke-linecap="round" />
        <circle id="spiral-dot" r="5" fill="#00ff88" opacity="0" />
      </svg>
      <div class="spiral-content">
        <div class="spiral-para" id="sp1">
          <p>We bridge the gap between fragmented global financial systems and seamless digital infrastructure, enabling real-time settlement for institutions worldwide.</p>
        </div>
        <div class="spiral-para" id="sp2">
          <p>Our proprietary rails connect over forty banking partners across six continents, reducing settlement friction and empowering businesses to move capital at the speed of thought.</p>
        </div>
        <div class="spiral-para" id="sp3">
          <p>Built on decades of cross-border expertise, GlobalGle's platform is designed to meet the complexity of modern finance with uncompromising reliability, security-first engineering, and adaptive compliance layers that evolve with regulation.</p>
        </div>
      </div>
    </section>

    <!-- BIG TEXT -->
    <section id="bigtext-section">
      <h2 id="big-header">GLOBAL GLE is a GLOBAL BUSINESS PARTNER AGENCY</h2>
      <p class="desc">Powering advanced Crypto Funding OTC, and Caller ID  infrastructure for ambitious teams worldwide.</p>
    </section>

    <!-- RUBIK -->
    <section id="rubik-section">
  <div class="rubik-text-left" id="rt-left">
    <h3>Move capital across borders without the friction.</h3>
    <p>Real-time settlement rails connecting institutions across 120+ countries seamlessly.</p>
  </div>
  <div class="rubik-wrap">
  <canvas id="rubik-canvas"></canvas>
</div>
  <div class="rubik-text-right" id="rt-right">
    <h3>Compliance built in, not bolted on.</h3>
    <p>Adaptive regulatory layers that evolve with global standards, automatically.</p>
  </div>
  <div class="rubik-text-bottom" id="rt-bottom">
    <h3>Transparency at every layer.</h3>
    <p>Full auditability and real-time reporting built into every transaction we process.</p>
  </div>
</section>

<div class="scene-wrapper">
  <div class="scene-gle-bg" aria-hidden="true">
    <span>GLE</span>
    <span>GLE</span>
    <span>GLE</span>
  </div>
  <ClientOnly>
    <Scene />
  </ClientOnly>
</div>


    <!-- FAQ -->
    <section id="faq-section">
      <p class="faq-label" id="faq-label">Questions</p>
      <h2 class="faq-header" id="faq-head">What this actually does.</h2>
      <div id="faq-list">
        <div class="faq-item">
          <div class="faq-q">How does GlobalGle process cross-border payments? <span>+</span></div>
          <div class="faq-a">Our platform leverages direct integrations with correspondent banks and local payment schemes, routing transactions through the most efficient path in real time with full auditability.</div>
        </div>
        <div class="faq-item">
          <div class="faq-q">What currencies and corridors are supported? <span>+</span></div>
          <div class="faq-a">We support 85+ currencies across 120+ countries. Our corridor coverage includes major G20 markets as well as underserved emerging market regions with native local settlement.</div>
        </div>
        <div class="faq-item">
          <div class="faq-q">How does compliance and KYC work? <span>+</span></div>
          <div class="faq-a">Compliance is embedded at every layer of our stack. We run real-time sanctions screening, automated KYC workflows, and jurisdiction-specific rule engines that update with regulatory changes automatically.</div>
        </div>
        <div class="faq-item">
          <div class="faq-q">What does integration look like for my team? <span>+</span></div>
          <div class="faq-a">Integration is API-first. Most teams are live within two weeks with our RESTful APIs, webhooks, and SDK libraries for major languages. Dedicated onboarding support is included.</div>
        </div>
        <div class="faq-item">
          <div class="faq-q">How is pricing structured? <span>+</span></div>
          <div class="faq-a">Pricing is volume-based with transparent per-transaction fees. We offer custom enterprise pricing for institutions clearing above $10M monthly. No hidden charges or subscription locks.</div>
        </div>
      </div>
    </section>

    <!-- FOOTER -->
    <footer id="footer">
      <div class="footer-col1">
        <h3>Work with us.</h3>
        <div class="footer-form">
          <div class="footer-fields">
            <div class="footer-input-row">
              <svg width="18" height="18" fill="none" stroke="#fff" stroke-width="1.5" viewBox="0 0 24 24">
                <rect x="2" y="4" width="20" height="16" rx="2"/><path d="m2 7 10 7 10-7"/>
              </svg>
              <input type="email" placeholder="placeholder@email.com"/>
            </div>
            <div class="footer-input-row">
              <svg width="18" height="18" fill="none" stroke="#fff" stroke-width="1.5" viewBox="0 0 24 24">
                <rect x="2" y="3" width="20" height="18" rx="2"/><path d="M16 3v4M8 3v4M2 11h20"/>
              </svg>
              <input type="text" placeholder="Company Name"/>
            </div>
            <div class="footer-input-row">
              <svg width="18" height="18" fill="none" stroke="#fff" stroke-width="1.5" viewBox="0 0 24 24">
                <path d="M22 16.92v3a2 2 0 0 1-2.18 2 19.79 19.79 0 0 1-8.63-3.07A19.5 19.5 0 0 1 4.14 12 19.79 19.79 0 0 1 1 3.18 2 2 0 0 1 3 1h3a2 2 0 0 1 2 1.72A12.84 12.84 0 0 0 9 6.07a2 2 0 0 1-.45 2L7.09 9.5a16 16 0 0 0 7.5 7.5l1.44-1.44a2 2 0 0 1 2-.45 12.84 12.84 0 0 0 3.36.65A2 2 0 0 1 23 17z"/>
              </svg>
              <input type="tel" placeholder="+1 Placeholder Number"/>
            </div>
          </div>
          <button class="footer-submit">Submit</button>
        </div>
        <p class="footer-copy">© 2025 GlobalGle. All rights reserved.</p>
      </div>

      <div class="footer-col2">
        <nav>
          <a href="#">Work</a>
          <a href="#">About</a>
          <a href="#">Services</a>
          <a href="#">Portfolio</a>
          <a href="#">Contact</a>
        </nav>
      </div>

      <div class="footer-col3">
        <div class="social-header">Social</div>
        <div class="social-icons">
          <div class="social-icon">
            <svg viewBox="0 0 24 24"><path d="M12 2.163c3.204 0 3.584.012 4.85.07 3.252.148 4.771 1.691 4.919 4.919.058 1.265.069 1.645.069 4.849 0 3.205-.012 3.584-.069 4.849-.149 3.225-1.664 4.771-4.919 4.919-1.266.058-1.644.07-4.85.07-3.204 0-3.584-.012-4.849-.07-3.26-.149-4.771-1.699-4.919-4.92-.058-1.265-.07-1.644-.07-4.849 0-3.204.013-3.583.07-4.849.149-3.227 1.664-4.771 4.919-4.919 1.266-.057 1.645-.069 4.849-.069zm0-2.163c-3.259 0-3.667.014-4.947.072-4.358.2-6.78 2.618-6.98 6.98-.059 1.281-.073 1.689-.073 4.948 0 3.259.014 3.668.072 4.948.2 4.358 2.618 6.78 6.98 6.98 1.281.058 1.689.072 4.948.072 3.259 0 3.668-.014 4.948-.072 4.354-.2 6.782-2.618 6.979-6.98.059-1.28.073-1.689.073-4.948 0-3.259-.014-3.667-.072-4.947-.196-4.354-2.617-6.78-6.979-6.98-1.281-.059-1.69-.073-4.949-.073zm0 5.838c-3.403 0-6.162 2.759-6.162 6.162s2.759 6.163 6.162 6.163 6.162-2.759 6.162-6.163c0-3.403-2.759-6.162-6.162-6.162zm0 10.162c-2.209 0-4-1.79-4-4 0-2.209 1.791-4 4-4s4 1.791 4 4c0 2.21-1.791 4-4 4zm6.406-11.845c-.796 0-1.441.645-1.441 1.44s.645 1.44 1.441 1.44c.795 0 1.439-.645 1.439-1.44s-.644-1.44-1.439-1.44z"/></svg>
          </div>
          <div class="social-icon">
            <svg viewBox="0 0 24 24"><path d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433c-1.144 0-2.063-.926-2.063-2.065 0-1.138.92-2.063 2.063-2.063 1.14 0 2.064.925 2.064 2.063 0 1.139-.925 2.065-2.064 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"/></svg>
          </div>
        </div>
      </div>

      <div class="footer-bottom-links">
        <a href="#">Privacy Policy</a>
        <a href="#">Terms of Service</a>
      </div>
    </footer>
  </main>
  <div class="star-field" aria-hidden="true"></div>
</template>

<style scoped>
* { margin: 0; padding: 0; box-sizing: border-box; }
html { scroll-behavior: auto; }
:global(body) {
  background: #000;
  color: #fff;
  font-family: 'Urbanist', sans-serif;
  overflow-x:hidden;
}

.star-field {
  position: fixed;
  inset: 0;
  z-index: 60;
  pointer-events: none;
  background-image:
    radial-gradient(circle, rgba(0,255,136,0.55) 1px, transparent 1px),
    radial-gradient(circle, rgba(0,255,136,0.3) 1px, transparent 1px),
    radial-gradient(circle, rgba(0,255,136,0.2) 1px, transparent 1px);
  background-size: 180px 180px, 120px 120px, 300px 300px;
  background-position: 0 0, 60px 90px, 30px 150px;
  animation: starDrift 60s linear infinite;
}

@keyframes starDrift {
  from { background-position: 0 0; }
  to { background-position: 512px 512px; }
}
@keyframes spinCircle {
  from { transform: rotate(0deg); }
  to   { transform: rotate(360deg); }
}

/* ── CHANGE 2: Green stars canvas — fixed, behind everything ──────────────── */
#stars-canvas {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  z-index: 0;
}

:global(#stars-canvas) {
  /* The canvas is drawn via JS below */
}

.rubik-text-bottom {
  position: absolute;
  bottom: 12vh;
  left: 8vw;
  max-width: 280px;
  opacity: 0;
  z-index: 6;
}
.rubik-text-bottom h3 {
  font-family: 'Urbanist', sans-serif;
  font-size: clamp(1.05rem, 2vw, 1.6rem);
  font-weight: 700;
  line-height: 1.35;
}
.rubik-text-bottom p {
  margin-top: 0.75rem;
  color: #6b7280;
  font-size: 0.9rem;
  line-height: 1.6;
}

/* Inline script to generate the stars — placed in a global style block
   so it runs once the DOM is ready. We use a Vue trick: the canvas
   is populated with a tiny inline script tag rendered via v-once. */

/* ── CHANGE 1: Hero background image ──────────────────────────────────────── */
.hero-bg-image {
  position: absolute;
  inset: 0;                
  background-size: fit;
  background-position: top;
  image-rendering: pixelated;
  background-repeat: no-repeat;
  z-index: 0;
  /* Subtle Ken-Burns drift — feels alive without distracting */
  animation: heroBgDrift 18s ease-in-out infinite alternate;
}

@keyframes heroBgDrift {
  from { transform: scale(1.04) translateY(0); }
  to   { transform: scale(1.08) translateY(-1.5%); }
}

/* Gradient overlay so text stays readable over any photo */
.hero-bg-overlay {
  position: absolute;
  inset: 0;
  background: linear-gradient(
    to bottom,
    rgba(0,0,0,0.35) 0%,
    rgba(0,0,0,0.10) 40%,
    rgba(0,0,0,0.60) 80%,
    rgba(0,0,0,0.95) 100%
  );
  z-index: 1;
}

/* ── HERO ─────────────────────────────────────────────────────────────────── */
#hero {
  position: relative;
  width: 100vw;
  /* CHANGE 4: taller hero gives the G animation more room */
  height: 260vh;
  overflow: hidden;
  padding-bottom: 40vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}
.hero-content {
  position: sticky;
  top: 20vh;
  bottom: 215vh;
  z-index: 4;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2.5rem;
  will-change: opacity;
}
#hero-title {
  font-family: 'Urbanist', sans-serif;
  font-size: clamp(3rem, 10vw, 9rem);
  font-weight: 800;
  letter-spacing: 0.1em;
  opacity: 0;
  transform: scale(0.6);
  color: #fff;
}
/* ── Outer glass pill container ───────────────────────────────────────────── */
.hero-btns {
  display: flex;
  align-items: center;
  gap: 0;
  opacity: 0;
  position: relative;
  overflow: hidden;
  border-radius: 100px;
  padding: 5px;
  background: rgba(8,8,8,0.48);
  backdrop-filter: blur(40px) saturate(200%) brightness(1.05);
  -webkit-backdrop-filter: blur(40px) saturate(200%) brightness(1.05);
  border: 1px solid rgba(255,255,255,0.13);
  box-shadow:
    inset 0 1px 0 rgba(255,255,255,0.1),
    inset 0 -1px 0 rgba(0,0,0,0.2),
    0 10px 40px rgba(0,0,0,0.45);
}
/* top shimmer on outer pill */
.hero-btns::before {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  background: linear-gradient(
    135deg,
    rgba(255,255,255,0.11) 0%,
    rgba(255,255,255,0.02) 50%,
    rgba(255,255,255,0.07) 100%
  );
  pointer-events: none;
  z-index: 0;
}

/* ── Get Started — green inner pill ──────────────────────────────────────── */
.btn-start {
  position: relative;
  z-index: 1;
  padding: 0.78rem 2.1rem;
  border-radius: 100px;
  font-family: 'Space Grotesk', sans-serif;
  font-size: 1rem;
  font-weight: 600;
  letter-spacing: 0.05em;
  cursor: pointer;
  color: #fff;
  background: rgba(0,22,13,0.62);
  border: 1.5px solid rgba(0,255,136,0.86);
  box-shadow:
    inset 0 1px 0 rgba(0,255,136,0.22),
    inset 0 -1px 0 rgba(0,0,0,0.3),
    0 0 18px rgba(0,255,136,0.2);
  transition: background 0.25s, border-color 0.25s, box-shadow 0.25s, transform 0.2s;
}
.btn-start:hover {
  transform: scale(1.03);
  background: rgba(0,32,18,0.68);
  border-color: #00ff88;
  box-shadow:
    inset 0 1px 0 rgba(0,255,136,0.3),
    inset 0 -1px 0 rgba(0,0,0,0.25),
    0 0 32px rgba(0,255,136,0.38);
}

/* ── Login — plain text inside the pill ──────────────────────────────────── */
.btn-login {
  position: relative;
  z-index: 1;
  padding: 0.78rem 2.1rem;
  border-radius: 100px;
  font-family: 'Space Grotesk', sans-serif;
  font-size: 1rem;
  font-weight: 500;
  letter-spacing: 0.05em;
  cursor: pointer;
  color: rgba(255,255,255,0.78);
  background: transparent;
  border: none;
  transition: color 0.2s, background 0.2s;
}
.btn-login:hover {
  color: #fff;
  background: rgba(255,255,255,0.07);
}

/* ── G SECTION: CHANGE 4 — taller, more breathing room ───────────────────── */
#g-section {
  position: sticky;
  top: 0;
  width: 100vw;
  /* Taller so the entire G animation arc has room — scrub happens over
     the full hero height now (200vh), giving the G plenty of travel   */
  height: 100vh;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  z-index: 20;
  /* Offset matches the new hero height */
  margin-top: -260vh;
  pointer-events: none;
}
#big-g {
  font-family: 'Urbanist', sans-serif;
  font-size: clamp(8rem, 25vw, 22rem);
  font-weight: 800;
  color: #00ff88;
  opacity: 0;
  position: absolute;
  z-index: 2;
  transform-origin: center center;
  will-change: transform, opacity;
}
#circle-text-svg {
  position: absolute;
  width: min(72vw, 640px);
  height: min(72vw, 640px);
  opacity: 0;
  z-index: 3;
  transform-origin: 50% 50%;
  will-change: transform, opacity;
}

#numbers-section {
  width: 100vw;
  min-height: 120vh;  /* taller so scroll animation has room */
  background: #000;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 4rem 2rem;
  overflow: hidden;
  position: relative;
  z-index: 2;
}

.numbers-screen {
  position: relative;
  width: min(860px, 90vw);
  min-height: 480px;
  border-radius: 16px;
  border: 1px solid rgba(0, 255, 136, 0.18);
  background: rgba(0, 0, 0, 0.82);
  box-shadow:
    0 0 0 1px rgba(0, 255, 136, 0.06),
    0 0 40px rgba(0, 255, 136, 0.06),
    inset 0 0 60px rgba(0, 0, 0, 0.6);
  padding: 3.5rem 2.5rem 2.5rem;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 1.2rem;
  overflow: hidden;  /* clips items outside the box */
}

.number-layer {
  display: flex;
  gap: 2.5rem;
  white-space: nowrap;
}

.number-item {
  font-family: 'Urbanist', sans-serif;
  font-weight: 800;
  color: rgba(0, 255, 136, 0.6);
  line-height: 1;
  user-select: none;
  opacity: 0;
  will-change: transform, opacity;
}

.nl-top .number-item { font-size: clamp(2rem, 5vw, 4.5rem); }
.nl-mid .number-item { font-size: clamp(3.5rem, 9vw, 8rem); -webkit-text-stroke: 2px rgba(0,255,136,0.35); color: transparent !important; }
.nl-bot .number-item { font-size: clamp(1.8rem, 4vw, 3.5rem); }

/* Screen top chrome — three dots like a terminal window */
.screen-chrome {
  position: absolute;
  top: 1.1rem;
  left: 1.4rem;
  display: flex;
  gap: 0.45rem;
  align-items: center;
}
.chrome-dot {
  width: 10px; height: 10px; border-radius: 50%;
  background: rgba(0, 255, 136, 0.25);
  border: 1px solid rgba(0, 255, 136, 0.4);
}
.chrome-dot:nth-child(1) { background: rgba(255, 80, 80, 0.55); border-color: rgba(255,80,80,0.7); }
.chrome-dot:nth-child(2) { background: rgba(255, 200, 0, 0.55); border-color: rgba(255,200,0,0.7); }
.chrome-dot:nth-child(3) { background: rgba(0, 255, 136, 0.55); border-color: rgba(0,255,136,0.7); }

/* CRT scanlines overlay */
.screen-scanlines {
  position: absolute;
  inset: 0;
  border-radius: 20px;
  background: repeating-linear-gradient(
    0deg,
    transparent,
    transparent 3px,
    rgba(0, 0, 0, 0.08) 3px,
    rgba(0, 0, 0, 0.08) 4px
  );
  pointer-events: none;
  z-index: 10;
}
/* Inner vignette */
.screen-vignette {
  position: absolute;
  inset: 0;
  border-radius: 20px;
  background: radial-gradient(ellipse at center, transparent 50%, rgba(0,0,0,0.55) 100%);
  pointer-events: none;
  z-index: 11;
}

.number-layer { display: flex; gap: 4rem; margin: 1.8rem 0; white-space: nowrap; }
.number-item {
  font-family: 'Urbanist', sans-serif; font-weight: 800;
  color: rgba(0,255,136,0.4); -webkit-text-stroke: 1px rgba(255,255,255,0.15);
  line-height: 1; user-select: none; opacity: 0;
}
.nl-top .number-item { font-size: clamp(3rem, 8vw, 7rem); }
.nl-mid .number-item { font-size: clamp(5rem, 14vw, 13rem); -webkit-text-stroke: 2px rgba(0,255,136,0.4); color: transparent !important; }
.nl-bot .number-item { font-size: clamp(2.5rem, 6vw, 5rem); }

/* ── SPIRAL ──────────────────────────────────────────────────────────────── */
#spiral-section { width: 100vw; min-height: 100vh; background: #000; position: relative; padding: 4rem 0; z-index: 10; }
#spiral-svg { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; z-index: 5; overflow: visible; }
.spiral-content { position: relative; z-index: 2; max-width: 600px; margin: 0 auto; padding-top: 6rem; display: flex; flex-direction: column; gap: 4rem;}
.spiral-para { opacity: 0; }
.spiral-para p { color: #fff; font-size: 1.2rem; line-height: 1.8; font-weight: 300;margin: 0 auto;text-align:center;width: 100%; }

/* ── BIG TEXT ────────────────────────────────────────────────────────────── */
#bigtext-section {
  width: 100vw; min-height: 80vh; background: #000;
  display: flex; flex-direction: column; justify-content: center;
  padding: 6rem 10vw; position: relative; z-index: 2;
  margin-top: 5vh;
}
#bigtext-section h2 {
  font-family: 'Urbanist', sans-serif; font-size: clamp(2rem, 6vw, 4rem);
  font-weight: 800; line-height: 1.15; color: #fff; opacity: 0; max-width: 900px;
  text-align: center;margin: 0 auto;
}
#bigtext-section .desc {
  margin-top: 2rem; font-size: 1.15rem; color: #6b7280; background: transparent;opacity: 0; max-width: 700px;margin: 0 auto;text-align: center;
}

/* ── RUBIK ───────────────────────────────────────────────────────────────── */
#rubik-section {
  width: 100vw;
  min-height: 200vh;  /* was 180vh — needs more room for 3 text stops */
  background: #000;
  position: relative;
  overflow: clip;
  z-index: 2;
}
.rubik-wrap {
  position: sticky;
  top: 0;           /* base — JS offsets from here */
  height: 420px;
  z-index: 5;
  pointer-events: none;
}

#rubik-canvas {
  position: absolute;
  left: 0;
  top: 0;           /* JS will control both left and top */
  width: 420px;
  height: 420px;
}
.rubik-text-left {
  position: absolute; top: 10vh; left: 8vw; max-width: 280px; opacity: 0; z-index: 6;
}
.rubik-text-right {
  position: absolute; bottom: 82vh; right: 8vw; max-width: 280px; opacity: 0; z-index: 6; text-align: right;
}
.rubik-text-left h3, .rubik-text-right h3 {
  font-family: 'Urbanist', sans-serif;
  font-size: clamp(1.2rem, 2vw, 1.6rem); font-weight: 700; line-height: 1.35;
  text-align:left;
  width: 120%;
}

.rubik-text-left p, .rubik-text-right p {
  margin-top: 0.75rem; color: #6b7280; font-size: 0.9rem; line-height: 1.6;
}
 .rubik-text-right h3{
text-align: center;
 } .rubik-text-right p {
  text-align: right;
 }

/* ── EXPERIENCE ──────────────────────────────────────────────────────────── */
#experience-section {
  width: 100vw; min-height: 60vh; background: #000;
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  text-align: center; padding: 0rem 10vw; position: relative; z-index: 2;
}
.exp-label { font-size: 0.8rem; letter-spacing: 0.2em; text-transform: uppercase; color: #00ff88; opacity: 0; }
.exp-header { font-family: 'Urbanist', sans-serif; font-size: clamp(2.5rem, 6vw, 5rem); font-weight: 800; margin: 1rem 0 1.5rem; opacity: 0; }
.exp-cta {
  display: inline-flex; align-items: center; gap: 0.75rem;
  padding: 0.9rem 2rem; border: 1px solid rgba(0,255,136,0.4); border-radius: 50px;
  color: #00ff88; font-size: 0.95rem; font-weight: 500; cursor: pointer;
  letter-spacing: 0.05em; opacity: 0; transition: background 0.2s; background: transparent;
}
.exp-cta:hover { background: rgba(0,255,136,0.1); }
.exp-stats { display: flex; gap: 3rem; margin-top: 4rem; flex-wrap: wrap; justify-content: center; }
.exp-stat { opacity: 0; text-align: center; }
.exp-stat .big { font-family: 'Urbanist', sans-serif; font-size: clamp(2rem, 4vw, 3.5rem); font-weight: 800; color: #fff; }
.exp-stat .label { font-size: 0.85rem; color: #6b7280; margin-top: 0.25rem; letter-spacing: 0.05em; }

/* ── SCENE WRAPPER ───────────────────────────────────────────────────────── */
.scene-wrapper {
  width: 100vw;
  height: 130vh;
  position: relative;
  z-index: 2;
  background: #000;
  overflow: hidden;
}

/* ── FAQ ─────────────────────────────────────────────────────────────────── */
#faq-section { width: 100vw; min-height: 80vh; background: #000; padding: 6rem 10vw; position: relative; z-index: 2; margin-bottom: 20vh;}
.faq-label { font-size: 0.8rem; letter-spacing: 0.2em; text-transform: uppercase; color: #00ff88; opacity: 0; }
.faq-header { font-family: 'Urbanist', sans-serif; font-size: clamp(2rem, 5vw, 4rem); font-weight: 800; margin: 1rem 0 3rem; opacity: 0; max-width: 700px; }
.faq-item { border-top: 1px solid #1a1a1a; padding: 1.5rem 0; opacity: 0; }
.faq-item:last-child { border-bottom: 1px solid #1a1a1a; }
.faq-q { display: flex; align-items: center; justify-content: space-between; cursor: pointer; font-size: 1.05rem; font-weight: 500; gap: 1rem; }
.faq-q span { color: #4b5563; font-size: 1.3rem; line-height: 1; transition: transform 0.3s; }
.faq-a { max-height: 0; overflow: hidden; transition: max-height 0.4s ease, padding 0.3s; color: #6b7280; font-size: 0.95rem; line-height: 1.7; }
.faq-item.open .faq-q span { transform: rotate(45deg); }
.faq-item.open .faq-a { max-height: 200px; padding-top: 1rem; }

/* ── GLE ─────────────────────────────────────────────────────────────────── */
#gle-section {
  width: 100vw; min-height: 60vh; background: #000;
  display: flex; align-items: center; justify-content: center;
  overflow: hidden; position: relative; z-index: 15;
}
.gle-text-wrap { display: flex; gap: clamp(1rem, 4vw, 4rem); position: relative; z-index: 2; }
.gle-word {
  font-family: 'Urbanist', sans-serif;
  font-size: clamp(4rem, 12vw, 11rem); font-weight: 800;
  color: transparent; -webkit-text-stroke: 1.5px rgba(255,255,255,0.18);
  opacity: 0; letter-spacing: 0.05em;
}
#w-overlay {
  position: absolute;
  font-family: 'Urbanist', sans-serif;
  font-size: clamp(6rem, 20vw, 18rem); font-weight: 800;
  color: #7c3aed; opacity: 0; transform: scale(0.3); z-index: 10;
  pointer-events: none; mix-blend-mode: screen;
}

/* ── FOOTER ──────────────────────────────────────────────────────────────── */
#footer {
  background: #7c3aed;
  width: 100vw;
  min-height: 70vh;
  padding: 5rem 6vw 4rem;
  display: grid;
  grid-template-columns: 1.6fr 0.7fr 0.7fr;
  gap: 4rem;
  position: relative;
  z-index: 30;
  clip-path: circle(0% at 50% 0%);
  opacity: 0;
  margin-top: -30vh;
}

.footer-col1 h3 {
  font-family: 'Urbanist', sans-serif;
  font-size: clamp(1.8rem, 3.5vw, 3rem);
  font-weight: 700; color: #fff; margin-bottom: 2rem;
}
.footer-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  align-items: stretch;
}
.footer-fields {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.85rem;
}
.footer-input-row {
  display: flex; align-items: center; gap: 0.75rem;
  padding: 0.9rem 1.25rem;
  border: 1px solid rgba(255,255,255,0.2); border-radius: 12px;
  background: rgba(255,255,255,0.08);
}
.footer-input-row svg { opacity: 0.6; flex-shrink: 0; }
.footer-input-row input {
  background: transparent; border: none; color: #fff;
  font-family: 'Space Grotesk', sans-serif; font-size: 0.95rem;
  outline: none; width: 100%;
}
.footer-input-row input::placeholder { color: rgba(255,255,255,0.45); }

.footer-submit {
  padding: 1rem 2rem;
  background: #fff; color: #7c3aed;
  border: none; border-radius: 14px;
  font-family: 'Urbanist', sans-serif;
  font-size: 1rem; font-weight: 700;
  cursor: pointer; white-space: nowrap;
  align-self: stretch;
  min-height: 100%;
  transition: opacity 0.2s, transform 0.2s;
  writing-mode: horizontal-tb;
}
.footer-submit:hover { opacity: 0.9; transform: translateY(-2px); }

.footer-copy { margin-top: 2rem; font-size: 0.8rem; color: rgba(255,255,255,0.45); }

.footer-col2 { padding-top: 0.5rem; }
.footer-col2 nav { display: flex; flex-direction: column; gap: 1.1rem; }
.footer-col2 nav a {
  color: rgba(255,255,255,0.7); text-decoration: none;
  font-size: 0.95rem; letter-spacing: 0.03em; transition: color 0.2s;
}
.footer-col2 nav a:hover { color: #fff; }

.footer-col3 { padding-top: 0.5rem; }
.footer-col3 .social-header {
  font-family: 'Urbanist', sans-serif; font-size: 1.2rem; font-weight: 700;
  color: #fff; margin-bottom: 1.25rem;
}
.social-icons { display: flex; gap: 1rem; }
.social-icon {
  width: 44px; height: 44px; border: 1px solid rgba(255,255,255,0.25);
  border-radius: 50%; display: flex; align-items: center; justify-content: center;
  cursor: pointer; transition: background 0.2s;
}
.social-icon:hover { background: rgba(255,255,255,0.15); }
.social-icon svg { width: 20px; height: 20px; fill: #fff; }

.footer-bottom-links {
  position: absolute; bottom: 3rem; right: 6vw;
  display: flex; gap: 2rem;
}
.footer-bottom-links a {
  color: rgba(255,255,255,0.45); text-decoration: none;
  font-size: 0.8rem; transition: color 0.2s;
}
.footer-bottom-links a:hover { color: #fff; }

@media (max-width: 768px) {
  #navbar { width: calc(100% - 2rem); }
  .nav-brand { display: none; }
  .nav-center { gap: 0; }
  .nav-link { font-size: 0.78rem; padding: 0.35rem 0.6rem; }
  #footer { grid-template-columns: 1fr; gap: 3rem; margin-top: 0; }
  .footer-form { flex-direction: column; }
  .footer-submit { min-height: 52px; align-self: flex-start; width: 100%; }
  .footer-bottom-links { position: relative; bottom: auto; right: auto; margin-top: 2rem; }
  .exp-stats { gap: 2rem; }
  .rubik-text-left, .rubik-text-right { display: none; }
  .numbers-screen { width: 98vw; padding: 3rem 1.2rem 2rem; }
}
.scene-gle-bg {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: clamp(1rem, 4vw, 4rem);
  z-index: 0;
  pointer-events: none;
}
.scene-gle-bg span {
  font-family: 'Urbanist', sans-serif;
  font-size: clamp(4rem, 12vw, 11rem);
  font-weight: 800;
  color: transparent;
  -webkit-text-stroke: 1.5px rgba(0, 255, 136, 0.15);
  letter-spacing: 0.05em;
  animation: gleFloat 2s ease-in-out infinite alternate;
}
.scene-gle-bg span:nth-child(2) { animation-delay: 0.2s; }
.scene-gle-bg span:nth-child(3) { animation-delay: 0.4s; }

@keyframes gleFloat {
  from { transform: translateY(0); }
  to   { transform: translateY(-28px); }
}
</style>