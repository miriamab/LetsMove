<script>
	import { onMount, onDestroy } from 'svelte';
	import { base } from '$app/paths';
	import * as d3 from 'd3';

	let bubbleData = $state([]);
	let loading = $state(true);
	let animationFrame;
	let time = 0;

	const activityColors = {
		'Laufen': '#ff8b4c',
		'Laufband': '#ff8b4c',
		'Fahrrad': '#ff8b4c',
		'Indoor Cycling': '#ff8b4c',
		'Cardio & Strength Training': '#ff8b4c'
	};

	function timeToMinutes(timeStr) {
		if (!timeStr) return 0;
		const parts = timeStr.split(':');
		const hours = parseInt(parts[0]) || 0;
		const minutes = parseInt(parts[1]) || 0;
		const seconds = parseInt(parts[2]) || 0;
		return hours * 60 + minutes + seconds / 60;
	}

	function parseCSVLine(line) {
		const result = [];
		let current = '';
		let insideQuotes = false;

		for (let i = 0; i < line.length; i++) {
			const char = line[i];
			if (char === '"') {
				insideQuotes = !insideQuotes;
			} else if (char === ',' && !insideQuotes) {
				result.push(current.trim());
				current = '';
			} else {
				current += char;
			}
		}
		result.push(current.trim());
		return result;
	}

	async function loadAllData() {
		try {
			const [laufenRes, cardioRes, fahrradRes] = await Promise.all([
				fetch(`${base}/Laufen_Daten.csv`),
				fetch(`${base}/Cardio_Daten.csv`),
				fetch(`${base}/Fahrrad_Daten.csv`)
			]);

			const [laufenText, cardioText, fahrradText] = await Promise.all([
				laufenRes.text(),
				cardioRes.text(),
				fahrradRes.text()
			]);

			const activityTotals = {
				'Laufen': 0,
				'Laufband': 0,
				'Fahrrad': 0,
				'Indoor Cycling': 0,
				   'Cardio & Strength Training': 0
			};

			// Parse Laufen - check activity type column
			laufenText.split('\n').slice(1).forEach(line => {
				if (!line.trim()) return;
				const cols = parseCSVLine(line);
				const dateStr = cols[1];
				if (!dateStr) return;
				const year = parseInt(dateStr.substring(0, 4));
				if (year < 2020 || year > 2025) return;

				const activityType = cols[0] || '';
				const timeStr = cols[6];
				const minutes = timeToMinutes(timeStr);

				if (activityType.toLowerCase().includes('laufband') || activityType.toLowerCase().includes('treadmill')) {
					activityTotals['Laufband'] += minutes;
				} else {
					activityTotals['Laufen'] += minutes;
				}
			});

			       // Parse Cardio
			       cardioText.split('\n').slice(1).forEach(line => {
				       if (!line.trim()) return;
				       const cols = parseCSVLine(line);
				       const dateStr = cols[1];
				       if (!dateStr) return;
				       const year = parseInt(dateStr.substring(0, 4));
				       if (year < 2020 || year > 2025) return;

				       const timeStr = cols[6];
				       const minutes = timeToMinutes(timeStr);
				       activityTotals['Cardio & Strength Training'] += minutes;
			       });

			// Parse Fahrrad - check for indoor/outdoor
			fahrradText.split('\n').slice(1).forEach(line => {
				if (!line.trim()) return;
				const cols = parseCSVLine(line);
				const dateStr = cols[1];
				if (!dateStr) return;
				const year = parseInt(dateStr.substring(0, 4));
				if (year < 2020 || year > 2025) return;

				const activityType = cols[0] || '';
				const timeStr = cols[6];
				const minutes = timeToMinutes(timeStr);

				if (activityType.toLowerCase().includes('indoor') || activityType.toLowerCase().includes('spinning')) {
					activityTotals['Indoor Cycling'] += minutes;
				} else {
					activityTotals['Fahrrad'] += minutes;
				}
			});

			// Convert to hours and create bubble data
			const data = Object.entries(activityTotals)
				.filter(([_, minutes]) => minutes > 0)
				.map(([name, minutes]) => ({
					name,
					hours: Math.round(minutes / 60),
					color: activityColors[name],
					// Random initial positions
					x: 400 + (Math.random() - 0.5) * 200,
					y: 250 + (Math.random() - 0.5) * 150,
					// For organic animation
					phase: Math.random() * Math.PI * 2,
					speed: 0.5 + Math.random() * 0.5
				}));

			// Calculate radius based on hours (area proportional to hours)
			const maxHours = Math.max(...data.map(d => d.hours));
			data.forEach(d => {
				d.radius = Math.sqrt(d.hours / maxHours) * 120 + 40;
			});

			bubbleData = data;
			loading = false;

			// Start D3 force simulation
			startSimulation();
		} catch (error) {
			console.error('Error loading data:', error);
			loading = false;
		}
	}

	function startSimulation() {
		// Use D3 Treemap for hierarchical layout
		const treemapLayout = d3.treemap()
			.size([1600, 800])
			.padding(0);

		// Create hierarchy
		const root = d3.hierarchy({ children: bubbleData })
			.sum(d => d.hours);

		const tree = treemapLayout(root);

		// Update bubble positions and sizes from treemap
		const leaves = tree.leaves();
		leaves.forEach((node, i) => {
			if (bubbleData[i]) {
				bubbleData[i].x = node.x0 + (node.x1 - node.x0) / 2;
				bubbleData[i].y = node.y0 + (node.y1 - node.y0) / 2 + 50;
				// Calculate radius from area (proportional to hours)
				const width = node.x1 - node.x0;
				const height = node.y1 - node.y0;
				const area = width * height;
				// radius from area: r = sqrt(area / PI), but scale down to fit nicely
				bubbleData[i].radius = Math.sqrt(area / Math.PI) * 0.85;
			}
		});

		bubbleData = [...bubbleData];
	}

	function generateBlobPath(cx, cy, radius, segments, time, phase, speed) {
		const points = [];
		for (let i = 0; i < segments; i++) {
			const angle = (i / segments) * Math.PI * 2;
			       // Organic wobble using multiple sine waves (dynamischer, aber nicht zu stark)
			       const wobble = 
				       Math.sin(angle * 3 + phase) * 4 * Math.sin(time * speed * 1.2 + phase) +
				       Math.sin(angle * 5 + phase * 2) * 2.7 * Math.sin(time * speed * 1.7) +
				       Math.sin(angle * 2 + phase * 0.5) * 2.5 * Math.sin(time * speed * 0.9) +
				       Math.sin(angle * 7 + phase * 1.1) * 1.2 * Math.sin(time * speed * 2.3);
			
			const r = radius + wobble;
			const x = cx + Math.cos(angle) * r;
			const y = cy + Math.sin(angle) * r;
			points.push({ x, y });
		}
		return points;
	}

	function pointsToPath(points) {
		if (points.length < 3) return '';
		
		// Create smooth curve through points using cubic bezier
		let path = `M ${points[0].x} ${points[0].y}`;
		
		for (let i = 0; i < points.length; i++) {
			const p0 = points[(i - 1 + points.length) % points.length];
			const p1 = points[i];
			const p2 = points[(i + 1) % points.length];
			const p3 = points[(i + 2) % points.length];
			
			// Catmull-Rom to Bezier conversion
			const cp1x = p1.x + (p2.x - p0.x) / 6;
			const cp1y = p1.y + (p2.y - p0.y) / 6;
			const cp2x = p2.x - (p3.x - p1.x) / 6;
			const cp2y = p2.y - (p3.y - p1.y) / 6;
			
			path += ` C ${cp1x} ${cp1y}, ${cp2x} ${cp2y}, ${p2.x} ${p2.y}`;
		}
		
		return path + ' Z';
	}

	function animate() {
		time += 0.02;
		bubbleData = bubbleData.map(b => ({ ...b })); // Trigger reactivity
		animationFrame = requestAnimationFrame(animate);
	}

	onMount(async () => {
		await loadAllData();
		animate();
	});

	onDestroy(() => {
		if (animationFrame) {
			cancelAnimationFrame(animationFrame);
		}
	});
