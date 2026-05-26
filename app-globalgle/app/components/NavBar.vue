<script setup>
import { ref, onMounted } from 'vue'

const isDark       = ref(true)
const logoSrc      = ref('/logo1.jpg')
const isAnimating  = ref(false)
const menuOpen = ref(false)

const toggleTheme = () => {
  if (isAnimating.value) return

  isAnimating.value = true
  isDark.value = !isDark.value
  const theme = isDark.value ? 'dark' : 'light'
  document.documentElement.setAttribute('data-theme', theme)
  localStorage.setItem('globalgle-theme', theme)

  // Swap the logo image at exactly the moment it's edge-on (invisible)
  setTimeout(() => {
    logoSrc.value = isDark.value ? '/logo1.jpg' : '/logo2.jpg'
  }, 155)

  // Unlock after the full animation completes
  setTimeout(() => { isAnimating.value = false }, 750)
}

onMounted(() => {
  const saved = localStorage.getItem('globalgle-theme')
  const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
  const theme = saved ?? (prefersDark ? 'dark' : 'light')
  isDark.value = theme === 'dark'
  logoSrc.value = isDark.value ? '/logo1.jpg' : '/logo2.jpg'
  document.documentElement.setAttribute('data-theme', theme)
})
</script>

<template>
  <nav id="navbar">

    <!-- LEFT: logo -->
    <div class="nav-left">
      <div class="logo-wrap" :class="{ 'is-flipping': isAnimating }">
        <span class="ring ring-1"></span>
        <span class="ring ring-2"></span>
        <span class="ring ring-3"></span>
        <img :src="logoSrc" alt="GlobalGLE logo" class="nav-logo" />
      </div>
    </div>

    <!-- CENTER: links -->
    <div class="nav-center">
      <a href="#" class="nav-link">Work</a>
      <a href="#" class="nav-link">About</a>
      <span class="nav-brand">GlobalGLE</span>
      <a href="#" class="nav-link">Services</a>
      <a href="#" class="nav-link">Portfolio</a>
    </div>

    <!-- RIGHT: theme toggle + contact -->
    <div class="nav-right">
      <button
        class="theme-btn"
        @click="toggleTheme"
        :aria-label="isDark ? 'Switch to light mode' : 'Switch to dark mode'"
        :title="isDark ? 'Light mode' : 'Dark mode'"
      >
        <Transition name="icon-swap" mode="out-in">
          <svg v-if="isDark" key="moon" class="theme-icon" viewBox="0 0 24 24"
               fill="none" stroke="currentColor" stroke-width="2"
               stroke-linecap="round" stroke-linejoin="round">
            <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/>
          </svg>
          <svg v-else key="sun" class="theme-icon" viewBox="0 0 24 24"
               fill="none" stroke="currentColor" stroke-width="2"
               stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="5"/>
            <line x1="12" y1="1"  x2="12" y2="3"/>
            <line x1="12" y1="21" x2="12" y2="23"/>
            <line x1="4.22"  y1="4.22"  x2="5.64"  y2="5.64"/>
            <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"/>
            <line x1="1"  y1="12" x2="3"  y2="12"/>
            <line x1="21" y1="12" x2="23" y2="12"/>
            <line x1="4.22"  y1="19.78" x2="5.64"  y2="18.36"/>
            <line x1="18.36" y1="5.64"  x2="19.78" y2="4.22"/>
          </svg>
        </Transition>
      </button>

      <button class="nav-contact-btn">Contact</button>
      <button class="hamburger" @click="menuOpen = !menuOpen" :class="{ open: menuOpen }" aria-label="Menu">
    <span></span><span></span><span></span>
  </button>
</div>

<!-- Mobile drawer -->
<Transition name="drawer">
  <div v-if="menuOpen" class="mobile-drawer" @click="menuOpen = false">
    <a href="#" class="drawer-link">Work</a>
    <a href="#" class="drawer-link">About</a>
    <a href="#" class="drawer-link">Services</a>
    <a href="#" class="drawer-link">Portfolio</a>
  </div>
</Transition>

  </nav>
</template>

