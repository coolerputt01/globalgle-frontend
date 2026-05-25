<template>
  <div>
    <NuxtLayout>
      <NuxtPage />
    </NuxtLayout>
  </div>
</template>

<style>
/* ── Default (dark) ───────────────────────────────────────────────────────── */
:root {
  color-scheme: dark;
}

/* ── Light mode global overrides ─────────────────────────────────────────── */
[data-theme="light"] {
  color-scheme: light;
}

[data-theme="light"] body {
  background: #f4f6f8 !important;
  color: #0d0d0d !important;
}

/* Star field: nearly invisible on light background */
[data-theme="light"] .star-field {
  opacity: 0.1 !important;
  background-image:
    radial-gradient(circle, rgba(0,80,45,0.3) 1px, transparent 1px),
    radial-gradient(circle, rgba(0,80,45,0.18) 1px, transparent 1px),
    radial-gradient(circle, rgba(0,80,45,0.1) 1px, transparent 1px) !important;
}

/* Sections with hardcoded #000 backgrounds → transparent in light mode
   so the body background colour shows through */
[data-theme="light"] #numbers-section,
[data-theme="light"] #faq-section,
[data-theme="light"] #gle-section,
[data-theme="light"] #bigtext-section,
[data-theme="light"] #rubik-section,
[data-theme="light"] .scene-wrapper {
  background: #f4f6f8 !important;
}

/* Keep the numbers screen itself visually dark — it's a terminal display */
[data-theme="light"] .numbers-screen {
  background: rgba(10,10,10,0.92) !important;
}

/* Big text section */
[data-theme="light"] #bigtext-section h2 {
  color: #0d0d0d !important;
}
[data-theme="light"] #bigtext-section .desc {
  color: #4b5563 !important;
}

/* Spiral section text */
[data-theme="light"] .spiral-para p {
  color: #374151 !important;
}

/* FAQ */
[data-theme="light"] .faq-item {
  border-top-color: rgba(0,0,0,0.1) !important;
}
[data-theme="light"] .faq-item:last-child {
  border-bottom-color: rgba(0,0,0,0.1) !important;
}
[data-theme="light"] .faq-header {
  color: #0d0d0d !important;
}
[data-theme="light"] .faq-q {
  color: #0d0d0d !important;
}
[data-theme="light"] .faq-q span {
  color: #374151 !important;
}
[data-theme="light"] .faq-a {
  color: #4b5563 !important;
}

/* GLE outline text: adapt stroke to dark colour */
[data-theme="light"] .gle-word {
  -webkit-text-stroke-color: rgba(0,0,0,0.15) !important;
}

/* Rubik section text blocks */
[data-theme="light"] .rubik-text-left h3,
[data-theme="light"] .rubik-text-right h3,
[data-theme="light"] .rubik-text-bottom h3 {
  color: #0d0d0d !important;
}
[data-theme="light"] .rubik-text-left p,
[data-theme="light"] .rubik-text-right p,
[data-theme="light"] .rubik-text-bottom p {
  color: #4b5563 !important;
}

/* Experience section */
[data-theme="light"] .exp-header {
  color: #0d0d0d !important;
}
[data-theme="light"] .exp-stat .big {
  color: #0d0d0d !important;
}
[data-theme="light"] .exp-stat .label {
  color: #6b7280 !important;
}

/* Footer stays purple — intentional brand colour, no override */
</style>
