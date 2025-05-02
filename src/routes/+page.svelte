<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import {
		FaceLandmarker,
		FilesetResolver,
		DrawingUtils,
		type FaceLandmarkerResult
	} from '@mediapipe/tasks-vision';

	let videoElement: HTMLVideoElement;
	let canvasElement: HTMLCanvasElement;
	let stream: MediaStream;
	let faceLandmarker: FaceLandmarker;
	let running = false;
	let lastVideoTime = -1;
	let result: FaceLandmarkerResult;

	async function initializeFaceLandmarker() {
		try {
			const filesetResolver = await FilesetResolver.forVisionTasks(
				'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0/wasm'
			);
			faceLandmarker = await FaceLandmarker.createFromOptions(filesetResolver, {
				baseOptions: {
					modelAssetPath: '/models/face_landmarker.task',
					delegate: 'GPU'
				},
				numFaces: 1,
				outputFaceBlendshapes: true,
				runningMode: 'VIDEO'
			});

			running = true;
		} catch (error) {
			console.error('Error initializing face landmarker:', error);
		}
	}

	async function startCamera() {
		try {
			stream = await navigator.mediaDevices.getUserMedia({ video: true });
			videoElement.srcObject = stream;
			// Wait for the video to load metadata before starting detection
			videoElement.onloadedmetadata = () => {
				canvasElement.width = videoElement.videoWidth;
				canvasElement.height = videoElement.videoHeight;
				requestAnimationFrame(detectFaces);
			};
		} catch (err) {
			console.error('Error accessing camera:', err);
		}
	}

	async function detectFaces() {
		if (!running || !faceLandmarker || !videoElement || !canvasElement) {
			return;
		}

		if (videoElement.currentTime === lastVideoTime) {
			requestAnimationFrame(detectFaces);
			return;
		}
		lastVideoTime = videoElement.currentTime;
		const detections = faceLandmarker.detectForVideo(videoElement, Date.now());

		result = detections;
		const canvasCtx = canvasElement.getContext('2d');
		// @ts-ignore
		canvasCtx.save();
		// @ts-ignore
		canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);

		if (detections.faceLandmarks && detections.faceLandmarks.length > 0) {
			const drawingUtils = new DrawingUtils(canvasCtx!);

			for (const landmarks of detections.faceLandmarks) {
				drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_TESSELATION, {
					color: '#C0C0C070',
					lineWidth: 1
				});
				drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_RIGHT_EYE, {
					color: '#FF3030'
				});
				drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_RIGHT_EYEBROW, {
					color: '#FF3030'
				});
				drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_LEFT_EYE, {
					color: '#30FF30'
				});
				drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_LEFT_EYEBROW, {
					color: '#30FF30'
				});
				drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_FACE_OVAL, {
					color: '#E0E0E0'
				});
				drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_LIPS, {
					color: '#E0E0E0'
				});
				drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_RIGHT_IRIS, {
					color: '#FF3030'
				});
				drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_LEFT_IRIS, {
					color: '#30FF30'
				});
			}
		}
		canvasCtx?.restore();
		requestAnimationFrame(detectFaces);
	}

	async function initialize() {
		await initializeFaceLandmarker();
		await startCamera();
	}

	onMount(initialize);

	onDestroy(() => {
		if (stream) {
			stream.getTracks().forEach((track) => track.stop());
		}
		if (faceLandmarker) {
			faceLandmarker.close();
		}
		running = false;
	});
</script>

<div class="w-full p-10 min-h-screen overflow-hidden bg-gray-100">
  <div class="flex flex-col items-center justify-cente">
	<div class="flex w-full items-start justify-around lg:flex-row">
		<div class="relative">
			<video bind:this={videoElement} autoplay playsinline class="rounded-lg shadow-md"></video>
			<canvas bind:this={canvasElement} class="absolute top-0 left-0 rounded-lg shadow-md"></canvas>
     <div class="flex justify-center">
       <button
			on:click={startCamera}
			disabled={running}
			class="mt-4 rounded-md bg-blue-500 px-4 py-2 text-white hover:bg-blue-600 focus:outline-none"
		>
			{running ? 'Detecting...' : 'Start Camera'}
		</button>
     </div>
		</div>
		
		<div class="">
			{#if result && result.faceBlendshapes.length > 0}
				<ul>
					{#each result.faceBlendshapes[0].categories as shape}
						<li>
							<span>{shape.displayName || shape.categoryName}</span>
							<span>{shape.score.toFixed(4)}</span>
						</li>
					{/each}
				</ul>
			{/if}
		</div>
	</div>
</div>

</div>