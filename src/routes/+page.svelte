<script lang="ts">
	import maplibregl from 'maplibre-gl';
	import 'maplibre-gl/dist/maplibre-gl.css';
	import issIcon from '$lib/assets/iss.png';
	import recentreIcon from '$lib/assets/recentre.png';
	import * as THREE from 'three';
	// @ts-ignore
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
	// @ts-ignore
	import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';
	import { Easing, Group, Tween } from '@tweenjs/tween.js';

	let currentPosition: { lng: number; lat: number } = { lng: 0, lat: 0 };
	let targetPosition: { lng: number; lat: number } = { lng: 0, lat: 0 };
	let followISS = $state(false);
	let isLoading = $state(true);
	let map: maplibregl.Map;
	const tweenGroup = new Group();

	$effect(() => {
		(async () => {
			const res = await fetch('https://api.wheretheiss.at/v1/satellites/25544');
			const data = await res.json();
			currentPosition = { lng: data.longitude, lat: data.latitude };
			targetPosition = { ...currentPosition };

			function animateTo(newLng: number, newLat: number, duration: number = 5000) {
				new Tween(currentPosition, tweenGroup)
					.to({ lng: newLng, lat: newLat }, duration)
					.easing(Easing.Linear.None)
					.start();
			}

			async function fetchISS() {
				const res = await fetch('https://api.wheretheiss.at/v1/satellites/25544');
				const data = await res.json();
				targetPosition = { lng: data.longitude, lat: data.latitude };
				animateTo(data.longitude, data.latitude, 5000);
			}
			setInterval(fetchISS, 5000); // update every 5 seconds

			map = new maplibregl.Map({
				container: 'map', // container id
				style: {
					version: 8,
					projection: {
						type: 'globe'
					},
					sources: {
						satellite: {
							type: 'raster',
							tiles: [
								// Example: Esri World Imagery
								'https://services.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}'
							],
							tileSize: 256,
							attribution:
								"© <a href='https://www.esri.com/'>Esri</a>, " +
								"<a href='https://www.maxar.com/'>Maxar</a>, " +
								'Earthstar Geographics, and the GIS User Community'
						}
					},
					layers: [
						{
							id: 'satellite-layer',
							type: 'raster',
							source: 'satellite'
						}
					]
				},

				center: [0, 0], // starting position [lng, lat]
				zoom: 2, // starting zoom
				maxZoom: 5,

				minZoom: 1,
				attributionControl: false
			});

			const customLayer = {
				id: '3d-model',
				type: 'custom',
				renderingMode: '3d', // The layer MUST be marked as 3D in order to get the proper depth buffer with globe depths in it.
				camera: undefined as THREE.Camera | undefined,
				scene: undefined as THREE.Scene | undefined,
				map: undefined as maplibregl.Map | undefined,
				renderer: undefined as THREE.WebGLRenderer | undefined,
				onAdd(map: maplibregl.Map, gl: WebGLRenderingContext) {
					this.camera = new THREE.Camera();
					this.scene = new THREE.Scene();

					this.scene.add(new THREE.AmbientLight(0xffffff, 0.6));

					const sun = new THREE.DirectionalLight(0xffffff, 1.2);
					sun.position.set(100, 100, 200);
					this.scene.add(sun);

					// use the three.js GLTF loader to add the 3D model to the three.js scene
					const loader = new GLTFLoader();
					const dracoLoader = new DRACOLoader();
					// Use Google’s CDN for Draco decoder binaries
					dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');

					loader.setDRACOLoader(dracoLoader);
					loader.load('/iss.glb', (gltf: any) => {
						gltf.scene.matrixAutoUpdate = false; // Important for custom matrix
						gltf.scene.frustumCulled = false; // Prevent small jitters
						this.scene!.add(gltf.scene);
					});
					this.map = map;

					// use the MapLibre GL JS map canvas for three.js
					this.renderer = new THREE.WebGLRenderer({
						// canvas: map.getCanvsas(),
						context: gl
					});

					this.renderer.autoClear = false;

					this.renderer.outputEncoding = THREE.sRGBEncoding;
					this.renderer.toneMapping = THREE.ACESFilmicToneMapping;
					this.renderer.toneMappingExposure = 1.5;
					const pixelRatio = window.devicePixelRatio;
					this.renderer.setPixelRatio(pixelRatio);
				},
				render(_gl: WebGLRenderingContext, args: any) {
					const now = performance.now();
					tweenGroup.update(now);
					const modelOrigin = [currentPosition.lng, currentPosition.lat];
					const modelAltitude = 637_100.0 + 408.0; // meters (earth radius + average iss altitude)

					const scaling = 30_000.0;

					const modelMatrix = this.map!.transform.getMatrixForModel(modelOrigin, modelAltitude);
					const m = new THREE.Matrix4().fromArray(args.defaultProjectionData.mainMatrix);
					const l = new THREE.Matrix4()
						.fromArray(modelMatrix)
						.scale(new THREE.Vector3(scaling, scaling, scaling));

					this.camera!.projectionMatrix = m.multiply(l);
					this.renderer!.resetState();
					this.renderer!.render(this.scene!, this.camera!);

					this.map!.triggerRepaint();
					if (followISS) {
						this.map!.setCenter([currentPosition.lng, currentPosition.lat]); // Center the map on the ISS
					}
				}
			};

			map.on('style.load', () => {
				map.addLayer(customLayer);

				map.flyTo({
					center: [targetPosition.lng, targetPosition.lat],
					zoom: 4,
					duration: 8000
				});
				isLoading = false;
			});
		})();
	});

	function toggleFollow() {
		followISS = !followISS;
		if (followISS) {
			map.setZoom(4);
			map.scrollZoom.disable();
			map.boxZoom.disable();
			map.dragRotate.disable();
			map.dragPan.disable();
			map.keyboard.disable();
			map.doubleClickZoom.disable();
			map.touchZoomRotate.disable();
		} else {
			map.scrollZoom.enable();
			map.boxZoom.enable();
			map.dragRotate.enable();
			map.dragPan.enable();
			map.keyboard.enable();
			map.doubleClickZoom.enable();
			map.touchZoomRotate.enable();
		}
	}
