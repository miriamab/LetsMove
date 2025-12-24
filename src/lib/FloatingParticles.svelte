<script>
	import { onMount } from 'svelte';
	import { scrollProgress } from './stores';

	let particles = $state([]);
	let animationFrame;
	const particleCount = 40;
	const cellSize = 14;

	onMount(() => {
		// Initialize particles
		particles = Array.from({ length: particleCount }, (_, i) => ({
			id: i,
			startX: Math.random() * 90 + 5,
			startY: Math.random() * 30 + 65,
			floatOffset: Math.random() * Math.PI * 2,
			floatSpeed: 0.5 + Math.random() * 0.5,
			opacity: 0.5 + Math.random() * 0.5
		}));

		// Animation loop for floating effect
		const animate = () => {
			particles = particles.map(p => ({ ...p })); // Trigger reactivity
			animationFrame = requestAnimationFrame(animate);
		};
		animate();

		// Scroll handler
		const handleScroll = () => {
			const heroEl = document.querySelector('#hero');
			const phase1El = document.querySelector('#phase-1');
			if (!heroEl || !phase1El) return;

			const heroRect = heroEl.getBoundingClientRect();
			const phase1Rect = phase1El.getBoundingClientRect();
			
			// Animate from early in hero scroll to phase-1 entrance
			const scrollStart = window.innerHeight * 0.6;
			const scrollEnd = -window.innerHeight * 0.5;
			
			let progress = (scrollStart - heroRect.bottom) / (scrollStart - scrollEnd);
			progress = Math.max(0, Math.min(1, progress));
			
			scrollProgress.set(progress);
		};

		window.addEventListener('scroll', handleScroll, { passive: true });
		handleScroll();

		return () => {
			window.removeEventListener('scroll', handleScroll);
			cancelAnimationFrame(animationFrame);
		};
	});

	function getStyle(p) {
		const time = Date.now() / 1000;
		const floatY = Math.sin(time * p.floatSpeed + p.floatOffset) * 8;
		const floatX = Math.cos(time * p.floatSpeed * 0.7 + p.floatOffset) * 4;
		
		// Calculate target position in calendar area
		const targetY = 15 + (p.id % 7) * 3;
		const targetX = 10 + ((p.id * 7) % 80);
		
		// Interpolate from start to target based on scroll
		const currentX = p.startX + (targetX - p.startX) * $scrollProgress;
		const currentY = p.startY + (targetY - p.startY) * $scrollProgress;
		
		// Reduce floating as we approach calendar
		const floatAmount = 1 - $scrollProgress;
		
		// Keep particles visible until calendar reaches full opacity, then fade out quickly
		// Calendar reaches full opacity at progress ~0.4 (opacity = progress * 2.5 = 1)
		const opacity = p.opacity * Math.max(0, 1 - Math.max(0, ($scrollProgress - 0.2) * 5));
		
		return `
			left: ${currentX + floatX * floatAmount}%;
			top: ${currentY + floatY * floatAmount}%;
			opacity: ${opacity};
		`;
	}
</script>

<div class="particles-container">
	{#each particles as particle (particle.id)}
		<div
			class="particle"
			style={getStyle(particle)}
		></div>
	{/each}
</div>

<style>
	.particles-container {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100vh;
		pointer-events: none;
		overflow: visible;
		z-index: 10;
	}

	.particle {
		position: absolute;
		width: 14px;
		height: 14px;
		background: #ff8b4c;
		border-radius: 50%;
		transform: translate(-50%, -50%);
		will-change: left, top, opacity;
	}
</style>