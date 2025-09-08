<script lang="ts">
	import '../app.css';
	import issIcon from '$lib/assets/iss.png';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={issIcon} />
	<title>Internation Space Station Tracker</title>
</svelte:head>

{@render children?.()}
