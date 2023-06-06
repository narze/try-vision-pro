<script lang="ts">
	import Canvas from '../components/Canvas.svelte';
	import Picker from 'vanilla-picker';
	import { saveAs } from 'file-saver';
	import { copyImageToClipboard } from 'copy-image-clipboard';
	import * as htmlToImage from 'html-to-image';
	import { onMount } from 'svelte';
	import { Facebook, Twitter } from 'svelte-share-buttons-component';
	import * as base64 from 'base64util';
	import * as faceapi from '@vladmandic/face-api';
	// import vision from '../../static/images/vision.png';

	import { browser } from '$app/environment';
	import { page } from '$app/stores';
	import type { FaceLandmark68TinyNet } from '@vladmandic/face-api';

	let video: HTMLVideoElement;
	let canvas: HTMLCanvasElement;
	let displaySize: { width: number; height: number };
	let faceLandmarkOptions: FaceLandmark68TinyNet;

	let data = $page.url.searchParams.get('d') || '';
	let decodedData = data.split(',').map((d) => {
		try {
			return base64.urlDecode(d);
		} catch (e) {
			return null;
		}
	});

	// let text: string = decodedData[0] || `${new Date().getMonth() + 1}`;
	// let color: string = decodedData[1] || '#ff7300';
	// let pickerRef: HTMLButtonElement;

	const glassesImage = new Image();
	glassesImage.src = '/images/vision.png';

	$: ogImageUrl = `https://try-vision-pro.vercel.app`;
	// $: encodedData = `${base64.urlEncode(text)},${base64.urlEncode(color)}`;
	$: shareUrl = `https://try-vision-pro.vercel.app`;

	onMount(async () => {
		await faceapi.nets.faceLandmark68TinyNet.loadFromUri('/models');
		await faceapi.nets.ssdMobilenetv1.loadFromUri('/models');

		let stream;
		const constraints = {
			audio: false,
			video: {
				facingMode: 'user'
			}
		};

		try {
			stream = await navigator.mediaDevices.getUserMedia(constraints);
		} catch (err: any) {
			if (err.name === 'PermissionDeniedError' || err.name === 'NotAllowedError')
				alert(`Camera Error: camera permission denied: ${err.message || err}`);
			if (err.name === 'SourceUnavailableError')
				alert(`Camera Error: camera not available: ${err.message || err}`);
			return null;
		}

		if (stream) {
			video.srcObject = stream;
		} else {
			console.error('no stream');
		}

		// return new Promise((resolve) => {
		video.onloadeddata = async () => {
			canvas.width = video.videoWidth;
			canvas.height = video.videoHeight;
			video.play();

			displaySize = { width: video.videoWidth, height: video.videoHeight };
			faceapi.matchDimensions(canvas, displaySize);

			await detectVideo(video, canvas);
		};
	});

	async function detectVideo(video: HTMLVideoElement, canvas: HTMLCanvasElement) {
		await drawVideo(video, canvas);

		// const t0 = performance.now();

		const faces = await faceapi.detectAllFaces(video).withFaceLandmarks(true);

		if (faces.length) {
			const resizedResults = faceapi.resizeResults(faces, displaySize);
			drawFace(canvas, resizedResults);
		}

		requestAnimationFrame(() => detectVideo(video, canvas));
	}

	function drawVideo(video: HTMLVideoElement, canvas: HTMLCanvasElement) {
		const ctx = canvas.getContext('2d');
		if (!ctx) return;

		ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
	}

	function drawFace(
		canvas: HTMLCanvasElement,
		data: faceapi.WithFaceLandmarks<
			{
				detection: faceapi.FaceDetection;
			},
			faceapi.FaceLandmarks68
		>[]
	) {
		const ctx = canvas.getContext('2d');
		if (!ctx) return;

		for (const person of data) {
			const { detection } = person;
			const { x, y, width, height } = detection.box;
			const glassesWidth = glassesImage.width;

			const face = person;

			const { angle } = face;
			const { roll, pitch, yaw } = angle as {
				roll: number;
				pitch: number;
				yaw: number;
			};
			const centerX = x + width / 2;
			const centerY = y + height / 2;

			// Save the current canvas state
			ctx.save();

			const left = [person.landmarks.positions[0].x, person.landmarks.positions[0].y];
			const right = [person.landmarks.positions[16].x, person.landmarks.positions[16].y];
			const landmarkWidth = Math.sqrt(
				Math.pow(left[0] - right[0], 2) + Math.pow(left[1] - right[1], 2)
			);
			const up = person.landmarks.positions[19].y - person.landmarks.positions[0].y;

			ctx.translate(left[0] - landmarkWidth * 0.1, left[1] + up * 1.6);
			ctx.rotate(-(roll * Math.PI) / 180);

			const ratio = (landmarkWidth / glassesWidth) * 1.2;

			ctx.drawImage(
				glassesImage,
				// -glassesWidth / 2,
				// -glassesImage.height / 2,
				// centerX,
				0,
				// centerY,
				0,
				glassesWidth * ratio,
				glassesImage.height * ratio
			);
			ctx.restore();

			// draw box around each face
			// ctx.lineWidth = 3;
			// ctx.strokeStyle = 'deepskyblue';
			// ctx.fillStyle = 'deepskyblue';
			// ctx.globalAlpha = 0.6;
			// ctx.beginPath();
			// ctx.rect(
			// 	person.detection.box.x,
			// 	person.detection.box.y,
			// 	person.detection.box.width,
			// 	person.detection.box.height
			// );
			// ctx.stroke();
			// ctx.globalAlpha = 1;
			// ctx.fillStyle = 'black';
			// ctx.fillText(
			// 	`roll:${person.angle.roll}° pitch:${person.angle.pitch}° yaw:${person.angle.yaw}°`,
			// 	person.detection.box.x,
			// 	person.detection.box.y - 5
			// );
			// ctx.fillStyle = 'lightblue';
			// ctx.fillText(
			// 	`roll:${person.angle.roll}° pitch:${person.angle.pitch}° yaw:${person.angle.yaw}°`,
			// 	person.detection.box.x,
			// 	person.detection.box.y - 6
			// );
			// // draw face points for each face
			// ctx.globalAlpha = 0.8;
			// ctx.fillStyle = 'lightblue';
			// const pointSize = 2;
			// for (let i = 0; i < person.landmarks.positions.length; i++) {
			// 	ctx.beginPath();
			// 	ctx.arc(
			// 		person.landmarks.positions[i].x,
			// 		person.landmarks.positions[i].y,
			// 		pointSize,
			// 		0,
			// 		2 * Math.PI
			// 	);
			// 	ctx.fill();
			// }
		}
	}

	function copyImage() {
		htmlToImage
			.toPng(canvas)
			.then(function (dataUrl) {
				const img = new Image();
				img.src = dataUrl;
				copyImageToClipboard(img.src);
				// saving = false
			})
			.catch(function (error) {
				console.error('oops, something went wrong!', error);
				console.error(error.message);
			});
	}

	function saveImage() {
		htmlToImage
			.toPng(canvas)
			.then(function (blob) {
				saveAs(blob, `try-vision-pro.png`);
				// saving = false
			})
			.catch(function (error) {
				console.error('oops, something went wrong!', error);
			});
	}

	// OG
	const title = 'Try Vision Pro';
	const description = '';
