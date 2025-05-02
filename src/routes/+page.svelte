<script lang="ts">
    import { onMount, onDestroy } from 'svelte';
    import {
      FaceLandmarker,
      FilesetResolver,
      DrawingUtils,
    } from '@mediapipe/tasks-vision';
  
    let videoElement: HTMLVideoElement;
    let canvasElement: HTMLCanvasElement;
    let stream: MediaStream;
    let faceLandmarker: FaceLandmarker;
    let running = false;
    let lastVideoTime = -1;
  
    async function initializeFaceLandmarker() {
      try {
        const filesetResolver = await FilesetResolver.forVisionTasks(
          'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.0/wasm'
        );
        faceLandmarker = await FaceLandmarker.createFromOptions(filesetResolver, {
          baseOptions: {
            modelAssetPath: '/models/face_landmarker.task',
            delegate: 'CPU' // Or 'CPU'
          },
          outputFaceBlendshapes: true,
          outputFacialTransformationMatrixes: true,
        });
        await faceLandmarker.setOptions({runningMode: 'VIDEO'});
        running = true;
        detectFaces();
      } catch (error) {
        console.error("Error initializing face landmarker:", error);
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
        console.error("Error accessing camera:", err);
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
      
      const canvasCtx = canvasElement.getContext('2d');
      // @ts-ignore
      canvasCtx.save();
      // @ts-ignore
      canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
  
      if (detections.faceLandmarks && detections.faceLandmarks.length > 0) {
        // @ts-ignore
        const drawingUtils = new DrawingUtils(canvasCtx);
        for (const landmarks of detections.faceLandmarks) {
          drawingUtils.drawConnectors(
            landmarks,
            FaceLandmarker.FACE_LANDMARKS_TESSELATION,
            { color: '#C0C0C0', lineWidth: 1 }
          );
          drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_RIGHT_EYE, {
            color: '#FF3030',
          });
          drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_RIGHT_EYEBROW, {
            color: '#FF3030',
          });
          drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_LEFT_EYE, {
            color: '#30FF30',
          });
          drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_LEFT_EYEBROW, {
            color: '#30FF30',
          });
          // @ts-ignore
          drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_UPPER_LIPS, {
            color: '#E0E0E0',
          });
          // @ts-ignore
          drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_LOWER_LIPS, {
            color: '#E0E0E0',
          });
          // @ts-ignore
          drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_RIGHT_CHEEK, {
            color: '#FF0000',
          });
          // @ts-ignore
          drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_LEFT_CHEEK, {
            color: '#00FF00',
          });
          drawingUtils.drawConnectors(landmarks, FaceLandmarker.FACE_LANDMARKS_FACE_OVAL, {
            color: '#E0E0E0',
          });
          drawingUtils.drawLandmarks(landmarks, {
            color: '#F0E68C',
            radius: 2,
          });
        }
      }
      // @ts-ignore
      canvasCtx.restore();
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
  
  <div class="flex flex-col items-center justify-center h-screen bg-gray-100">
    <div class="relative">
      <video bind:this={videoElement} autoplay class="rounded-lg shadow-md"></video>
      <canvas bind:this={canvasElement} class="absolute top-0 left-0 rounded-lg shadow-md"></canvas>
    </div>
    <button on:click={startCamera} class="mt-4 px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 focus:outline-none" disabled={running}>
      {running ? 'Detecting...' : 'Start Camera'}
    </button>
  </div>