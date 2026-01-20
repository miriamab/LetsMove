<script>
	import { onMount } from 'svelte';
	import { base } from '$app/paths';
	import * as d3 from 'd3';

	let container;
	const startYear = 2020;
	const endYear = 2025;
	const cellSize = 14;
	const cellGapX = 10;
    const cellGapY = 4;

	const yearTexts = {
		2020: "Pandemic lockdowns provided a significant increase in available time, leading to a massive surge in overall training volume.",
		2021: "Continued lockdown combined with focused training for a half-marathon.",
		2022: "A transition year where I discovered basketball, causing a noticeable decline in recorded running sessions.",
		2023: "Basketball twice a week; however, these hours remain unrecorded in the digital dataset.",
		2024: "The end of my basketball era overlaping with the start of University, leading to more irregular training patterns.",
		2025: "New gym membership since October and returning to more running."
	};

	let calendarData = $state({});

	function parseCsv(text) {
		const lines = text.split('\n').filter(line => line.trim());
		const data = {};

		for (let i = 1; i < lines.length; i++) {
			const line = lines[i];
			// Split by comma, but be careful with quoted fields
			let dateString = '';
			const firstQuote = line.indexOf('"');
			const secondQuote = line.indexOf('"', firstQuote + 1);
			
			if (firstQuote !== -1 && secondQuote !== -1) {
				// There are quotes, extract the date more carefully
				const beforeFirst = line.substring(0, firstQuote);
				const parts = beforeFirst.split(',');
				dateString = parts[1]?.trim();
			} else {
				// No quotes or simple case
				const parts = line.split(',');
				dateString = parts[1]?.trim();
			}

			if (dateString && dateString.length >= 10) {
				const date = dateString.substring(0, 10); // Get YYYY-MM-DD
				const year = parseInt(date.substring(0, 4));
				if (year >= 2020 && year <= 2025) {
					data[date] = 1;
				}
			}
		}
		return data;
	}

	async function loadCsvData() {
		const allDates = {};

		try {
			// Try to fetch from public folder
			const files = [
				   { name: 'Cardio & Strength Training', path: `${base}/Cardio_Daten.csv` },
				{ name: 'Laufen_Daten.csv', path: `${base}/Laufen_Daten.csv` },
				{ name: 'Fahrrad_Daten.csv', path: `${base}/Fahrrad_Daten.csv` }
			];

			for (const file of files) {
				try {
					const response = await fetch(file.path);
					if (response.ok) {
						const text = await response.text();
						const dates = parseCsv(text);
						Object.assign(allDates, dates);
						console.log(`Loaded ${file.name}: ${Object.keys(dates).length} unique dates`);
					} else {
						console.warn(`Failed to load ${file.name}: ${response.status}`);
					}
				} catch (e) {
					console.warn(`Error loading ${file.name}:`, e);
				}
			}
		} catch (error) {
			console.error('Error in loadCsvData:', error);
		}

		console.log(`Total unique dates with activity: ${Object.keys(allDates).length}`);
		return allDates;
	}

	function getColorIntensity(value) {
		if (value === 0) return 'rgba(100, 100, 100, 0.2)'; // Dunkelgrau
		return '#ff8b4c'; // Vollständiges Orange
	}

	function drawCalendar() {
		if (!container) return;

		d3.select(container).selectAll('*').remove();

		const monthLabels = [
			'JAN',
			'FEB',
			'MAR',
			'APR',
			'MAY',
			'JUN',
			'JUL',
			'AUG',
			'SEP',
			'OCT',
			'NOV',
			'DEC'
		];

		for (let year = startYear; year <= endYear; year++) {
			const yearGroup = d3
				.select(container)
				.append('div')
				.attr('class', 'year-block');

			// Calculate total weeks first to determine SVG width
			let totalWeeks = 0;
			const monthPositions = [0];
			
			for (let month = 0; month < 12; month++) {
				const daysInMonth = new Date(year, month + 1, 0).getDate();
				const firstDay = new Date(year, month, 1).getDay();
				const weeksInMonth = Math.ceil((firstDay + daysInMonth) / 7);
				totalWeeks += weeksInMonth;
				monthPositions.push(totalWeeks);
			}

			const svgWidth = totalWeeks * (cellSize + cellGapX) - cellGapX + 20;
			const svgHeight = 7 * (cellSize + cellGapY) - cellGapY + 40;

			const svg = yearGroup
				.append('svg')
				.attr('width', '100%')
				.attr('height', svgHeight)
				.attr('viewBox', `0 0 ${svgWidth} ${svgHeight}`)
				.attr('preserveAspectRatio', 'xMidYMid meet')
				.attr('class', 'calendar-svg');

			// Add year overlay as SVG text (hidden by default, shown on hover)
			const yearLabel = svg
				.append('text')
				.attr('class', 'year-label overlay-text')
				.attr('x', svgWidth * 0.25)
				.attr('y', svgHeight / 2)
				.attr('font-size', '72px')
				.attr('font-weight', 'bold')
				.attr('fill', '#ff8b4c')
				.attr('text-anchor', 'middle')
				.attr('dominant-baseline', 'middle')
				.attr('opacity', 0)
				.attr('pointer-events', 'none')
				.text(year);

			const infoLabel = svg
				.append('foreignObject')
				.attr('class', 'info-label overlay-text')
				.attr('x', svgWidth * 0.5)
				.attr('y', svgHeight / 2 - 40)
				.attr('width', svgWidth * 0.45)
				.attr('height', 80)
				.attr('opacity', 0)
				.style('pointer-events', 'none');

			infoLabel
				.append('xhtml:div')
				.style('font-size', '16px')
				.style('color', '#ff8b4c')
				.style('line-height', '1.4')
				.style('height', '100%')
				.style('display', 'flex')
				.style('align-items', 'center')
				.text(yearTexts[year] || "Keine Beschreibung verfügbar.");

			// Add hover effect with JavaScript
			yearGroup
				.on('mouseenter', function() {
					// Fade out all circles and month labels
					svg.selectAll('.day').attr('opacity', 0.1);
					svg.selectAll('text:not(.overlay-text)').attr('opacity', 0.1);
					// Show overlays
					yearLabel.attr('opacity', 1);
					infoLabel.attr('opacity', 1);
				})
				.on('mouseleave', function() {
					// Restore circles and month labels to original opacity
					svg.selectAll('.day').attr('opacity', 1);
					svg.selectAll('text:not(.overlay-text)').attr('opacity', 0.5);
					// Hide overlays
					yearLabel.attr('opacity', 0);
					infoLabel.attr('opacity', 0);
				});

			// Month labels
			for (let month = 0; month < 12; month++) {
				const x = monthPositions[month] * (cellSize + cellGapX) + cellSize ;
				svg
					.append('text')
					.attr('x', x)
				    .attr('y', 15)
					.attr('font-size', '15px')
					.attr('fill', '#ff8b4c')
					.attr('font-weight', 'bold')
					.attr('text-anchor', 'middle')				
                    .attr('opacity', 0.5)					
                    .text(monthLabels[month]);
			}

			// Draw all days of the year
			for (let month = 0; month < 12; month++) {
				const daysInMonth = new Date(year, month + 1, 0).getDate();
				const firstDay = new Date(year, month, 1).getDay();
				const monthWeekOffset = monthPositions[month];

				// Placeholder circles (days before month starts)
				for (let i = 0; i < firstDay; i++) {
					const cellIdx = i;
					const week = monthWeekOffset + Math.floor(cellIdx / 7);
					const dayOfWeek = cellIdx % 7;
					const x = week * (cellSize + cellGapX);
					const y = dayOfWeek * (cellSize + cellGapY) + 20;

					svg
						.append('circle')
						.attr('cx', x + cellSize / 2)
						.attr('cy', y + cellSize / 2)
						.attr('r', cellSize / 2)
						.attr('fill', 'rgba(100, 100, 100, 0.2)')
						.attr('stroke', 'rgba(100, 100, 100, 0.15)')
						.attr('stroke-width', 0.5);
				}

				// Actual days
				for (let day = 1; day <= daysInMonth; day++) {
					const cellIdx = firstDay + day - 1;
					const week = monthWeekOffset + Math.floor(cellIdx / 7);
					const dayOfWeek = cellIdx % 7;
					const x = week * (cellSize + cellGapX);
					const y = dayOfWeek * (cellSize + cellGapY) + 20;

					const dateStr = `${year}-${String(month + 1).padStart(2, '0')}-${String(day).padStart(2, '0')}`;
					const value = calendarData[dateStr] ? 1 : 0;

					svg
						.append('circle')
						.attr('cx', x + cellSize / 2)
						.attr('cy', y + cellSize / 2)
						.attr('r', cellSize / 2)
						.attr('fill', getColorIntensity(value))
						.attr('stroke', value === 1 ? '#ff8b4c' : 'rgba(100, 100, 100, 0.15)')
						.attr('stroke-width', 0.5)
						.attr('class', 'day')
						.attr('title', dateStr)
						.on('mouseenter', function () {
							d3.select(this)
								.attr('stroke', '#ff8b4c')
								.attr('stroke-width', 1.5)
								.style('filter', 'drop-shadow(0 0 4px rgba(255, 139, 76, 0.5))');
						})
						.on('mouseleave', function () {
							d3.select(this)
								.attr('stroke', value === 1 ? '#ff8b4c' : 'rgba(100, 100, 100, 0.15)')
								.attr('stroke-width', 0.5)
								.style('filter', 'none');
						});
				}
			}
		}
	}

	onMount(async () => {
		calendarData = await loadCsvData();
		drawCalendar();
	});
