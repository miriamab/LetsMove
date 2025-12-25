<script>
	import { onMount } from 'svelte';
	import * as d3 from 'd3';

	let container;
	const startYear = new Date().getFullYear() - 5;
	const endYear = new Date().getFullYear();
	const cellSize = 14;
	const cellGapX = 5;
	const cellGapY = 10;

	let chartData = $state([]);

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

	function parseCsvForDistance(text) {
		const lines = text.split('\n').filter(line => line.trim());
		const yearData = {};

		// Initialize years
		for (let year = startYear; year <= endYear; year++) {
			yearData[year] = 0;
		}

		for (let i = 1; i < lines.length; i++) {
			const line = lines[i].trim();
			if (!line) continue;

			const parts = parseCSVLine(line);

			// Column 2 (index 1) is Datum (Date)
			const dateString = parts[1];
			if (!dateString || dateString.length < 10) continue;

			const date = dateString.substring(0, 10);
			const year = parseInt(date.substring(0, 4));

			// Column 5 (index 4) is Distanz (Distance) - format: "3,83" with comma as decimal separator
			let distance = 0;
			if (parts[4]) {
				// Replace German comma with period for parseFloat
				const distanceStr = parts[4].replace(',', '.');
				distance = parseFloat(distanceStr);
				if (isNaN(distance)) distance = 0;
			}

			if (yearData[year] !== undefined && distance > 0) {
				yearData[year] += distance;
			}
		}

		return yearData;
	}

	async function loadDistanceData() {
		try {
			const response = await fetch('/Laufen_Daten.csv');
			if (response.ok) {
				const text = await response.text();
				const yearData = parseCsvForDistance(text);
				
				// Convert to array sorted by year
				const data = Object.entries(yearData).map(([year, km]) => ({
					year: parseInt(year),
					km: km
				})).sort((a, b) => a.year - b.year);

				console.log('Distance data loaded:', data);
				return data;
			}
		} catch (error) {
			console.error('Error loading distance data:', error);
		}
		return [];
	}

	function drawChart() {
		if (!container || chartData.length === 0) return;

		d3.select(container).selectAll('*').remove();

		const margin = { top: 20, right: 40, bottom: 40, left: 60 };
		const maxKm = Math.max(...chartData.map(d => d.km));
		const maxCircles = Math.round((maxKm / maxKm) * 80);
		const calculatedWidth = maxCircles * (cellSize + cellGapX) + 100;
		const width = calculatedWidth;
		const height = 400 - margin.top - margin.bottom;

		const xScale = d3.scaleLinear()
			.domain([0, maxKm * 1.02])
			.range([0, width]);

		const yScale = d3.scaleBand()
			.domain(chartData.map(d => d.year))
			.range([0, height])
			.padding(0.3);

		const svg = d3.select(container)
			.append('svg')
			.attr('width', '100%')
			.attr('height', height + margin.top + margin.bottom)
			.attr('viewBox', `0 0 ${width + margin.left + margin.right} ${height + margin.top + margin.bottom}`)
			.attr('preserveAspectRatio', 'xMinYMid')
			.attr('class', 'distance-chart');

		const g = svg.append('g')
			.attr('transform', `translate(${margin.left},${margin.top})`);

		// Y-axis
		const yAxis = g.append('g')
			.call(d3.axisLeft(yScale))
			.style('color', '#ff8b4c')
			.style('font-size', '14px');
		yAxis.selectAll('text')
			.style('fill', '#ff8b4c');
		yAxis.selectAll('line')
			.style('display', 'none');

		// X-axis
		const xAxis = g.append('g')
			.attr('transform', `translate(0,${height})`)
			.call(d3.axisBottom(xScale).ticks(5))
			.style('color', '#ff8b4c')
			.style('font-size', '12px');
		xAxis.selectAll('text')
			.style('fill', '#ff8b4c');
		xAxis.selectAll('line')
			.style('display', 'none');

		// Draw circles as bars
		chartData.forEach((d) => {
			const y = yScale(d.year);
			const numCircles = Math.round((d.km / maxKm) * 80);
			const kmPerCircle = maxKm / 80;

			for (let i = 0; i < numCircles; i++) {
				const kmPosition = (i + 0.5) * kmPerCircle;
				const x = xScale(kmPosition);

				g.append('circle')
					.attr('cx', x)
					.attr('cy', y + yScale.bandwidth() / 2)
					.attr('r', cellSize / 2)
					.attr('fill', '#ff8b4c')
					.attr('opacity', 1)
					.on('mouseenter', function () {
						d3.select(this)
							.attr('opacity', 1)
							.attr('r', cellSize / 2 + 1);
					})
					.on('mouseleave', function () {
						d3.select(this)
							.attr('opacity', 1)
							.attr('r', cellSize / 2);
					});
			}
		});

		// X-axis label
		g.append('text')
			.attr('x', width+100)
			.attr('y', height + 3)
			.attr('text-anchor', 'end')
			.attr('fill', '#ff8b4c')
			.style('font-size', '12px')
			.text('Kilometers');

		// Y-axis label
		g.append('text')
			.attr('x', -13)
			.attr('y', 0 - 12)
			.attr('text-anchor', 'start')
			.attr('fill', '#ff8b4c')
			.style('font-size', '12px')
			.text('Year');
	}

	onMount(async () => {
		chartData = await loadDistanceData();
		drawChart();
	});
</script>

<div class="distance-chart-container" bind:this={container}></div>

<style>
	.distance-chart-container {
		display: flex;
		justify-content: center;
	}

	:global(.distance-chart text) {
		font-family: inherit;
	}

	:global(.distance-chart .domain) {
		stroke: #ff8b4c;
		opacity: 0.2;
	}

	@media (min-width: 768px) {
		.distance-chart-container {
			padding: 2rem 3rem;
		}
	}
</style>
