<script>
	import { onMount } from 'svelte';

	let activeSection = $state('hero');

	function updateActiveSection() {
		const sections = ['hero', 'phase-1', 'phase-2', 'phase-3', 'phase-4', 'phase-5'];
		const scrollPosition = window.scrollY;
		const windowHeight = window.innerHeight;
		const threshold = windowHeight * 0.5; // 50% of viewport height

		// Find which section is most visible
		let currentSection = 'hero';
		
		for (const sectionId of sections) {
			const element = document.getElementById(sectionId);
			if (element) {
				const rect = element.getBoundingClientRect();
				const elementTop = rect.top + scrollPosition;
				const elementBottom = elementTop + rect.height;
				
				// Check if section is currently visible and above the threshold
				if (scrollPosition + threshold >= elementTop && scrollPosition + threshold < elementBottom) {
					currentSection = sectionId;
				}
			}
		}
		
		activeSection = currentSection;
	}

	onMount(() => {
		window.addEventListener('scroll', updateActiveSection);
		updateActiveSection(); // Check initial position

		return () => {
			window.removeEventListener('scroll', updateActiveSection);
		};
	});
</script>

<header>
	<div class="header-gradient">
		<nav>
			<a href="#hero" class="logo line-decoration" class:active={activeSection === 'hero'}>Miriam Abbas</a>
			<div class="nav-links">
				<a href="#phase-1" class="line-decoration" class:active={activeSection === 'phase-1'}>Duration</a>
				<a href="#phase-2" class="line-decoration" class:active={activeSection === 'phase-2'}>Distance</a>
				<a href="#phase-3" class="line-decoration" class:active={activeSection === 'phase-3'}>Endurance</a>
				<a href="#phase-4" class="line-decoration" class:active={activeSection === 'phase-4'}>Pace</a>
				<a href="#phase-5" class="line-decoration" class:active={activeSection === 'phase-5'}>Force</a>
			</div>
		</nav>
	</div>
</header>

<style>
	header {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		z-index: 50;
	}

	.header-gradient {
		background: linear-gradient(to bottom, 
			rgba(15, 15, 15, 0.98) 0%, 
			rgba(15, 15, 15, 0.95) 20%,
			rgba(15, 15, 15, 0.85) 40%,
			rgba(15, 15, 15, 0.6) 60%,
			rgba(15, 15, 15, 0.3) 85%,
			rgba(15, 15, 15, 0) 100%);
		backdrop-filter: blur(0px);
		-webkit-backdrop-filter: blur(10px);
		padding: 1rem 1.5rem 3rem;
	}

	nav {
		display: flex;
		align-items: center;
		justify-content: space-between;
	}

	.logo {
		font-size: 0.875rem;
		letter-spacing: 0.1em;
		text-transform: uppercase;
		font-weight: 500;
		text-decoration: none;
		color: var(--foreground);
		opacity: 1;
		transition: opacity 0.3s ease;
	}

	.logo:hover {
		opacity: 0.8;
	}

	.nav-links {
		display: flex;
		align-items: center;
		gap: 1rem;
		font-size: 0.75rem;
		letter-spacing: 0.05em;
		text-transform: uppercase;
	}

	.nav-links a {
		opacity: 0.7;
		transition: opacity 0.3s ease;
		text-decoration: none;
		color: var(--foreground);
	}

	.nav-links a:hover,
	.nav-links a.active {
		opacity: 1;
	}

	.logo.active {
		opacity: 1;
	}

	@media (min-width: 768px) {
		.header-gradient {
			padding: 1.5rem 3rem 1.5rem;
		}

		.logo {
			font-size: 1rem;
		}

		.nav-links {
			gap: 1.5rem;
			font-size: 0.875rem;
		}
	}
</style>
