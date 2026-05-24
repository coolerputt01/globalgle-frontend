<template>
    <nav id="navbar">
      <div class="nav-left">
        <span class="nav-logo-mark"></span>
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
  .nav-logo-mark {
    font-family: 'Urbanist', sans-serif;
    font-size: 1.1rem;
    font-weight: 800;
    color: rgba(255,255,255,0.5);
    letter-spacing: 0.05em;
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
  
  @media (max-width: 768px) {
    #navbar { width: calc(100% - 2rem); padding: 0 1rem; }
    .nav-brand { display: none; }
    .nav-link { font-size: 0.78rem; padding: 0.35rem 0.6rem; }
  }
  </style>