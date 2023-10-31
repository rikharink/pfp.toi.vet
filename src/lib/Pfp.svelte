<script lang="ts">
	import { createEventDispatcher } from 'svelte';
	const dispatch = createEventDispatcher();

	let output: HTMLDivElement;

	export let isAnonymous = false;
	export let treshold = 50;
	export let padding = 10;
	export let xOffset = 0;
	export let yOffset = 0;
	export let color = '#E31C79';

	export let blob: Blob | null = null;

	const canvas = document.createElement('canvas');
	const ctx = canvas.getContext('2d')!;
	const bg = new Image();
	const fg = new Image();

	$: onBlob();

	function onBlob() {
		if (!blob) return;
		bg.src = './topicus-background.jpg';
		bg.addEventListener('load', () => {
			canvas.width = bg.width;
			canvas.height = bg.height;
			fg.addEventListener('load', updateImage);
			fg.src = URL.createObjectURL(blob!);
		});
	}

	function updateImage() {
		if (isAnonymous) {
			recolorAnonymouseImage(fg).then((recolored) => {
				drawImage(recolored);
			});
		} else {
			cropTransparentImage(fg).then((cropped) => {
				drawImage(cropped);
			});
		}
	}

	function drawImage(img: HTMLImageElement) {
		// draw background
		ctx.drawImage(bg, 0, 0, bg.width, bg.height);

		// calculate width and height that makes sure that fg fits in bg but keeps its aspect ratio
		const ratio = Math.min(bg.width / img.width, bg.height / img.height);
		const width = img.width * ratio;
		const height = img.height * ratio;

		// calculate x and y so x is in the center and y is on the bottom
		const x = (bg.width - width) / 2 + xOffset;
		const y = bg.height - height + yOffset;

		ctx.drawImage(img, x, y, width, height);
		const dataUrl = canvas.toDataURL('image/png');
		output.innerHTML = `<img src="${dataUrl}" />`;
	}

	function recolorAnonymouseImage(img: HTMLImageElement): Promise<HTMLImageElement> {
		const canvas = document.createElement('canvas');
		const ctx = canvas.getContext('2d')!;
		canvas.width = img.width;
		canvas.height = img.height;
		ctx.drawImage(img, 0, 0, img.width, img.height);
		const data = ctx.getImageData(0, 0, img.width, img.height);
		for (let i = 0; i < data.data.length; i += 4) {
			if (data.data[i + 3] === 255) {
				data.data[i] = parseInt(color.slice(1, 3), 16);
				data.data[i + 1] = parseInt(color.slice(3, 5), 16);
				data.data[i + 2] = parseInt(color.slice(5, 7), 16);
			}
		}
		ctx.putImageData(data, 0, 0);
		const dataUrl = canvas.toDataURL('image/png');
		return new Promise((resolve) => {
			let result = new Image();
			result.addEventListener('load', () => {
				resolve(result);
			});
			result.src = dataUrl;
		});
	}

	function cropTransparentImage(img: HTMLImageElement): Promise<HTMLImageElement> {
		dispatch('message', 'cropping image...');
		let result = new Image();
		const canvas = document.createElement('canvas');
		const ctx = canvas.getContext('2d')!;
		canvas.width = img.width;
		canvas.height = img.height;
		ctx.drawImage(img, 0, 0, img.width, img.height);
		const data = ctx.getImageData(0, 0, img.width, img.height).data;
		let left = canvas.width,
			right = 0,
			top = canvas.height,
			bottom = 0;
		for (let y = 0; y < canvas.height; y++) {
			for (let x = 0; x < canvas.width; x++) {
				const alpha = data[(y * canvas.width + x) * 4 + 3];
				if (alpha > treshold) {
					if (x < left) left = x;
					if (x > right) right = x;
					if (y < top) top = y;
					if (y > bottom) bottom = y;
				}
			}
		}

		const croppedWidth = right - left + padding * 2;
		const croppedHeight = bottom - top + padding;
		const croppedImageData = ctx.getImageData(left, top, croppedWidth, croppedHeight);

		canvas.width = croppedWidth;
		canvas.height = croppedHeight;
		ctx.putImageData(croppedImageData, padding, padding);
		return new Promise((resolve) => {
			result.addEventListener('load', () => resolve(result));
			result.src = canvas.toDataURL();
			dispatch('message', '');
		});
	}

	function onChange(_: Event & { currentTarget: EventTarget & HTMLInputElement }) {
		updateImage();
	}
</script>

<div class="pfp">
	<div id="output" bind:this={output} />
	<div class="tweaks">
		{#if !isAnonymous}
			<div class="input">
				<label for="x-offset">horizontal offset</label>
				<input
					id="x-offset"
					type="range"
					bind:value={xOffset}
					on:change={onChange}
					min="-512"
					max="512"
					step="1"
				/>
			</div>
			<div class="input">
				<label for="y-offset">vertical offset</label>
				<input
					id="y-offset"
					type="range"
					bind:value={yOffset}
					on:change={onChange}
					min="-512"
					max="512"
					step="1"
				/>
			</div>
			<div class="input">
				<label for="alpha">transparency treshold</label>
				<input
					id="alpha"
					type="range"
					bind:value={treshold}
					on:change={onChange}
					min="0"
					max="255"
					step="1"
				/>
			</div>
			<div class="input">
				<label for="padding">padding</label>
				<input
					id="padding"
					type="range"
					bind:value={padding}
					on:change={onChange}
					min="0"
					max="100"
					step="1"
				/>
			</div>
		{:else}
			<div class="input">
				<label for="color">color</label>
				<input id="color" type="color" bind:value={color} on:change={onChange} />
			</div>
		{/if}
	</div>
</div>

<style>
	.pfp {
		display: flex;
		align-items: space-between;
		padding-bottom: 2em;
	}

	.input {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		padding-top: 1em;
	}

	.input label {
		font-size: 1.5rem;
		font-weight: bold;
		text-align: center;
		padding: 0 0 0.5em 0;
	}

	input[type='range'] {
		-webkit-appearance: none;
		appearance: none;
		width: 100%;
		height: 15px;
		border-radius: 5px;
		background: var(--dark);
		outline: none;
		-webkit-transition: 0.2s;
		transition: opacity 0.2s;
		margin: 1em 0;
	}

	input[type='range']::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		width: 25px;
		height: 25px;
		border-radius: 50%;
		border: 4px solid var(--dark);
		background: var(--yellow);
		cursor: pointer;
	}

	input[type='range']::-moz-range-thumb {
		width: 25px;
		height: 25px;
		border-radius: 50%;
		border: 4px solid var(--dark);
		background: var(--yellow);
		cursor: pointer;
	}

	input[type='color'] {
		-webkit-appearance: none;
		appearance: none;
		border-radius: 50%;
		height: 60px;
		width: 60px;
		border: none;
		outline: none;
	}

	input[type='color']::-webkit-color-swatch-wrapper {
		padding: 0;
	}

	input[type='color']::-moz-color-swatch-wrapper {
		padding: 0;
	}

	input[type='color']::-moz-color-swatch {
		border: none;
		border-radius: 100%;
	}

	input[type='color']::-webkit-color-swatch {
		border: none;
		border-radius: 100%;
	}

	.tweaks {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		padding: 1em;
		min-height: 100%;
	}
</style>
