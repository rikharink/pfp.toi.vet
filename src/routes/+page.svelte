<script lang="ts">
	import imglyRemoveBackground from "@imgly/background-removal"
	import Pfp from '$lib/Pfp.svelte';
	import ImagePicker from '$lib/ImagePicker.svelte';
	import { onMount } from 'svelte';
	let blob: Promise<Blob>;
	let imageSelected = false;
	let message = '';
	let isAnonymous = false;
	let anonymousImage: ArrayBuffer;

	async function fetchImage() {
		// Fetch the image from the static directory
		const response = await fetch('/slack-anoniem.png');
		// Ensure the fetch was successful
		if (!response.ok) {
			throw new Error('Failed to fetch the image');
		}
		// Read the image as an ArrayBuffer
		anonymousImage = await response.arrayBuffer();
	}

	function onImageSelected(event: CustomEvent<{ file: ArrayBuffer }>) {
		isAnonymous = false;
		const file = event?.detail?.file;
		if (!file) return;
		imageSelected = true;
		blob = imglyRemoveBackground(file, {
			publicPath: "https://pfp.toi.vet/assets/",
			progress: onProgress
		});
	}

	function onProgress(key: string) {
		if (key.startsWith('fetch')) {
			message = 'fetching models...';
		} else {
			message = 'removing background...';
		}
	}

	function onMessage(event: CustomEvent<string>) {
		message = event.detail;
	}

	function makeAnonPicture(_: MouseEvent & { currentTarget: EventTarget & HTMLButtonElement }) {
		imageSelected = true;
		isAnonymous = true;
		blob = new Promise((resolve) => {
			resolve(new Blob([anonymousImage]));
		});
	}

	onMount(async () => {
		await fetchImage();
	});
</script>

<main>
	<h1>pfp.toi.vet</h1>
	<p class="disclaimer">
		Select a picture and preview how awesome you look in topicus style! <br/>
		If you'd rather stay anonymous there is also an option for you! <br/><br/>
		Your picture(s) will not be uploaded, all processing
		happens on your device, no data is stored anywhere.
	</p>
	{#if message}
		<p class="message">{message}</p>
	{/if}
	{#if imageSelected}
		{#await blob then blob}
			<Pfp {blob} {isAnonymous} on:message={onMessage} />
		{:catch}
			<p>something went wrong 🙈</p>
		{/await}
	{/if}
	<div class="actions">
		<ImagePicker on:fileSelected={onImageSelected} />
		<button class="btn" on:click={makeAnonPicture}> anonymous </button>
	</div>
</main>

<style>
	h1 {
		font-size: 3rem;
		font-weight: 700;
		margin-bottom: 0.5em;
	}

	.disclaimer {
		max-width: 60ch;
		font-size: 1.2em;
		font-weight: bold;
		margin-bottom: 0.5em;
	}

	main {
		display: flex;
		flex-direction: column;
		align-items: center;
		width: 100%;
	}

	.message {
		font-size: 2em;
		font-weight: bold;
	}

	.actions {
		display: flex;
		flex-direction: row;
		justify-content: center;
		gap: 1em;
		padding: 1em;
		width: 100%;
	}
</style>
