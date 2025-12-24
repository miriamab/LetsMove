<script>
	import Hero from '$lib/Hero.svelte';
	import FloatingParticles from '$lib/FloatingParticles.svelte';
	import YearCalendar from '$lib/visualizations/YearCalendar.svelte';
	import ProjectGrid from '$lib/ProjectGrid.svelte';
	import TextBlocks from '$lib/TextBlocks.svelte';
	import AbstractLines from '$lib/AbstractLines.svelte';
	import Footer from '$lib/Footer.svelte';
</script>

<FloatingParticles />
<section id="hero" class="section">
	<Hero />
</section>

<!-- Phase 1: Duration -->
<section id="phase-1" class="section">
	<div class="phase-header">
		<h2>DURATION</h2>
		<p>A comprehensive breakdown of time invested over 70 months. 
			The Chronologic Grid maps the frequency of effort, while the Duration Analysis 
			quantifies the cumulative hours across all disciplines.</p>
	<YearCalendar
	<ProjectGrid />
</section>

<!-- Phase 2: Distance -->
<section id="phase-2" class="section">
	<div class="phase-header">
		<h2>DISTANCE</h2>
		<p>Where intent meets the local terrain.</p>
	</div>
	<TextBlocks />
</section>

<!-- Phase 3: Endurance -->
<section id="phase-3" class="section">
	<div class="phase-header">
		<h2>ENDURANCE</h2>
		<p>bla</p>
	</div>
	<AbstractLines />
</section>

<!-- Phase 4: Pace -->
<section id="phase-4" class="section">
	<div class="phase-header">
		<h2>PACE</h2>
		<p>bla</p>
	</div>
	<ProjectGrid />
</section>

<!-- Phase 5: Force -->
<section id="phase-5" class="section">
	<div class="phase-header">
		<h2>FORCE</h2>
		<p>blabla</p>
	</div>
	<TextBlocks />
</section>

<Footer />

<style>
	.section {
		min-height: 100vh;
		padding: 40px 0 0 0;
		display: flex;
		flex-direction: column;
	}


	.phase-header {
		padding: 4rem 1.5rem 2rem;
		border-bottom: 1px solid rgba(255, 139, 76, 0.2);
	}

	.phase-header h2 {
		font-size: 3rem;
		font-weight: 700;
		letter-spacing: -1px;
		margin-bottom: 0.5rem;
		color: #ff8b4c;
	}

	.phase-header p {
		font-size: 1rem;
		color: #ff8b4c;
		opacity: 0.7;
		letter-spacing: 0.025em;
		width: 36rem;
	}

	@media (min-width: 768px) {
		.phase-header {
			padding: 0rem 3rem 3rem;
		}

		.phase-header h2 {
			font-size: 4rem;
		}
	}
</style>
