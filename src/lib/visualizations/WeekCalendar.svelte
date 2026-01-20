<script>
	import { onMount } from 'svelte';
	import { base } from '$app/paths';

	let weeks = $state({ peak: null, low: null, average: null });
	let selectedWeek = $state('peak');
	let dropdownOpen = $state(false);
	let currentWeek = $state(null);
	let loading = $state(true);

	const dayNames = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'];
	const sportColors = {
		Running: '#ff8b4c',
		'Cardio & Strength Training': '#ff8b4c',
		'Indoor Cycling': '#ff8b4c',
		'Outdoor Cycling': '#ff8b4c',
		Cycling: '#ff8b4c'
	};

	function timeToMinutes(timeStr) {
		if (!timeStr) return 0;
		const parts = timeStr.split(':');
		const hours = parseInt(parts[0]) || 0;
		const minutes = parseInt(parts[1]) || 0;
		const seconds = parseInt(parts[2]) || 0;
		return hours * 60 + minutes + seconds / 60;
	}

	function minutesToHours(minutes) {
		const hours = Math.floor(minutes / 60);
		const mins = Math.round(minutes % 60);
		return `${String(hours).padStart(2, '0')}:${String(mins).padStart(2, '0')}`;
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

			// Parse all activities by week
			const weeklyData = {};

			// Parse Laufen
			laufenText.split('\n').slice(1).forEach(line => {
				if (!line.trim()) return;
				const cols = parseCSVLine(line);
				const dateStr = cols[1];
				const timeStr = cols[6];
				if (!dateStr) return;

				const date = new Date(dateStr);
				const year = date.getFullYear();
				if (year < 2020 || year > 2025) return;

				const weekKey = getWeekKey(date);
				const dayKey = date.getDay() === 0 ? 6 : date.getDay() - 1; // Convert to Mon-Sun

				if (!weeklyData[weekKey]) weeklyData[weekKey] = {};
				if (!weeklyData[weekKey][dayKey]) weeklyData[weekKey][dayKey] = [];

				weeklyData[weekKey][dayKey].push({
					type: 'Running',
					duration: timeToMinutes(timeStr)
				});
			});

			       // Parse Cardio
			       cardioText.split('\n').slice(1).forEach(line => {
				       if (!line.trim()) return;
				       const cols = parseCSVLine(line);
				       const dateStr = cols[1];
				       const timeStr = cols[6];
				       if (!dateStr) return;

				       const date = new Date(dateStr);
				       const year = date.getFullYear();
				       if (year < 2020 || year > 2025) return;

				       const weekKey = getWeekKey(date);
				       const dayKey = date.getDay() === 0 ? 6 : date.getDay() - 1;

				       if (!weeklyData[weekKey]) weeklyData[weekKey] = {};
				       if (!weeklyData[weekKey][dayKey]) weeklyData[weekKey][dayKey] = [];

				       weeklyData[weekKey][dayKey].push({
					       type: 'Cardio & Strength Training',
					       duration: timeToMinutes(timeStr)
				       });
			       });

			// Parse Fahrrad
			fahrradText.split('\n').slice(1).forEach(line => {
				if (!line.trim()) return;
				const cols = parseCSVLine(line);
				const dateStr = cols[1];
				const timeStr = cols[6];
				if (!dateStr) return;

				const date = new Date(dateStr);
				const year = date.getFullYear();
				if (year < 2020 || year > 2025) return;

				const weekKey = getWeekKey(date);
				const dayKey = date.getDay() === 0 ? 6 : date.getDay() - 1;

				if (!weeklyData[weekKey]) weeklyData[weekKey] = {};
				if (!weeklyData[weekKey][dayKey]) weeklyData[weekKey][dayKey] = [];

				const actType = cols[0];
				weeklyData[weekKey][dayKey].push({
					type: actType.includes('Laufband') ? 'Cycling' : 'Cycling',
					duration: timeToMinutes(timeStr)
				});
			});

			// Calculate week stats
			const weekStats = Object.entries(weeklyData).map(([key, days]) => {
				let totalMinutes = 0;
				let sessionCount = 0;

				Object.values(days).forEach(dayActivities => {
					dayActivities.forEach(activity => {
						totalMinutes += activity.duration;
						sessionCount++;
					});
				});

				return {
					key,
					totalMinutes,
					sessionCount,
					days: formatWeekData(days)
				};
			});

// Find peak (most hours), low (2 sessions from 2022), average (5 sessions from 2024)
		const peakWeek = weekStats.reduce((max, w) => (w.totalMinutes > max.totalMinutes ? w : max));
		
		// Low: 2 sessions from 2022
		const lowWeeks = weekStats.filter(w => {
			const year = new Date(w.key).getFullYear();
			return w.sessionCount === 2 && year === 2022;
		});
		const lowWeek = lowWeeks.length > 0 
			? lowWeeks[0]
			: weekStats.find(w => w.sessionCount === 2) || peakWeek;
		
		// Average: 5 sessions from 2024
		const avgWeeks = weekStats.filter(w => {
			const year = new Date(w.key).getFullYear();
			return w.sessionCount === 5 && year === 2024;
		});
		const averageWeek = avgWeeks.length > 0 
			? avgWeeks[0]
			: weekStats.find(w => w.sessionCount === 5) || peakWeek;

			weeks = {
				peak: peakWeek,
				low: lowWeek || peakWeek,
				average: averageWeek || peakWeek
			};

			currentWeek = weeks.peak;
			loading = false;

			console.log('Week data loaded:', weeks);
		} catch (error) {
			console.error('Error loading week data:', error);
			loading = false;
		}
	}

	function getWeekKey(date) {
		const d = new Date(date);
		d.setHours(0, 0, 0, 0);
		d.setDate(d.getDate() - (d.getDay() || 7) + 1); // Start of week (Monday)
		return d.toISOString().split('T')[0];
	}

	function formatWeekData(daysObj) {
		const formatted = Array(7)
			.fill(null)
			.map(() => []);
		Object.entries(daysObj).forEach(([dayKey, activities]) => {
			formatted[parseInt(dayKey)] = activities;
		});
		return formatted;
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

	function getWeekDateRange(weekKey) {
		const start = new Date(weekKey + 'T00:00:00');
		const end = new Date(start);
		end.setDate(end.getDate() + 6);
		const formatDate = (d) => `${String(d.getDate()).padStart(2, '0')}.${String(d.getMonth() + 1).padStart(2, '0')}.${d.getFullYear()}`;
		return `${formatDate(start)} - ${formatDate(end)}`;
	}

	onMount(async () => {
		await loadAllData();
	});
</script>

<div class="week-calendar-container">
	{#if loading}
		<p>Loading week data...</p>
	{:else if currentWeek}
		<div class="controls">
			<label>Week Type:</label>
			<div class="dropdown" class:open={dropdownOpen}>
				<button class="dropdown-button" onclick={() => (dropdownOpen = !dropdownOpen)}>
					{selectedWeek === 'peak' ? 'Peak (Most Hours)' : selectedWeek === 'average' ? 'Average' : 'Low Intensity'}
					<span class="dropdown-arrow" class:rotated={dropdownOpen}>▼</span>
				</button>
				{#if dropdownOpen}
					<div class="dropdown-menu">
						<button
							class="dropdown-item"
							class:active={selectedWeek === 'peak'}
							onclick={() => {
								selectedWeek = 'peak';
								currentWeek = weeks['peak'];
								dropdownOpen = false;
							}}
						>
							Peak (Most Hours)
						</button>
						<button
							class="dropdown-item"
							class:active={selectedWeek === 'average'}
							onclick={() => {
								selectedWeek = 'average';
								currentWeek = weeks['average'];
								dropdownOpen = false;
							}}
						>
							Average
						</button>
						<button
							class="dropdown-item"
							class:active={selectedWeek === 'low'}
							onclick={() => {
								selectedWeek = 'low';
								currentWeek = weeks['low'];
								dropdownOpen = false;
							}}
						>
							Low Intensity
						</button>
					</div>
				{/if}
			</div>
		</div>

		<div class="week-info">
			<h3>{selectedWeek.toUpperCase()}</h3>
			<p>{getWeekDateRange(currentWeek.key)}</p>
			<p>{currentWeek.sessionCount} sessions • {minutesToHours(currentWeek.totalMinutes)} hours</p>
		</div>

		<div class="week-grid">
			{#each dayNames as dayName, dayIndex}
				<div class="day-column">
					<div class="day-header">{dayName}</div>
					<div class="activities">
						{#each currentWeek.days[dayIndex] as activity}
							<div
								class="activity"
								style="background-color: {sportColors[activity.type] || '#ff8b4c'}"
							>
								<span class="sport">{activity.type}</span>
								<span class="duration">{minutesToHours(activity.duration)}h</span>
							</div>
						{:else}
							<div class="empty">-</div>
						{/each}
					</div>
				</div>
			{/each}
		</div>
	{/if}
</div>

<style>
	.week-calendar-container {
		padding: 3rem 1.5rem;
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.controls {
		margin-bottom: 2rem;
	}

	.controls {
		margin-bottom: 2rem;
		display: flex;
		gap: 1rem;
		align-items: center;
		color: #ff8b4c;
		font-size: 1rem;
	}

	.dropdown {
		position: relative;
	}

	.dropdown-button {
		padding: 0.5rem 1rem;
		background-color: #0f0f0f;
		color: #ff8b4c;
		border: 1px solid #ff8b4c;
		border-radius: 4px;
		cursor: pointer;
		font-family: inherit;
		font-size: 0.95rem;
		transition: all 0.2s ease;
		display: flex;
		align-items: center;
		gap: 0.5rem;
		min-width: 180px;
		white-space: nowrap;
	}

	.dropdown-button:hover {
		background-color: rgba(255, 139, 76, 0.1);
	}

	.dropdown-arrow {
		font-size: 0.7rem;
		transition: transform 0.2s ease;
	}

	.dropdown-arrow.rotated {
		transform: rotate(180deg);
	}

	.dropdown-menu {
		display: block;
		position: absolute;
		top: 100%;
		left: 0;
		background-color: #0f0f0f;
		border: 1px solid #ff8b4c;
		border-top: none;
		border-radius: 0 0 4px 4px;
		min-width: 380px;
		z-index: 10;
	}

	.dropdown-item {
		display: block;
		width: 100%;
		padding: 0.75rem 1rem;
		background: none;
		border: none;
		color: #ff8b4c;
		text-align: left;
		cursor: pointer;
		transition: all 0.2s ease;
		font-family: inherit;
		font-size: 0.95rem;
		white-space: nowrap;
	}

	.dropdown-item:hover {
		background-color: rgba(255, 139, 76, 0.15);
	}

	.dropdown-item.active {
		background-color: rgba(255, 139, 76, 0.25);
		font-weight: 600;
	}

	.week-info {
		text-align: center;
		margin-bottom: 2rem;
		color: #ff8b4c;
	}

	.week-info h3 {
		font-size: 1.5rem;
		margin-bottom: 0.5rem;
	}

	.week-info p {
		font-size: 0.9rem;
		opacity: 0.7;
	}

	.week-grid {
		display: grid;
		grid-template-columns: repeat(7, minmax(80px, 1fr));
		gap: 1rem;
		max-width: 1200px;
		width: 100%;
	}

	.week-calendar-container {
		overflow-x: auto;
	}

	.day-column {
		display: flex;
		flex-direction: column;
		min-width: 80px;
	}

	.day-header {
		padding: 0.75rem;
		text-align: center;
		color: #ff8b4c;
		font-weight: 600;
		font-size: 0.9rem;
		border-bottom: 2px solid rgba(255, 139, 76, 0.3);
		margin-bottom: 0.5rem;
	}

	.activities {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		min-height: 60px;
	}

	.activity {
		padding: 0.75rem;
		border-radius: 4px;
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 0.25rem;
		opacity: 0.85;
		transition: opacity 0.2s ease;
	}

	.activity:hover {
		opacity: 1;
	}

	.sport {
		font-size: 0.75rem;
		font-weight: 600;
		color: #0f0f0f;
	}

	.duration {
		font-size: 0.85rem;
		color: #0f0f0f;
		font-weight: 500;
	}

	.empty {
		color: rgba(255, 139, 76, 0.3);
		text-align: center;
		padding: 1rem 0;
	}

	@media (min-width: 768px) {
		.week-calendar-container {
			padding: 3rem 3rem;
		}

		.week-grid {
			gap: 1.5rem;
		}

		.day-column {
			min-width: 140px;
		}
	}
</style>