</script>

<svelte:head>
	<title>{title}</title>

	<meta name="title" content={title} />
	<meta name="description" content={description} />
	<meta name="viewport" content="width=device-width, initial-scale=1" />
	<meta property="og:url" content={'https://try-vision-pro.vercel.app'} />
	<meta property="og:type" content="website" />
	<meta property="og:title" content={title} />
	<meta property="og:description" content={description} />
	<meta property="og:image" content={ogImageUrl} />
	<meta name="twitter:title" content={title} />
	<meta name="twitter:card" content="summary_large_image" />
	<meta name="twitter:image" content={ogImageUrl} />
</svelte:head>

<div class="container hero">
	<h1 class="text-6xl">Try Vision Pro</h1>

	<!-- svelte-ignore a11y-media-has-caption -->
	<video bind:this={video} id="video" playsinline class="video" style="display: none;" />
	<canvas bind:this={canvas} />

	<div class="flex gap-4">
		<button
			on:click={() => copyImage()}
			class="bg-blue-500 hover: text-white font-bold py-2 px-4 rounded"
		>
			ก็อปภาพ
		</button>
		<button
			on:click={() => saveImage()}
			class="bg-blue-500 hover: text-white font-bold py-2 px-4 rounded"
		>
			เซฟภาพ
		</button>
	</div>

	<div class="flex gap-2 justify-center items-center w-full bottom-4 center">
		<span class="text-lg"> Share: </span>
		<Facebook class="h-10 w-10 rounded" url={shareUrl} text="" />
		<Twitter class="h-10 w-10 rounded" url={shareUrl} text="" />
	</div>

	<!-- <Canvas bind:color bind:text bind:imageDom /> -->
</div>

<style lang="postcss">
	.hero {
		display: flex;
		vertical-align: middle;
		flex-direction: column;
		flex-wrap: nowrap;
		align-items: center;
		justify-content: center;
		gap: 40px;
	}

	@media screen and (min-width: 768px) {
		.hero {
			gap: 60px;
		}
	}
</style>
