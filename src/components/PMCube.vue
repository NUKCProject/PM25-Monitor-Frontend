<template>
	<div ref="threeContainer" class="three-container"></div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { RoundedBoxGeometry } from 'three/examples/jsm/geometries/RoundedBoxGeometry.js';

const threeContainer = ref(null);
const pmData = ref({});
let fetchInterval = null;
let lastUpdate = Date.now();
let materials = [];
let camera, renderer;

onMounted(() => {
	const scene = new THREE.Scene();
	camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
	renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
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
		const texts = [
			`CF=1\nPM1.0: ${pmData.value.cf_pm1_0} μg/m³\nPM2.5: ${pmData.value.cf_pm2_5} μg/m³\nPM10: ${pmData.value.cf_pm10} μg/m³`,
			`ATM\nPM1.0: ${pmData.value.atm_pm1_0} μg/m³\nPM2.5: ${pmData.value.atm_pm2_5} μg/m³\nPM10: ${pmData.value.atm_pm10} μg/m³`,
			`Particles\n>1.0μm: ${pmData.value.particles_1_0}\n>2.5μm: ${pmData.value.particles_2_5}\n>10μm: ${pmData.value.particles_10}`,
			new Date().toLocaleTimeString()
		];

		const faces = [0, 1, 4, 5];

		for (let i = 0; i < faces.length; i++) {
			const texture = createTextTexture(texts[i]);
			texture.center.set(0.5, 0.5);
			texture.rotation = Math.PI;
			materials[faces[i]].map = texture;
			materials[faces[i]].needsUpdate = true;
		}
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

		context.font = 'bold 40px Arial';
		context.textAlign = 'center';
		context.textBaseline = 'middle';
		context.fillStyle = '#FFFFFF';
		context.shadowColor = '#00FFFF';
		context.shadowBlur = 30;

		const lines = text.split('\n');
		const lineHeight = 50;
		const startY = (size / 2) - ((lines.length - 1) * lineHeight) / 2;

		lines.forEach((line, index) => {
			context.fillText(line, size / 2, startY + index * lineHeight);
		});

		return new THREE.CanvasTexture(canvas);
	}

	// RWD 監聽
	window.addEventListener('resize', onWindowResize);

	function onWindowResize() {
		camera.aspect = window.innerWidth / window.innerHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(window.innerWidth, window.innerHeight);
	}
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
