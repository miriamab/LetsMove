<script>
	import { onMount } from 'svelte';
	import * as d3 from 'd3';
	import { scrollProgress } from '../stores';

	let container;
	const startYear = new Date().getFullYear() - 5;
	const endYear = new Date().getFullYear();
	const cellSize = 14;
	const cellGapX = 10;
    const cellGapY = 4;

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
				data[date] = 1;
			}
		}
		return data;
	}

	async function loadCsvData() {
		const allDates = {};

		try {
			// Try to fetch from public folder
			const files = [
				{ name: 'Cardio_Daten.csv', path: '/Cardio_Daten.csv' },
				{ name: 'Laufen_Daten.csv', path: '/Laufen_Daten.csv' },
				{ name: 'Fahrrad_Daten.csv', path: '/Fahrrad_Daten.csv' }
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

			const weeksInYear = 54;
			const svgWidth = weeksInYear * (cellSize + cellGapX) - cellGapX + 20;
			const svgHeight = 7 * (cellSize + cellGapY) - cellGapY + 40;

			const svg = yearGroup
				.append('svg')
				.attr('width', '100%')
				.attr('height', svgHeight)
				.attr('viewBox', `0 0 ${svgWidth} ${svgHeight}`)
				.attr('preserveAspectRatio', 'xMinYMid')
				.attr('class', 'calendar-svg');

			// Add year overlay as SVG text (hidden by default, shown on hover)
			const yearLabel = svg
				.append('text')
				.attr('class', 'year-label')
				.attr('x', svgWidth / 2)
				.attr('y', svgHeight / 2)
				.attr('font-size', '72px')
				.attr('font-weight', 'bold')
				.attr('fill', '#ff8b4c')
				.attr('text-anchor', 'middle')
				.attr('dominant-baseline', 'middle')
				.attr('opacity', 0)
				.attr('pointer-events', 'none')
				.text(year);

			// Add hover effect with JavaScript
			yearGroup
				.on('mouseenter', function() {
					// Fade out all circles and month labels
					svg.selectAll('.day').attr('opacity', 0.3);
					svg.selectAll('text:not(.year-label)').attr('opacity', 0.3);
					// Show year label
					yearLabel.attr('opacity', 1);
				})
				.on('mouseleave', function() {
					// Restore circles and month labels to original opacity
					svg.selectAll('.day').attr('opacity', 1);
					svg.selectAll('text:not(.year-label)').attr('opacity', 0.5);
					// Hide year label
					yearLabel.attr('opacity', 0);
				});

			// Month labels: calculate week positions per month upfront
			const monthPositions = [0];
			let weekOffset = 0;
			for (let month = 0; month < 12; month++) {
				const daysInMonth = new Date(year, month + 1, 0).getDate();
				const firstDay = new Date(year, month, 1).getDay();
				const weeksInMonth = Math.ceil((firstDay + daysInMonth) / 7);
				const x = weekOffset * (cellSize + cellGapX) + cellSize ;
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
				weekOffset += weeksInMonth;
				monthPositions.push(weekOffset);
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
		
		// Subscribe to scroll progress
		const unsubscribe = scrollProgress.subscribe(progress => {
			const phase1El = document.querySelector('#phase-1');
			if (!phase1El) return;
			
			// Fade in phase-1 synchronized with particles fading out - use same multiplier
			const opacity = Math.min(1, progress * 2.5);
			phase1El.style.opacity = opacity;
			phase1El.style.pointerEvents = progress < 0.15 ? 'none' : 'auto';
		});
		
		drawCalendar();
		
		return unsubscribe;
	});
</script>

<section class="year-calendar" bind:this={container}></section>

<style>
	.year-calendar {
		padding: 0rem 1.5rem;
	}

	.year-block {
		margin-bottom: 3rem;
	}

	.calendar-svg {
		display: block;
		margin-bottom: 2rem;
		overflow: visible;
	}

	:global(.calendar-svg .day) {
		transition: opacity 0.4s ease;
	}

	:global(.calendar-svg text:not(.year-label)) {
		transition: opacity 0.4s ease;
	}

	:global(.year-label) {
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
			padding: 4rem 0rem;
		}

		.year-block h3 {
			font-size: 2rem;
		}
	}
</style>