<style scoped>
/* ── Navbar shell ─────────────────────────────────────────────────────────── */
#navbar {
  position: fixed;
  top: 1.25rem;
  left: 50%;
  transform: translateX(-50%);
  width: calc(100% - 4rem);
  max-width: 1200px;
  height: 56px;
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  align-items: center;
  padding: 0 1rem 0 0.6rem;
  border-radius: 100px;
  border: 1px solid rgba(255,255,255,0.12);
  background: rgba(0,0,0,0.35);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  z-index: 100;
  transition: background 0.4s, border-color 0.4s, box-shadow 0.4s;
}
#navbar.scrolled {
  background: rgba(0,0,0,0.65);
  border-color: rgba(255,255,255,0.18);
  box-shadow: 0 8px 40px rgba(0,0,0,0.4);
}

/* ── Columns ──────────────────────────────────────────────────────────────── */
.nav-left  { display: flex; align-items: center; justify-content: flex-start; }
.nav-center { display: flex; align-items: center; gap: 0.25rem; }
.nav-right  { display: flex; align-items: center; justify-content: flex-end; gap: 0.75rem; }

/* ── Logo ─────────────────────────────────────────────────────────────────── */
.logo-wrap {
  position: relative;
  width: 42px;
  height: 42px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.nav-logo {
  width: 42px;
  height: 42px;
  border-radius: 50%;
  object-fit: cover;
  border: 1.5px solid rgba(255,255,255,0.18);
  display: block;
  position: relative;
  z-index: 2;
  /* smooth border colour transition when theme changes */
  transition: border-color 0.4s;
}

/* ── Burst rings (hidden at rest) ─────────────────────────────────────────── */
.ring {
  position: absolute;
  inset: 0;
  border-radius: 50%;
  border: 2px solid #00ff88;
  opacity: 0;
  pointer-events: none;
  z-index: 1;
}

/* ── Logo flip animation ──────────────────────────────────────────────────── */
@keyframes logoFlip {
  0%   { transform: perspective(420px) rotateY(0deg)   rotateZ(0deg)   scale(1);    filter: brightness(1); }
  20%  { transform: perspective(420px) rotateY(72deg)  rotateZ(6deg)   scale(0.82); filter: brightness(2.8) saturate(2.5); }
  /* edge-on — logo swapped by JS here */
  37%  { transform: perspective(420px) rotateY(134deg) rotateZ(-4deg)  scale(0.72); filter: brightness(0.4) blur(1px); }
  55%  { transform: perspective(420px) rotateY(210deg) rotateZ(8deg)   scale(1.18); filter: brightness(3.5) saturate(3); }
  72%  { transform: perspective(420px) rotateY(288deg) rotateZ(-6deg)  scale(0.88); filter: brightness(1.8); }
  88%  { transform: perspective(420px) rotateY(345deg) rotateZ(3deg)   scale(1.06); filter: brightness(1.2); }
  100% { transform: perspective(420px) rotateY(360deg) rotateZ(0deg)   scale(1);    filter: brightness(1); }
}

@keyframes ring1 {
  0%   { transform: scale(1);   opacity: 0.95; }
  100% { transform: scale(5.5); opacity: 0; }
}
@keyframes ring2 {
  0%   { transform: scale(1);   opacity: 0.75; }
  100% { transform: scale(4.2); opacity: 0; }
}
@keyframes ring3 {
  0%   { transform: scale(1);   opacity: 0.5; }
  100% { transform: scale(3.2); opacity: 0; }
}

/* Trigger all animations via a single class on the wrapper */
.logo-wrap.is-flipping .nav-logo {
  animation: logoFlip 0.72s cubic-bezier(0.34, 1.3, 0.64, 1) forwards;
}
.logo-wrap.is-flipping .ring-1 {
  animation: ring1 0.65s ease-out forwards;
}
.logo-wrap.is-flipping .ring-2 {
  animation: ring2 0.65s 0.09s ease-out forwards;
}
.logo-wrap.is-flipping .ring-3 {
  animation: ring3 0.65s 0.18s ease-out forwards;
}

/* ── Nav links ────────────────────────────────────────────────────────────── */
.nav-link {
  color: rgba(255,255,255,0.72);
  text-decoration: none;
  font-size: 0.9rem;
  font-weight: 500;
  letter-spacing: 0.03em;
  padding: 0.4rem 0.9rem;
  border-radius: 50px;
  transition: color 0.2s, background 0.2s;
}
.nav-link:hover { color: #fff; background: rgba(255,255,255,0.08); }

.nav-brand {
  font-family: 'Urbanist', sans-serif;
  font-size: 0.95rem;
  font-weight: 700;
  color: rgba(255,255,255,0.5);
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

/* ── Theme toggle button ──────────────────────────────────────────────────── */
.theme-btn {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: 1px solid rgba(255,255,255,0.2);
  background: rgba(255,255,255,0.07);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: border-color 0.3s, background 0.3s, box-shadow 0.3s;
}
.theme-btn:hover {
  border-color: rgba(0,255,136,0.55);
  background: rgba(0,255,136,0.1);
  box-shadow: 0 0 16px rgba(0,255,136,0.22);
}
.theme-btn:active { transform: scale(0.92); }

.theme-icon {
  width: 16px;
  height: 16px;
  color: rgba(255,255,255,0.88);
  display: block;
  transition: color 0.3s;
}

/* Icon swap transition */
.icon-swap-enter-active,
.icon-swap-leave-active { transition: opacity 0.16s ease, transform 0.16s ease; }
.icon-swap-enter-from   { opacity: 0; transform: rotate(80deg)  scale(0.4); }
.icon-swap-leave-to     { opacity: 0; transform: rotate(-80deg) scale(0.4); }

/* ── Contact button ───────────────────────────────────────────────────────── */
.nav-contact-btn {
  padding: 0.5rem 1.4rem;
  border-radius: 50px;
  border: 1.5px solid rgba(0,255,136,0.82);
  background: rgba(0,22,13,0.72);
  color: #fff;
  font-family: 'Urbanist', sans-serif;
  font-size: 0.8rem;
  font-weight: 700;
  letter-spacing: 0.13em;
  text-transform: uppercase;
  cursor: pointer;
  position: relative;
  overflow: hidden;
  backdrop-filter: blur(36px) saturate(180%);
  -webkit-backdrop-filter: blur(36px) saturate(180%);
  box-shadow:
    inset 0 1px 0 rgba(0,255,136,0.18),
    inset 0 -1px 0 rgba(0,0,0,0.28),
    0 0 14px rgba(0,255,136,0.15);
  transition: background 0.25s, border-color 0.25s, box-shadow 0.25s, transform 0.2s;
}
/* top-edge shimmer */
.nav-contact-btn::before {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  background: linear-gradient(
    135deg,
    rgba(255,255,255,0.1) 0%,
    rgba(255,255,255,0.02) 55%,
    rgba(255,255,255,0.06) 100%
  );
  pointer-events: none;
}
.nav-contact-btn:hover {
  transform: scale(1.04);
  background: rgba(0,32,18,0.78);
  border-color: #00ff88;
  box-shadow:
    inset 0 1px 0 rgba(0,255,136,0.26),
    inset 0 -1px 0 rgba(0,0,0,0.22),
    0 0 26px rgba(0,255,136,0.32);
}

/* ── Responsive ───────────────────────────────────────────────────────────── */
@media (max-width: 768px) {
  #navbar {
    width: calc(100% - 2rem);
    padding: 0 0.75rem 0 0.5rem;
    grid-template-columns: auto 1fr auto;  /* logo | spacer | right */
  }
  .nav-center { display: none; }  /* hide all center links on mobile */
  .nav-brand  { display: none; }
  .logo-wrap, .nav-logo { width: 36px; height: 36px; }
  .theme-btn  { width: 32px; height: 32px; }
  .theme-icon { width: 14px; height: 14px; }
  .nav-contact-btn {
    font-size: 0.72rem;
    padding: 0.45rem 1rem;
    letter-spacing: 0.08em;
  }
}
</style>

<!-- Non-scoped: light-mode overrides keyed to [data-theme] on <html> -->
<style>
[data-theme="light"] #navbar {
  background: rgba(255,255,255,0.78) !important;
  border-color: rgba(0,0,0,0.1) !important;
}
[data-theme="light"] #navbar.scrolled {
  background: rgba(255,255,255,0.95) !important;
  border-color: rgba(0,0,0,0.13) !important;
  box-shadow: 0 8px 40px rgba(0,0,0,0.1) !important;
}
[data-theme="light"] .nav-logo {
  border-color: rgba(0,0,0,0.15) !important;
}
[data-theme="light"] .ring {
  border-color: #00aa55 !important;
}
[data-theme="light"] .nav-link { color: rgba(0,0,0,0.65) !important; }
[data-theme="light"] .nav-link:hover { color: #000 !important; background: rgba(0,0,0,0.06) !important; }
[data-theme="light"] .nav-brand { color: rgba(0,0,0,0.45) !important; }
[data-theme="light"] .nav-contact-btn {
  border-color: rgba(0,140,75,0.82) !important;
  color: #fff !important;
  background: rgba(0,40,22,0.78) !important;
  box-shadow:
    inset 0 1px 0 rgba(0,200,100,0.15),
    0 0 14px rgba(0,140,75,0.18) !important;
}
[data-theme="light"] .nav-contact-btn:hover {
  background: rgba(0,50,28,0.85) !important;
  border-color: #00aa55 !important;
  box-shadow:
    inset 0 1px 0 rgba(0,200,100,0.22),
    0 0 26px rgba(0,140,75,0.3) !important;
}
[data-theme="light"] .theme-btn {
  border-color: rgba(0,0,0,0.15) !important;
  background: rgba(0,0,0,0.05) !important;
}
[data-theme="light"] .theme-btn:hover {
  border-color: rgba(0,140,75,0.5) !important;
  background: rgba(0,140,75,0.1) !important;
  box-shadow: 0 0 16px rgba(0,140,75,0.2) !important;
}
[data-theme="light"] .theme-icon { color: rgba(0,0,0,0.72) !important; }
.hamburger {
  display: none;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 5px;
  width: 36px;
  height: 36px;
  background: rgba(255,255,255,0.07);
  border: 1px solid rgba(255,255,255,0.2);
  border-radius: 50%;
  cursor: pointer;
  flex-shrink: 0;
}
.hamburger span {
  display: block;
  width: 16px;
  height: 1.5px;
  background: rgba(255,255,255,0.85);
  border-radius: 2px;
  transition: transform 0.3s, opacity 0.3s;
}
.hamburger.open span:nth-child(1) { transform: translateY(6.5px) rotate(45deg); }
.hamburger.open span:nth-child(2) { opacity: 0; }
.hamburger.open span:nth-child(3) { transform: translateY(-6.5px) rotate(-45deg); }

.mobile-drawer {
  position: fixed;
  top: 80px;
  left: 50%;
  transform: translateX(-50%);
  width: calc(100% - 2rem);
  background: rgba(0,0,0,0.88);
  backdrop-filter: blur(24px);
  -webkit-backdrop-filter: blur(24px);
  border: 1px solid rgba(255,255,255,0.12);
  border-radius: 20px;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  z-index: 99;
}
.drawer-link {
  color: rgba(255,255,255,0.8);
  text-decoration: none;
  font-size: 1rem;
  font-weight: 500;
  padding: 0.85rem 1.2rem;
  border-radius: 12px;
  transition: background 0.2s, color 0.2s;
}
.drawer-link:hover {
  background: rgba(255,255,255,0.08);
  color: #fff;
}

.drawer-enter-active, .drawer-leave-active { transition: opacity 0.2s, transform 0.2s; }
.drawer-enter-from, .drawer-leave-to { opacity: 0; transform: translateX(-50%) translateY(-8px); }

@media (max-width: 768px) {
  .hamburger { display: flex; }
}
[data-theme="light"] .hamburger {
  border-color: rgba(0,0,0,0.15) !important;
  background: rgba(0,0,0,0.05) !important;
}
[data-theme="light"] .hamburger span { background: rgba(0,0,0,0.7) !important; }
[data-theme="light"] .mobile-drawer {
  background: rgba(255,255,255,0.95) !important;
  border-color: rgba(0,0,0,0.1) !important;
}
[data-theme="light"] .drawer-link { color: rgba(0,0,0,0.7) !important; }
[data-theme="light"] .drawer-link:hover { background: rgba(0,0,0,0.06) !important; color: #000 !important; }
</style>
