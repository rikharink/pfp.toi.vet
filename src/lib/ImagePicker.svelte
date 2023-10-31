<script lang="ts">
	import { createEventDispatcher } from 'svelte';
	const dispatch = createEventDispatcher();
	let selectedFiles: FileList | null = null;

	function handleFiles() {
		if (!selectedFiles || selectedFiles.length === 0) return;
		const file = selectedFiles[0];
		const reader = new FileReader();
		reader.addEventListener('load', async () => {
			dispatch('fileSelected', { file: reader.result as ArrayBuffer });
		});
		reader.readAsArrayBuffer(file);
	}
</script>

<div class="input">
	<label class="btn" for="file">select picture</label>
	<input
		id="file"
		type="file"
		accept="image/*"
		bind:files={selectedFiles}
		on:change={handleFiles}
	/>
</div>

<style>
	#file {
		display: none;
	}

	label {
		display: flex;
		align-items: center;
	}
</style>
