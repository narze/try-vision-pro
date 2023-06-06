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

	import { browser } from '$app/environment';
	import { page } from '$app/stores';
	import type { FaceLandmark68TinyNet } from '@vladmandic/face-api';

	let video: HTMLVideoElement;
	let canvas: HTMLCanvasElement;
	let faceLandmarkOptions: FaceLandmark68TinyNet;

	let data = $page.url.searchParams.get('d') || '';
	let decodedData = data.split(',').map((d) => {
		try {
			return base64.urlDecode(d);
		} catch (e) {
			return null;
		}
	});

	let text: string = decodedData[0] || `${new Date().getMonth() + 1}`;
	let color: string = decodedData[1] || '#ff7300';
	let pickerRef: HTMLButtonElement;
	let imageDom: HTMLElement;

	$: ogImageUrl = `https://sunny-pass.vercel.app/i?t=${text}&c=${color.replace('#', '')}`;
	$: encodedData = `${base64.urlEncode(text)},${base64.urlEncode(color)}`;
	$: shareUrl = `https://sunny-pass.vercel.app?d=${encodedData}`;

	onMount(async () => {
		await faceapi.nets.faceLandmark68TinyNet.loadFromUri('/models');
		await faceapi.nets.ssdMobilenetv1.loadFromUri('/models');

		// faceLandmarkOptions = new faceapi.FaceLandmark68TinyNet();

		let stream;
		const constraints = {
			audio: false,
			video: {
				facingMode: 'user'
				// width: { ideal: 4096 },
				// height: { ideal: 2160 },
			}
		};

		try {
			stream = await navigator.mediaDevices.getUserMedia(constraints);
		} catch (err) {
			console.error({ err });

			// if (err.name === 'PermissionDeniedError' || err.name === 'NotAllowedError') log(`Camera Error: camera permission denied: ${err.message || err}`);
			// if (err.name === 'SourceUnavailableError') log(`Camera Error: camera not available: ${err.message || err}`);
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

			await detectVideo(video, canvas);
			// detectVideo(video, canvas);
			// resolve(true);
		};
	});

	async function detectVideo(video: HTMLVideoElement, canvas: HTMLCanvasElement) {
		const t0 = performance.now();

		const result = await faceapi.detectSingleFace(video).withFaceLandmarks(true);

		console.log({ result });

		if (result) {
		}

		requestAnimationFrame(() => detectVideo(video, canvas));
		// .withFaceExpressions()
		// .withFaceDescriptors()
		// .withAgeAndGender()
		// .then((result: any) => {
		// 	const fps = 1000 / (performance.now() - t0);
		// 	// drawFaces(canvas, result, fps.toLocaleString());
		// 	console.log({ result });

		// 	requestAnimationFrame(() => detectVideo(video, canvas));
		// 	return true;
		// })
		// .catch((err) => {
		// 	console.error({ err });

		// 	return false;
		// });
	}

	function copyImage() {
		htmlToImage
			.toPng(imageDom)
			.then(function (dataUrl) {
				const img = new Image();
				img.src = dataUrl;
				copyImageToClipboard(img.src);
				// saving = false
			})
			.catch(function (error) {
				console.error('oops, something went wrong!', error);
				console.log(error.message);
			});
	}

	function saveImage() {
		htmlToImage
			.toPng(imageDom)
			.then(function (blob) {
				saveAs(blob, `sunny-pass.png`);
				// saving = false
			})
			.catch(function (error) {
				console.error('oops, something went wrong!', error);
			});
	}

	// OG
	const title = 'Sunny Pass';
	const description = 'แคล้วคลาดปลอดภัย';
</script>

<svelte:head>
	<title>{title}</title>

	<meta name="title" content={title} />
	<meta name="description" content={description} />
	<meta name="viewport" content="width=device-width, initial-scale=1" />
	<meta property="og:url" content={'https://sunny-pass.vercel.app'} />
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
	<video bind:this={video} id="video" playsinline class="video" />
	<canvas
		bind:this={canvas}
		id="canvas"
		class="canvas"
		style="position: fixed; top: 0; left: 0; z-index: 10"
	/>
	<div id="log" style="overflow-y: scroll; height: 16.5rem" />

	<div class="flex gap-4">
		<button bind:this={pickerRef} class="bg-blue-500 hover: text-white font-bold py-2 px-4 rounded">
			เปลี่ยนสี
		</button>
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

	<div class="flex items-center justify-center text-black">
		<input
			type="text"
			bind:value={text}
			placeholder="เดือน (?)"
			class="text-center rounded p-2 text-xl"
		/>
	</div>

	<div class="flex gap-2 justify-center items-center w-full bottom-4 center">
		<span class="text-lg"> Share: </span>
		<Facebook class="h-10 w-10 rounded" url={shareUrl} text="สร้างสติ๊กเกอร์ของคุณได้ที่นี่" />
		<Twitter class="h-10 w-10 rounded" url={shareUrl} text="สร้างสติ๊กเกอร์ของคุณได้ที่นี่" />
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
