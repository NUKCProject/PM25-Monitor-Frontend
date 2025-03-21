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
let isDragging = false;
let previousMousePosition = { x: 0, y: 0 };
let previousTime = performance.now();

/**
 * Initializes the Three.js scene.
 *
 * This function creates a new Three.js scene, sets up the camera and renderer
 * objects, and adds the renderer's DOM element to the page.
 *
 * @return {THREE.Scene} The initialized scene.
 */

function initScene() {
	// build the scene
	const scene = new THREE.Scene();
	// build the camera (fov aspect near/far )
	camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
	// 
	renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
	renderer.setSize(window.innerWidth, window.innerHeight);
	threeContainer.value.appendChild(renderer.domElement);
	return scene;
}

/**
 * Initializes the Three.js lights.
 *
 * This function adds two lights to the Three.js scene. The first is a global
 * ambient light with a low intensity, while the second is a high-intensity
 * directional light pointing down at the cube.
 *
 * @param {THREE.Scene} scene The scene to add the lights to.
 */
function initLights(scene) {
	scene.add(new THREE.AmbientLight(0xffffff, 5));
	const light = new THREE.DirectionalLight(0xffffff, 10);
	light.position.set(0, 10, 50);
	scene.add(light);
}

/**
 * Initializes the cube.
 *
 * This function creates a new cube with the specified dimensions using
 * the RoundedBoxGeometry class. It then creates six MeshStandardMaterial
 * objects and assigns them to the cube's material array. Finally, it adds
 * the cube to the scene and positions the camera to view the cube.
 *
 * @param {THREE.Scene} scene The scene to add the cube to.
 */
function initCube(scene) {
	materials = Array.from({ length: 6 }, () =>
		new THREE.MeshStandardMaterial({
			color: 0xffffff,
			metalness: 1,
			roughness: 0.1,
			transparent: true,
			opacity: 1
		})
	);

	const geometry = new RoundedBoxGeometry(3, 3, 3, 12, 0.3);
	cube = new THREE.Mesh(geometry, materials);
	scene.add(cube);
	camera.position.z = 6;
}

/**
 * Initializes the mouse and touch event listeners for the cube.
 *
 * This function adds event listeners to the renderer's DOM element to handle
 * mouse and touch events. It tracks when the user is dragging the cube and
 * updates the cube's rotation accordingly.
 */
function initControls() {
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
}

/**
 * Animates the cube by rendering it every frame and incrementing its rotation
 * when the user is not dragging it.
 *
 * This function is called by the Three.js requestAnimationFrame function and
 * updates the cube's rotation every frame. When the user is not dragging the
 * cube, the cube's rotation is incremented at a rate of 0.5 degrees per second.
 * The cube is then rendered with the camera.
 */
function animate() {
	requestAnimationFrame(animate);

	const currentTime = performance.now();
	const deltaTime = (currentTime - previousTime) / 1000;
	previousTime = currentTime;

	if (!isDragging) cube.rotation.y += deltaTime * 0.5;
	renderer.render(cube.parent, camera);
}

/**
 * Fetches the latest PM data from the server and updates the cube text.
 *
 * This function fetches the latest PM data from the server using the Fetch
 * API. It then updates the cube text by calling updateCubeText(). If the
 * fetch fails, it calls showWaitingText() to display a waiting message.
 */
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

/**
 * Updates the text displayed on each face of the cube.
 *
 * This function creates text textures for specific faces of the cube
 * based on the latest particulate matter (PM) data. It applies these
 * textures to the corresponding materials of the cube's faces. If no
 * text is specified for a face, it applies an empty texture. The function
 * also sets the emissive properties for the materials to give a subtle
 * glow effect.
 */

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
			materials[i].emissive = new THREE.Color(0xffffff);
			materials[i].emissiveIntensity = 0.1;
			materials[i].emissiveMap = texture;
		} else {
			materials[i].map = createEmptyTexture();
			materials[i].emissiveMap = null;
		}
		materials[i].needsUpdate = true;
	}
}

/**
 * Creates an empty texture with a solid black background.
 *
 * This function initializes a canvas element and sets its dimensions
 * to 512x512 pixels. It fills the entire canvas with a black color,
 * creating a blank texture. The resulting texture is then converted
 * into a THREE.CanvasTexture object for use in Three.js materials.
 *
 * @return {THREE.CanvasTexture} A texture filled with a black color.
 */

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

/**
 * Creates a text texture from the given string.
 *
 * This function initializes a canvas element and sets its dimensions
 * to 512x512 pixels. It fills the canvas with a black background and
 * draws each line of the input text centered on the canvas, using a
 * white font with a cyan shadow effect. The text is split by new lines
 * and rendered with a specific line height to ensure proper spacing.
 * The resulting texture is then converted into a THREE.CanvasTexture
 * object for use in Three.js materials.
 *
 * @param {string} text - The text to render onto the texture.
 * @return {THREE.CanvasTexture} A texture containing the rendered text.
 */

function createTextTexture(text) {
	const canvas = document.createElement('canvas');
	const context = canvas.getContext('2d');
	const size = 512;
	canvas.width = size;
	canvas.height = size;

	context.fillStyle = '#000000';
	context.fillRect(0, 0, size, size);

	context.font = '40px Arial';
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

onMounted(() => {
	const scene = initScene();
	initLights(scene);
	initCube(scene);
	initControls();
	animate();
	fetchPMData();
	fetchInterval = setInterval(fetchPMData, 1000);
});

onUnmounted(() => {
	clearInterval(fetchInterval);
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
</style>