</script>

<div class="activity-bubbles-container">
	{#if loading}
		<p>Loading activity data...</p>
	{:else}
		<svg viewBox="0 0 1600 900" class="bubbles-svg">
			<defs>
				{#each bubbleData as bubble, i}
					<filter id="glow-{i}" x="-50%" y="-50%" width="200%" height="200%">
						<feGaussianBlur stdDeviation="8" result="coloredBlur"/>
						<feMerge>
							<feMergeNode in="coloredBlur"/>
							<feMergeNode in="SourceGraphic"/>
						</feMerge>
					</filter>
				{/each}
			</defs>
			
			{#each bubbleData as bubble, i}
				{@const points = generateBlobPath(bubble.x, bubble.y, bubble.radius, 32, time, bubble.phase, bubble.speed)}
				{@const path = pointsToPath(points)}
				
				<g class="bubble-group">
					<!-- Glow effect -->
					<path
						d={path}
						fill={bubble.color}
						opacity="0.15"
						filter="url(#glow-{i})"
					/>
					
					<!-- Main blob -->
					<path
						d={path}
						fill={bubble.color}
						opacity="0.80"
						class="blob"
					/>
					
					<!-- Label -->
					<text
						x={bubble.x}
						y={bubble.y - 10}
						text-anchor="middle"
						class="bubble-label"
					>
						{bubble.name}
					</text>
					<text
						x={bubble.x}
						y={bubble.y + 15}
						text-anchor="middle"
						class="bubble-hours"
					>
						{bubble.hours}h
					</text>
				</g>
			{/each}
		</svg>
		
		<div class="legend">
			{#each bubbleData as bubble}
				<div class="legend-item">
					<span class="legend-color" style="background-color: {bubble.color}"></span>
					<span class="legend-text">{bubble.name}: {bubble.hours} hours</span>
				</div>
			{/each}
		</div>
	{/if}
</div>

<style>
	.activity-bubbles-container {
		padding: 3rem 1.5rem;
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.bubbles-svg {
		width: 100%;
		max-width: 1400px;
		height: auto;
	}

	.blob {
		transition: opacity 0.3s ease;
	}

	.bubble-group:hover .blob {
		opacity: 1;
	}

	.bubble-label {
		fill: #0f0f0f;
		font-size: 14px;
		font-weight: 600;
		pointer-events: none;
	}

	.bubble-hours {
		fill: #0f0f0f;
		font-size: 20px;
		font-weight: 700;
		pointer-events: none;
	}

	.legend {
		display: flex;
		flex-wrap: wrap;
		gap: 1.5rem;
		margin-top: 2rem;
		justify-content: center;
	}

	.legend-item {
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.legend-color {
		width: 12px;
		height: 12px;
		border-radius: 50%;
	}

	.legend-text {
		font-size: 0.85rem;
		color: #ff8b4c;
		opacity: 0.8;
	}

	@media (min-width: 768px) {
		.activity-bubbles-container {
			padding: 3rem 3rem;
		}

		.bubble-label {
			font-size: 16px;
		}

		.bubble-hours {
			font-size: 24px;
		}
	}
</style>
