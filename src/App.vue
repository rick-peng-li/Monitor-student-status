<template>
  <div class="app-container">
    <a-layout class="layout">
      <a-layout-header class="header">
        <div class="logo">
          <span class="logo-icon">🎓</span>
          课堂 AI 监测系统
        </div>
      </a-layout-header>
      <a-layout-content class="content">
        <StudentMonitor />
      </a-layout-content>
    </a-layout>
  </div>
</template>

<script setup>
import StudentMonitor from './components/StudentMonitor.vue';
</script>

<style>
/* Global Reset */
html, body, #app {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
}

.app-container {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.layout {
  height: 100%;
  background: #f0f2f5;
}

.header {
  background: #001529;
  padding: 0 24px;
  display: flex;
  align-items: center;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  z-index: 10;
  height: 64px;
  flex-shrink: 0;
}

@media (max-width: 768px) {
  .header {
    height: 48px;
    padding: 0 12px;
  }
  
  .logo {
    font-size: 1rem;
  }
  
  .logo-icon {
    font-size: 1.2rem;
  }
}

.logo {
  color: white;
  font-size: 1.25rem;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 10px;
}

.logo-icon {
  font-size: 1.5rem;
}

.content {
  flex: 1;
  overflow: hidden; /* Prevent scrolling on the main container */
  position: relative;
}

/* Scrollbar styling */
::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}
::-webkit-scrollbar-track {
  background: transparent;
}
::-webkit-scrollbar-thumb {
  background: #ccc;
  border-radius: 3px;
}
::-webkit-scrollbar-thumb:hover {
  background: #999;
}
</style>
