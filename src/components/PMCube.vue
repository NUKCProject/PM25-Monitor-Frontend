<template>
	<div ref="threeContainer" class="three-container"></div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import { RoundedBoxGeometry } from 'three/examples/jsm/geometries/RoundedBoxGeometry.js';

const threeContainer = ref(null);
const pmData = ref({});
let fetchInterval = null;
let lastUpdate = Date.now();
let materials = [];
let camera, renderer, cube;

onMounted(() => {
	const scene = new THREE.Scene();
	camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
	renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
	renderer.setSize(window.innerWidth, window.innerHeight);
	threeContainer.value.appendChild(renderer.domElement);

	scene.add(new THREE.AmbientLight(0xffffff, 5));
	const light = new THREE.DirectionalLight(0xffffff, 10);
	light.position.set(0, 10, 50);
	scene.add(light);

	// ✅ 每面獨立材質
	materials = Array.from({ length: 6 }, () =>
		new THREE.MeshStandardMaterial({
			color: 0xffffff,          // 深色金屬
			metalness: 1,             // 完全金屬
			roughness: 0.1,           // 微微粗糙，增加反射細節
			transparent: true,
			opacity: 1
		})
	);

	const geometry = new RoundedBoxGeometry(3, 3, 3, 12, 0.3);
	cube = new THREE.Mesh(geometry, materials);
	scene.add(cube);

	camera.position.z = 6;

	let isDragging = false;
	let previousMousePosition = { x: 0, y: 0 };

	renderer.domElement.addEventListener('mousedown', (event) => {
		isDragging = true;
		previousMousePosition = { x: event.clientX, y: event.clientY };
	});

	renderer.domElement.addEventListener('mousemove', (event) => {
		if (!isDragging) return;
		const deltaX = event.clientX - previousMousePosition.x;
		const deltaY = event.clientY - previousMousePosition.y;
		cube.rotation.y += deltaX * 0.005;
		cube.rotation.x += deltaY * 0.005;
		previousMousePosition = { x: event.clientX, y: event.clientY };
	});

	renderer.domElement.addEventListener('mouseup', () => isDragging = false);
	renderer.domElement.addEventListener('mouseleave', () => isDragging = false);

	const animate = () => {
		requestAnimationFrame(animate);
		if (!isDragging) cube.rotation.y += 0.001;
		renderer.render(scene, camera);
	};
	animate();

	fetchPMData();
	fetchInterval = setInterval(() => {
		fetchPMData();
		if (Date.now() - lastUpdate > 5000) showWaitingText();
	}, 1000);

	function fetchPMData() {
		fetch('https://aiot-451916.de.r.appspot.com/latest-pm')
			.then(res => res.json())
			.then(data => {
				pmData.value = data;
				lastUpdate = Date.now();
				updateCubeText();
			})
			.catch(() => showWaitingText());
	}

	function updateCubeText() {
		const faceTexts = {
			0: `CF=1\nPM1.0: ${pmData.value.cf_pm1_0} μg/m³\nPM2.5: ${pmData.value.cf_pm2_5} μg/m³\nPM10: ${pmData.value.cf_pm10} μg/m³`,
			1: `ATM\nPM1.0: ${pmData.value.atm_pm1_0} μg/m³\nPM2.5: ${pmData.value.atm_pm2_5} μg/m³\nPM10: ${pmData.value.atm_pm10} μg/m³`,
			4: `Particles\n>1.0μm: ${pmData.value.particles_1_0}\n>2.5μm: ${pmData.value.particles_2_5}\n>10μm: ${pmData.value.particles_10}`,
			5: new Date().toLocaleTimeString()
		};

		for (let i = 0; i < materials.length; i++) {
			if (faceTexts[i]) {
				const texture = createTextTexture(faceTexts[i]);
				texture.center.set(0.5, 0.5);
				texture.rotation = Math.PI;
				materials[i].map = texture;

				// ✅ 套用發光效果
				materials[i].emissive = new THREE.Color(0xffffff); // 白色發光
				materials[i].emissiveIntensity = 0.1; // 發光強度，可調整
				materials[i].emissiveMap = texture; // 將文字貼圖當作發光貼圖
			} else {
				const emptyTexture = createEmptyTexture();
				materials[i].map = emptyTexture;
				materials[i].emissiveMap = null; // 清除空白面的發光
			}
			materials[i].needsUpdate = true;
		}

	}

	function createEmptyTexture() {
		const canvas = document.createElement('canvas');
		const context = canvas.getContext('2d');
		const size = 512;
		canvas.width = size;
		canvas.height = size;

		context.fillStyle = '#000000';
		context.fillRect(0, 0, size, size);

		return new THREE.CanvasTexture(canvas);
	}

	function showWaitingText() {
		const waitingTexture = createTextTexture('等待更新...', '#555');
		waitingTexture.center.set(0.5, 0.5);
		waitingTexture.rotation = Math.PI;
		[0, 1, 4, 5].forEach(i => {
			materials[i].map = waitingTexture;
			materials[i].needsUpdate = true;
		});
	}

	function createTextTexture(text) {
		const canvas = document.createElement('canvas');
		const context = canvas.getContext('2d');
		const size = 512;
		canvas.width = size;
		canvas.height = size;

		context.fillStyle = '#000000';
		context.fillRect(0, 0, size, size);

		context.font = ' 40px Arial';
		context.textAlign = 'center';
		context.textBaseline = 'middle';
		context.fillStyle = '#FFFFFF';
		context.shadowColor = '#00FFFF';
		context.shadowBlur = 10;

		const lines = text.split('\n');
		const lineHeight = 50;
		const startY = (size / 2) - ((lines.length - 1) * lineHeight) / 2;

		lines.forEach((line, index) => {
			context.fillText(line, size / 2, startY + index * lineHeight);
		});

		return new THREE.CanvasTexture(canvas);
	}

	window.addEventListener('resize', onWindowResize);

	function onWindowResize() {
		camera.aspect = window.innerWidth / window.innerHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(window.innerWidth, window.innerHeight);
	}
	// 滑鼠事件
	renderer.domElement.addEventListener('mousedown', (event) => {
		isDragging = true;
		previousMousePosition = { x: event.clientX, y: event.clientY };
	});
	renderer.domElement.addEventListener('mousemove', (event) => {
		if (!isDragging) return;
		const deltaX = event.clientX - previousMousePosition.x;
		const deltaY = event.clientY - previousMousePosition.y;
		cube.rotation.y += deltaX * 0.005;
		cube.rotation.x += deltaY * 0.005;
		previousMousePosition = { x: event.clientX, y: event.clientY };
	});
	renderer.domElement.addEventListener('mouseup', () => isDragging = false);
	renderer.domElement.addEventListener('mouseleave', () => isDragging = false);

	// ✅ 手機觸控事件
	renderer.domElement.addEventListener('touchstart', (event) => {
		isDragging = true;
		const touch = event.touches[0];
		previousMousePosition = { x: touch.clientX, y: touch.clientY };
	});
	renderer.domElement.addEventListener('touchmove', (event) => {
		if (!isDragging) return;
		const touch = event.touches[0];
		const deltaX = touch.clientX - previousMousePosition.x;
		const deltaY = touch.clientY - previousMousePosition.y;
		cube.rotation.y += deltaX * 0.005;
		cube.rotation.x += deltaY * 0.005;
		previousMousePosition = { x: touch.clientX, y: touch.clientY };
	});
	renderer.domElement.addEventListener('touchend', () => isDragging = false);
	renderer.domElement.addEventListener('touchcancel', () => isDragging = false);
});

onUnmounted(() => {
	clearInterval(fetchInterval);
	window.removeEventListener('resize', onWindowResize);
});
</script>

<style scoped>
.three-container {
	width: 100vw;
	height: 100vh;
	margin: 0;
	padding: 0;
	overflow: hidden;
	position: fixed;
	top: 0;
	left: 0;
	background-color: #111;
}

html,
body {
	width: 100%;
	height: 100%;
	margin: 0;
	padding: 0;
	overflow: hidden;
}
</style>
