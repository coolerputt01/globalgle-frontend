<script setup>
import { ref, onMounted } from 'vue'

const isDark = ref(true)

const toggleTheme = () => {
  isDark.value = !isDark.value
  const theme = isDark.value ? 'dark' : 'light'
  document.documentElement.setAttribute('data-theme', theme)
  localStorage.setItem('globalgle-theme', theme)
}

onMounted(() => {
  const saved = localStorage.getItem('globalgle-theme')
  const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
  const theme = saved ?? (prefersDark ? 'dark' : 'light')
  isDark.value = theme === 'dark'
  document.documentElement.setAttribute('data-theme', theme)
})
</script>

<template>
  <nav id="navbar">
    <div class="nav-left">
      <button
        class="theme-btn"
        @click="toggleTheme"
        :aria-label="isDark ? 'Switch to light mode' : 'Switch to dark mode'"
        :title="isDark ? 'Light mode' : 'Dark mode'"
      >
        <Transition name="icon-swap" mode="out-in">
          <!-- Moon — shown in dark mode -->
          <svg
            v-if="isDark"
            key="moon"
            class="theme-icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/>
          </svg>
          <!-- Sun — shown in light mode -->
          <svg
            v-else
            key="sun"
            class="theme-icon"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <circle cx="12" cy="12" r="5"/>
            <line x1="12" y1="1"    x2="12" y2="3"/>
            <line x1="12" y1="21"   x2="12" y2="23"/>
            <line x1="4.22" y1="4.22"   x2="5.64" y2="5.64"/>
            <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"/>
            <line x1="1"  y1="12" x2="3"  y2="12"/>
            <line x1="21" y1="12" x2="23" y2="12"/>
            <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"/>
            <line x1="18.36" y1="5.64" x2="19.78" y2="4.22"/>
          </svg>
        </Transition>
      </button>
    </div>

    <div class="nav-center">
      <a href="#" class="nav-link">Work</a>
      <a href="#" class="nav-link">About</a>
      <span class="nav-brand">GlobalGLE</span>
      <a href="#" class="nav-link">Services</a>
      <a href="#" class="nav-link">Portfolio</a>
    </div>

    <div class="nav-right">
      <button class="nav-contact-btn">Contact</button>
    </div>
  </nav>
</template>

<style scoped>
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
  padding: 0 1.5rem;
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

.nav-left {
  display: flex;
  align-items: center;
  justify-content: flex-start;
}
.nav-center {
  display: flex;
  align-items: center;
  gap: 0.25rem;
}
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
.nav-link:hover {
  color: #fff;
  background: rgba(255,255,255,0.08);
}
.nav-right {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 1rem;
}
.nav-brand {
  font-family: 'Urbanist', sans-serif;
  font-size: 0.95rem;
  font-weight: 700;
  color: rgba(255,255,255,0.5);
  letter-spacing: 0.08em;
  text-transform: uppercase;
}
.nav-contact-btn {
  padding: 0.5rem 1.4rem;
  border-radius: 50px;
  border: 1px solid rgba(0,255,136,0.45);
  background: rgba(0,255,136,0.1);
  color: #00ff88;
  font-family: 'Urbanist', sans-serif;
  font-size: 0.875rem;
  font-weight: 600;
  letter-spacing: 0.04em;
  cursor: pointer;
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  transition: background 0.2s, border-color 0.2s, box-shadow 0.2s;
}
.nav-contact-btn:hover {
  background: rgba(0,255,136,0.2);
  border-color: rgba(0,255,136,0.7);
  box-shadow: 0 0 20px rgba(0,255,136,0.2);
}

/* ── Theme Toggle Button ──────────────────────────────────────────────────── */
.theme-btn {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  border: 1px solid rgba(255,255,255,0.2);
  background: rgba(255,255,255,0.07);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: border-color 0.3s, background 0.3s, box-shadow 0.3s;
  flex-shrink: 0;
}
.theme-btn:hover {
  border-color: rgba(0,255,136,0.55);
  background: rgba(0,255,136,0.1);
  box-shadow: 0 0 18px rgba(0,255,136,0.22);
}
.theme-btn:active {
  transform: scale(0.93);
}
.theme-icon {
  width: 17px;
  height: 17px;
  color: rgba(255,255,255,0.88);
  display: block;
  transition: color 0.3s;
}

/* Icon swap animation */
.icon-swap-enter-active,
.icon-swap-leave-active {
  transition: opacity 0.18s ease, transform 0.18s ease;
}
.icon-swap-enter-from {
  opacity: 0;
  transform: rotate(80deg) scale(0.4);
}
.icon-swap-leave-to {
  opacity: 0;
  transform: rotate(-80deg) scale(0.4);
}

@media (max-width: 768px) {
  #navbar { width: calc(100% - 2rem); padding: 0 1rem; }
  .nav-brand { display: none; }
  .nav-link { font-size: 0.78rem; padding: 0.35rem 0.6rem; }
  .theme-btn { width: 34px; height: 34px; }
  .theme-icon { width: 15px; height: 15px; }
}
</style>

<!-- Non-scoped: light mode overrides that rely on [data-theme] on <html> -->
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
[data-theme="light"] .nav-link {
  color: rgba(0,0,0,0.65) !important;
}
[data-theme="light"] .nav-link:hover {
  color: #000 !important;
  background: rgba(0,0,0,0.06) !important;
}
[data-theme="light"] .nav-brand {
  color: rgba(0,0,0,0.45) !important;
}
[data-theme="light"] .nav-contact-btn {
  border-color: rgba(0,140,75,0.5) !important;
  color: #00773d !important;
  background: rgba(0,140,75,0.1) !important;
}
[data-theme="light"] .nav-contact-btn:hover {
  background: rgba(0,140,75,0.18) !important;
  border-color: rgba(0,140,75,0.75) !important;
  box-shadow: 0 0 20px rgba(0,140,75,0.2) !important;
}
[data-theme="light"] .theme-btn {
  border-color: rgba(0,0,0,0.15) !important;
  background: rgba(0,0,0,0.05) !important;
}
[data-theme="light"] .theme-btn:hover {
  border-color: rgba(0,140,75,0.5) !important;
  background: rgba(0,140,75,0.1) !important;
  box-shadow: 0 0 18px rgba(0,140,75,0.2) !important;
}
[data-theme="light"] .theme-icon {
  color: rgba(0,0,0,0.72) !important;
}
</style>
