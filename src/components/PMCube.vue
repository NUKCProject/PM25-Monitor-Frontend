<template>
    <div ref="threeContainer" class="three-container"></div>
  </template>
  
  <script setup>
  import { ref, onMounted, onUnmounted } from 'vue';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
  
  const threeContainer = ref(null);
  const pmData = ref({ pm1_0: 0, pm2_5: 0, pm10: 0 });
  let fetchInterval = null;
  let materials = [];
  
  onMounted(() => {
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    threeContainer.value.appendChild(renderer.domElement);
  
    // 依照 BoxGeometry 順序建立材質
    materials = [
      new THREE.MeshBasicMaterial({ color: 0x222222 }), // 右面 (+X)
      new THREE.MeshBasicMaterial({ color: 0x222222 }), // 左面 (-X)
      new THREE.MeshBasicMaterial({ color: 0x000000 }), // 上面 (+Y) 空白
      new THREE.MeshBasicMaterial({ color: 0x000000 }), // 下面 (-Y) 空白
      new THREE.MeshBasicMaterial({ color: 0x222222 }), // 前面 (+Z)
      new THREE.MeshBasicMaterial({ color: 0x222222 })  // 後面 (-Z)
    ];
  
    const geometry = new THREE.BoxGeometry(2, 2, 2);
    const cube = new THREE.Mesh(geometry, materials);
    scene.add(cube);
  
    camera.position.z = 5;
  
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
  
    const animate = () => {
      requestAnimationFrame(animate);
      cube.rotation.y += 0.005;
      controls.update();
      renderer.render(scene, camera);
    };
    animate();
  
    fetchPMData();
    fetchInterval = setInterval(fetchPMData, 1000);
  
    function fetchPMData() {
      fetch('http://192.168.0.21:8080/latest-pm')
        .then(res => res.json())
        .then(data => {
          pmData.value = data;
          console.log('最新 PM 數據:', pmData.value);
          updateCubeText();
        })
        .catch(err => console.error('獲取數據失敗:', err));
    }
  
    function updateCubeText() {
      const texts = [
        `PM1.0: ${pmData.value.pm1_0} μg/m³`,
        `PM2.5: ${pmData.value.pm2_5} μg/m³`,
        `PM10: ${pmData.value.pm10} μg/m³`,
        new Date().toLocaleTimeString()
      ];
  
      const textFaces = [0, 1, 4, 5]; // 四個顯示文字的面
  
      for (let i = 0; i < textFaces.length; i++) {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');
        canvas.width = 256;
        canvas.height = 256;
  
        ctx.fillStyle = '#000';
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = '#FFF';
        ctx.font = 'Bold 32px Arial';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(texts[i], canvas.width / 2, canvas.height / 2);
  
        const texture = new THREE.CanvasTexture(canvas);
        materials[textFaces[i]].map = texture;
        materials[textFaces[i]].needsUpdate = true;
      }
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
  }
  </style>
  