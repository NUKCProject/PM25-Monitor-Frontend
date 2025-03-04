<template>
    <div ref="threeContainer" class="three-container"></div>
  </template>
  
  <script setup>
  import { ref, onMounted, onUnmounted } from 'vue';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
  import { RoundedBoxGeometry } from 'three/examples/jsm/geometries/RoundedBoxGeometry.js';
  
  const threeContainer = ref(null);
  const pmData = ref({ pm1_0: 0, pm2_5: 0, pm10: 0 });
  let fetchInterval = null;
  let lastUpdate = Date.now();
  let materials = [];
  
  onMounted(() => {
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    threeContainer.value.appendChild(renderer.domElement);
  
    scene.add(new THREE.AmbientLight(0xffffff, 0.6));
    const light = new THREE.DirectionalLight(0xffffff, 0.8);
    light.position.set(5, 5, 5);
    scene.add(light);
  
    materials = [
      new THREE.MeshStandardMaterial({ color: 0x888888, transparent: true, opacity: 0.9 }),
      new THREE.MeshStandardMaterial({ color: 0x888888, transparent: true, opacity: 0.9 }),
      new THREE.MeshStandardMaterial({ color: 0x222222 }),
      new THREE.MeshStandardMaterial({ color: 0x222222 }),
      new THREE.MeshStandardMaterial({ color: 0x888888, transparent: true, opacity: 0.9 }),
      new THREE.MeshStandardMaterial({ color: 0x888888, transparent: true, opacity: 0.9 })
    ];
  
    const geometry = new RoundedBoxGeometry(3, 3, 3, 12, 0.3);
    const cube = new THREE.Mesh(geometry, materials);
    scene.add(cube);
  
    camera.position.z = 6;
  
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
  
    let isUserInteracting = false;
  
    renderer.domElement.addEventListener('pointerdown', () => isUserInteracting = true);
    renderer.domElement.addEventListener('pointerup', () => isUserInteracting = false);
  
    const animate = () => {
      requestAnimationFrame(animate);
      if (!isUserInteracting) cube.rotation.y += 0.002;
      controls.update();
      renderer.render(scene, camera);
    };
    animate();
  
    fetchPMData();
    fetchInterval = setInterval(() => {
      fetchPMData();
      if (Date.now() - lastUpdate > 5000) showWaitingText();
    }, 1000);
  
    function fetchPMData() {
      fetch('http://192.168.0.21:8080/latest-pm')
        .then(res => res.json())
        .then(data => {
          pmData.value = data;
          lastUpdate = Date.now();
          updateCubeText();
        })
        .catch(() => showWaitingText());
    }
  
    function updateCubeText() {
      const texts = [
        `PM1.0: ${pmData.value.pm1_0} μg/m³`,
        `PM2.5: ${pmData.value.pm2_5} μg/m³`,
        `PM10: ${pmData.value.pm10} μg/m³`,
        new Date().toLocaleTimeString()
      ];
      const faces = [0, 1, 4, 5];
  
      for (let i = 0; i < faces.length; i++) {
        const texture = createTextTexture(texts[i]);
        texture.center.set(0.5, 0.5);
        texture.rotation = Math.PI; // ✅ 修正顛倒
        materials[faces[i]].map = texture;
        materials[faces[i]].needsUpdate = true;
      }
    }
  
    function showWaitingText() {
      const waitingTexture = createTextTexture('等待更新...', '#555');
      waitingTexture.center.set(0.5, 0.5);
      waitingTexture.rotation = Math.PI; // ✅ 修正顛倒
      [0, 1, 4, 5].forEach(i => {
        materials[i].map = waitingTexture;
        materials[i].needsUpdate = true;
      });
    }
  
    function createTextTexture(text, bgColor = '#000') {
      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d');
      canvas.width = 512;
      canvas.height = 512;
  
      ctx.fillStyle = bgColor;
      ctx.fillRect(0, 0, canvas.width, canvas.height);
  
      const borderPadding = 30;
      ctx.strokeStyle = '#FFF';
      ctx.lineWidth = 12;
      ctx.strokeRect(
        borderPadding,
        borderPadding,
        canvas.width - borderPadding * 2,
        canvas.height - borderPadding * 2
      );
  
      const maxLength = 20;
      if (text.length > maxLength) {
        text = text.slice(0, maxLength) + '...';
      }
  
      ctx.fillStyle = '#FFF';
      ctx.font = 'bold 48px "Microsoft JhengHei"';
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      ctx.fillText(text, canvas.width / 2, canvas.height / 2);
  
      return new THREE.CanvasTexture(canvas);
    }
  });
  
  onUnmounted(() => {
    clearInterval(fetchInterval);
  });
  </script>
  
  <style scoped>
  .three-container {
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    margin: 0;
    background-color: #111;
  }
  </style>
  