</script>

<div
	class="absolute inset-0 z-50 flex flex-col items-center justify-center gap-2 bg-black/20 text-white backdrop-blur-2xl
         transition-opacity duration-1000 ease-in-out"
	class:opacity-100={isLoading}
	class:opacity-0={!isLoading}
	class:pointer-events-none={!isLoading}
>
	<h1>International Space Station Tracker</h1>
	<div class="text-xs text-zinc-300">Loading...</div>
</div>
<div class="h-screen w-full bg-black" id="map">
	{#if !isLoading}
		<div class="absolute top-2 right-2 z-100 flex flex-col">
			<button
				class=" inline-flex items-center gap-2 rounded-2xl bg-white px-4 py-1 hover:cursor-pointer"
				onclick={toggleFollow}
			>
				<img class="h-4 w-4" src={issIcon} alt="Internation Space Station Icon" />
				{!followISS ? 'Follow ISS' : 'Unfollow ISS'}</button
			>
		</div>
		<div class="absolute right-2 bottom-2 z-100 text-neutral-500">
			Map Libre | © Esri, Maxar, Earthstar Geographics, and the GIS User Community
		</div>
		<div class="absolute right-2 bottom-10 z-100">
			<button
				onclick={() => map.flyTo({ center: [currentPosition.lng, currentPosition.lat], zoom: 4 })}
				class="mt-2 inline-flex items-center gap-2 rounded-2xl border border-white bg-transparent px-2 py-1 text-white hover:cursor-pointer"
			>
				<img class="h-4 w-4 invert" src={recentreIcon} alt="Recentre Icon" />
				Re-centre</button
			>
		</div>
	{/if}
</div>
