<script>
	import favicon from '$lib/assets/favicon.svg';
	import Header from '$lib/Header.svelte';
	import InteractiveGrid from '$lib/InteractiveGrid.svelte';
	import '../app.css';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

<InteractiveGrid />
<Header />
<main>
	{@render children()}
</main>

<style>
	main {
		position: relative;
		width: 100%;
		z-index: 10;
	}
</style>
