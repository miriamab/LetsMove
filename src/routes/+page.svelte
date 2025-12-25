<script>
	import Hero from '$lib/Hero.svelte';
	import FloatingParticles from '$lib/FloatingParticles.svelte';
	import YearCalendar from '$lib/visualizations/YearCalendar.svelte';
	import DistanceChart from '$lib/visualizations/DistanceChart.svelte';
	import WeekCalendar from '$lib/visualizations/WeekCalendar.svelte';
	import PaceMap from '$lib/visualizations/PaceMap.svelte';

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
		<p>Each orange dot represents a day with tracked athletic activity.</p>
	<YearCalendar
	<ProjectGrid />
</section>

<!-- Phase 2: Distance -->
<section id="phase-2" class="section">
	<div class="phase-header">
		<h2>DISTANCE</h2>
		<p>Each dot represents a fixed distance, marking the extreme activity peaks during the lockdown years.</p>
	</div>
	<DistanceChart />
</section>

<!-- Phase 3: Endurance -->
<section id="phase-3" class="section">
	<div class="phase-header">
		<h2>ENDURANCE</h2>
		<p>A calendar where each entry offers a closer look at a single week.</p>
	</div>
	<WeekCalendar />
</section>

<!-- Phase 4: Pace -->
<section id="phase-4" class="section">
	<div class="phase-header">
		<h2>PACE</h2>
		<p>A closer look into a single session where we can see path, pace, and heart rate.
		</p>
	</div>
	<PaceMap />
</section>

<!-- Phase 5: Force -->
<section id="phase-5" class="section">
	<div class="phase-header">
		<h2>FORCE</h2>
		<p>blabla</p>
	</div>
</section>

<Footer />

<style>
	.section {
		min-height: 10vh;
		padding-top: 10rem;
		display: flex;
		flex-direction: column;
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
			padding: 0 3rem 1.5rem;
		}

		.phase-header h2 {
			font-size: 4rem;
		}
	}
</style>
