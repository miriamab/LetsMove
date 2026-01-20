<script>
	import { onMount } from 'svelte';
	import { base } from '$app/paths';

	let runs = $state([]);
	let selectedRun = $state(0);
	let currentRun = $state(null);
	let loading = $state(true);
	let dropdownOpen = $state(false);

	const runFiles = [
		'activity_19450829184.gpx',
		'activity_20880715810.gpx',
		'activity_4765499487.gpx'
	];

	async function parseGPX(gpxText) {
		const parser = new DOMParser();
		const gpxDoc = parser.parseFromString(gpxText, 'application/xml');

		const trackPoints = [];
		const trkpts = gpxDoc.getElementsByTagName('trkpt');

		for (let trkpt of trkpts) {
			const lat = parseFloat(trkpt.getAttribute('lat'));
			const lon = parseFloat(trkpt.getAttribute('lon'));
			const timeEl = trkpt.getElementsByTagName('time')[0];
			const time = timeEl ? new Date(timeEl.textContent) : new Date();

			// Extract heart rate
			const hrEl = trkpt.querySelector('ns3\\:hr') || trkpt.querySelector('[nodeName="ns3:hr"]');
			const extensions = trkpt.getElementsByTagName('ns3:TrackPointExtension')[0];
			let hr = null;
			if (extensions) {
				const hrChild = extensions.getElementsByTagName('ns3:hr')[0];
				if (hrChild) {
					hr = parseInt(hrChild.textContent);
				}
			}

			trackPoints.push({ lat, lon, time, hr });
		}

		// Extract metadata
		const nameEl = gpxDoc.getElementsByTagName('name')[0];
		const name = nameEl ? nameEl.textContent : 'Run';

		// Calculate distance
		let distance = 0;
		for (let i = 1; i < trackPoints.length; i++) {
			distance += getDistance(
				trackPoints[i - 1].lat,
				trackPoints[i - 1].lon,
				trackPoints[i].lat,
				trackPoints[i].lon
			);
		}

		// Calculate pace per point
		const paceData = [];
		for (let i = 1; i < trackPoints.length; i++) {
			const segmentDist = getDistance(
				trackPoints[i - 1].lat,
				trackPoints[i - 1].lon,
				trackPoints[i].lat,
				trackPoints[i].lon
			);
			const segmentTime = (trackPoints[i].time - trackPoints[i - 1].time) / (1000 * 60); // minutes
			const segmentPace = segmentTime / (segmentDist || 0.001);
			paceData.push(segmentPace);
		}

		// Calculate pace (min/km)
		const duration = trackPoints[trackPoints.length - 1].time - trackPoints[0].time;
		const durationMinutes = duration / (1000 * 60);
		const pace = durationMinutes / distance;

		// Get HR data (filter nulls)
		const hrData = trackPoints.filter(p => p.hr !== null).map(p => p.hr);

		return {
			name,
			trackPoints,
			distance: distance.toFixed(2),
			pace: pace.toFixed(2),
			duration: formatDuration(durationMinutes),
			hrData,
			paceData: paceData.filter(p => p < 30 && p > 0) // Filter outliers
		};
	}

	function getDistance(lat1, lon1, lat2, lon2) {
		const R = 6371; // Earth radius in km
		const dLat = ((lat2 - lat1) * Math.PI) / 180;
		const dLon = ((lon2 - lon1) * Math.PI) / 180;
		const a =
			Math.sin(dLat / 2) * Math.sin(dLat / 2) +
			Math.cos((lat1 * Math.PI) / 180) *
				Math.cos((lat2 * Math.PI) / 180) *
				Math.sin(dLon / 2) *
				Math.sin(dLon / 2);
		const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
		return R * c;
	}

	function formatDuration(minutes) {
		const hours = Math.floor(minutes / 60);
		const mins = Math.round(minutes % 60);
		return `${String(hours).padStart(2, '0')}:${String(mins).padStart(2, '0')}`;
	}

	function generateLineChart(data, width = 140, height = 80) {
		if (!data || data.length === 0) return '';

		const padding = 12;
		const chartWidth = width - 2 * padding;
		const chartHeight = height - 2 * padding;

		const minVal = Math.min(...data);
		const maxVal = Math.max(...data);
		const range = maxVal - minVal || 1;

		const points = data
			.map((val, i) => {
				const x = padding + (i / (data.length - 1 || 1)) * chartWidth;
				const y = padding + chartHeight - ((val - minVal) / range) * chartHeight;
				return `${x},${y}`;
			})
			.join(' ');

		return `<svg width="${width}" height="${height}" viewBox="0 0 ${width} ${height}" style="border-radius: 2px; background: rgba(255, 139, 76, 0.02);">
			<polyline points="${points}" fill="none" stroke="#ff8b4c" stroke-width="1.5" opacity="0.8" />
		</svg>`;
	}

	function projectCoordinates(trackPoints) {
		if (trackPoints.length === 0) return [];

		const lats = trackPoints.map(p => p.lat);
		const lons = trackPoints.map(p => p.lon);

		const minLat = Math.min(...lats);
		const maxLat = Math.max(...lats);
		const minLon = Math.min(...lons);
		const maxLon = Math.max(...lons);

		const padding = 40;
		const width = 600;
		const height = 500;

		const latRange = maxLat - minLat || 0.01;
		const lonRange = maxLon - minLon || 0.01;

		const aspect = lonRange / latRange;
		let projWidth = width - 2 * padding;
		let projHeight = height - 2 * padding;

		if (aspect > 1) {
			projHeight = projWidth / aspect;
		} else {
			projWidth = projHeight * aspect;
		}

		return {
			points: trackPoints.map(p => ({
				x: padding + ((p.lon - minLon) / lonRange) * projWidth,
				y: padding + projHeight - ((p.lat - minLat) / latRange) * projHeight
			})),
			width,
			height
		};
	}

	function selectRun(index) {
		selectedRun = index;
		currentRun = runs[index];
	}

	async function loadRuns() {
		try {
			const loadedRuns = [];
			for (const file of runFiles) {
				const response = await fetch(`${base}/single_runs/${file}`);
				if (response.ok) {
					const gpxText = await response.text();
					const runData = await parseGPX(gpxText);
					loadedRuns.push(runData);
				}
			}

			runs = loadedRuns;
			if (runs.length > 0) {
				currentRun = runs[0];
			}
			loading = false;
		} catch (error) {
			console.error('Error loading runs:', error);
			loading = false;
		}
	}

	onMount(async () => {
		await loadRuns();
	});