</script>

<section class="year-calendar" bind:this={container}></section>

<style>
	.year-calendar {
		padding: 0rem 1.5rem;
		width: 100%;
		box-sizing: border-box;
		overflow-x: auto;
	}

	.year-block {
		margin-bottom: 2rem;
		display: flex;
		justify-content: center;
		align-items: center;
		width: 100%;
		min-width: 0;
	}

	.calendar-svg {
		max-width: 100%;
		height: auto;
		display: block;
		margin-bottom: 2rem;
		overflow: visible;
		flex-shrink: 0;
	}

	:global(.calendar-svg .day) {
		transition: opacity 0.4s ease;
	}

	:global(.calendar-svg text:not(.overlay-text)) {
		transition: opacity 0.4s ease;
	}

	:global(.overlay-text) {
		transition: opacity 0.4s ease;
	}

	.legend {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 0.75rem;
		margin-top: 3rem;
		padding-top: 2rem;
		border-top: 1px solid rgba(255, 139, 76, 0.2);
		flex-wrap: wrap;
	}

	.legend-label {
		font-size: 0.75rem;
		color: #ff8b4c;
		opacity: 0.6;
		letter-spacing: 0.05em;
		text-transform: uppercase;
	}

	.legend-item {
		width: 16px;
		height: 16px;
		border: 1px solid rgba(255, 139, 76, 0.2);
	}

	@media (min-width: 768px) {
		.year-calendar {
			padding: 4rem 5rem;
		}

		.year-block h3 {
			font-size: 2rem;
		}
	}
</style>