</script>

<div class="pace-map-container">
	{#if loading}
		<p>Loading runs...</p>
	{:else if currentRun}
		{@const projection = projectCoordinates(currentRun.trackPoints)}
		<div class="controls">
			<label>Run:</label>
			<div class="dropdown" class:open={dropdownOpen}>
				<button class="dropdown-button" onclick={() => (dropdownOpen = !dropdownOpen)}>
					{currentRun.name}
					<span class="dropdown-arrow" class:rotated={dropdownOpen}>▼</span>
				</button>
				{#if dropdownOpen}
					<div class="dropdown-menu">
						{#each runs as run, index}
							<button
								class="dropdown-item"
								class:active={index === selectedRun}
								onclick={() => {
									selectRun(index);
									dropdownOpen = false;
								}}
							>
								{run.name}
							</button>
						{/each}
					</div>
				{/if}
			</div>
		</div>

		<div class="map-wrapper">
			<svg class="map" width={projection.width} height={projection.height} viewBox="0 0 {projection.width} {projection.height}">

				<polyline
					points={projection.points.map(p => `${p.x},${p.y}`).join(' ')}
					fill="none"
					stroke="#ff8b4c"
					stroke-width="2"
					opacity="0.9"
				/>

				<!-- Start marker -->
				{#if projection.points.length > 0}
					<circle
						cx={projection.points[0].x}
						cy={projection.points[0].y}
						r="5"
						fill="#ff8b4c"
						stroke="#0f0f0f"
						stroke-width="1.5"
					/>
				{/if}

				<!-- End marker -->
				{#if projection.points.length > 1}
					<circle
						cx={projection.points[projection.points.length - 1].x}
						cy={projection.points[projection.points.length - 1].y}
						r="5"
						fill="#ff8b4c"
						stroke="#0f0f0f"
						stroke-width="1.5"
					/>
				{/if}
			</svg>

			<div class="run-info">
				<div class="info-item">
					<span class="label">Distance</span>
					<span class="value">{currentRun.distance} km</span>
				</div>
				<div class="info-item">
					<span class="label">Pace</span>
					<span class="value">{currentRun.pace} min/km</span>
				</div>
				<div class="info-item">
					<span class="label">Duration</span>
					<span class="value">{currentRun.duration}</span>
				</div>

				{#if currentRun.hrData && currentRun.hrData.length > 0}
					<div class="info-chart">
						<span class="label">Heart Rate</span>
						{@html generateLineChart(currentRun.hrData, 140, 60)}
						<span class="chart-label">{Math.min(...currentRun.hrData)} - {Math.max(...currentRun.hrData)} bpm</span>
					</div>
				{/if}

				{#if currentRun.paceData && currentRun.paceData.length > 0}
					<div class="info-chart">
						<span class="label">Pace Over Time</span>
						{@html generateLineChart(currentRun.paceData, 140, 60)}
						<span class="chart-label">{(Math.min(...currentRun.paceData)).toFixed(1)} - {(Math.max(...currentRun.paceData)).toFixed(1)} min/km</span>
					</div>
				{/if}
			</div>
		</div>
	{/if}
</div>

<style>
	.pace-map-container {
		padding: 3rem 1.5rem;
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.controls {
		display: flex;
		gap: 1rem;
		align-items: center;
		margin-bottom: 2rem;
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
		min-width: 250px;
		z-index: 10;
	}

	.dropdown.open .dropdown-menu {
		display: block;
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
	}

	.dropdown-item:hover {
		background-color: rgba(255, 139, 76, 0.15);
	}

	.dropdown-item.active {
		background-color: rgba(255, 139, 76, 0.25);
		font-weight: 600;
	}

	.map-wrapper {
		width: 100%;
		max-width: 1300px;
		display: grid;
		grid-template-columns: 1fr 140px;
		gap: 0;
	}

	.map {
		width: 100%;
		border-radius: 4px;
		background: transparent;
	}

	.run-info {
		display: flex;
		flex-direction: column;
		gap: 1.5rem;
		padding: 0;
		background-color: transparent;
	}

	.info-item {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}

	.info-chart {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
	}

	.chart-label {
		font-size: 0.75rem;
		color: #ff8b4c;
		opacity: 0.6;
	}

	.label {
		font-size: 0.85rem;
		color: #ff8b4c;
		opacity: 0.7;
		text-transform: uppercase;
		letter-spacing: 0.05em;
	}

	.value {
		font-size: 1.2rem;
		color: #ff8b4c;
		font-weight: 600;
	}

	@media (min-width: 768px) {
		.pace-map-container {
			padding: 3rem 3rem;
		}

		.map-wrapper {
			grid-template-columns: 1fr 180px;
			gap: 0;
		}
	}

	@media (max-width: 768px) {
		.map-wrapper {
			grid-template-columns: 1fr;
		}

		.run-info {
			grid-row: 2;
			grid-template-columns: repeat(3, 1fr);
			flex-direction: row;
			gap: 1rem;
		}

		.info-item {
			text-align: center;
		}
	}
</style>
