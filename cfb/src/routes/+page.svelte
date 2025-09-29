<script>
	import { onMount, tick } from 'svelte';
	import { browser } from '$app/environment';

	let games = [];
	let filteredGames = [];
	let loading = true;
	let error = null;
	let lastUpdated = null;
	let selectedGame = null;
	let showModal = false;
	
	// Filter states
	let showFilters = false;
	let selectedConferences = [];
	let selectedStatuses = [];
	let selectedNetworks = [];
	let selectedRankings = [];
	let minRecord = '';
	let maxRecord = '';
	let minLosses = '';
	let maxLosses = '';
	let dateRange = { start: '', end: '' };
	let searchTerm = '';
	let sortBy = 'date';
	let sortOrder = 'desc';
	let showOnlyRanked = false;
	let showOnlyPower5 = false;
	let showOnlyLive = false;
	// Removed rivalry toggle (no reliable signal from API)


	// Function to fetch CFB games from ESPN API
	async function fetchGames() {
		try {
			loading = true;
			error = null;
			
			// Get current week's games with Sunday early-morning grace period
			const { start: startOfWeek, end: endOfWeek } = getCurrentWeekRange(6);
			
			// Use local date formatting to avoid UTC shifting
			const startDateStr = formatDateAsYYYYMMDDLocal(startOfWeek);
			const endDateStr = formatDateAsYYYYMMDDLocal(endOfWeek);
			
			console.log('Fetching games for week:', startDateStr, 'to', endDateStr);
			
			const endpoints = [
				`https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard?dates=${startDateStr}-${endDateStr}`,
				'https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard',
				'https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard?limit=100'
			];
			
			let allGames = [];
			let data = null;
			
			// Try each endpoint
			for (const endpoint of endpoints) {
				try {
					console.log('Trying endpoint:', endpoint);
					const response = await fetch(endpoint);
					
					if (response.ok) {
						data = await response.json();
						console.log(`Success with ${endpoint}:`, data.events?.length || 0, 'games');
						
						if (data.events && data.events.length > 0) {
							allGames = [...allGames, ...data.events];
							// Don't break, continue to get more games from other endpoints
						}
					} else {
						console.log(`HTTP error with ${endpoint}:`, response.status);
					}
				} catch (err) {
					console.log(`Failed with ${endpoint}:`, err.message);
					continue;
				}
			}
			
			if (allGames.length === 0 && data) {
				allGames = data.events || [];
			}
			
			console.log('Total games found:', allGames.length);
			
			// Remove duplicates based on game ID
			const uniqueGames = allGames.filter((game, index, self) => 
				index === self.findIndex(g => g.id === game.id)
			);
			
			console.log('Unique games after deduplication:', uniqueGames.length);
			
			// Filter out old games and only keep current week (with grace)
			const currentWeekGames = uniqueGames.filter(event => {
				const gameDate = new Date(event.date);
				const { start: startOfWeek, end: endOfWeek } = getCurrentWeekRange(6);
				return gameDate >= startOfWeek && gameDate <= endOfWeek;
			});
			
			console.log('Current week games found:', currentWeekGames.length);
			
			// Process the games data
			games = currentWeekGames.map(event => {
				const competition = event.competitions[0];
				const homeTeam = competition.competitors.find(c => c.homeAway === 'home');
				const awayTeam = competition.competitors.find(c => c.homeAway === 'away');
				
				return {
					id: event.id,
					name: event.name,
					date: event.date,
					status: competition.status.type.description,
					statusType: competition.status.type.name,
					homeTeam: {
						name: homeTeam.team.displayName,
						abbreviation: homeTeam.team.abbreviation,
					logo: homeTeam.team.logo,
						score: homeTeam.score,
						record: homeTeam.records?.[0]?.summary || '',
					conference: (competition.conferenceCompetition && competition.groups?.shortName) ? competition.groups.shortName : (homeTeam.team.conference || null),
					conferenceId: homeTeam.team.conferenceId || homeTeam.team.conference?.id || null,
						ranked: (homeTeam.curatedRank?.current && homeTeam.curatedRank.current !== 99) ? homeTeam.curatedRank.current : null,
						receivingVotes: homeTeam.team.receivingVotes || false,
						location: homeTeam.team.location || 'Unknown',
						// Add more ranking data
						apRank: (homeTeam.curatedRank?.current && homeTeam.curatedRank.current !== 99) ? homeTeam.curatedRank.current : null,
						coachesRank: homeTeam.team.coachesRank || null,
						cfpRank: homeTeam.team.cfpRank || null
					},
					awayTeam: {
						name: awayTeam.team.displayName,
						abbreviation: awayTeam.team.abbreviation,
					logo: awayTeam.team.logo,
						score: awayTeam.score,
						record: awayTeam.records?.[0]?.summary || '',
					conference: (competition.conferenceCompetition && competition.groups?.shortName) ? competition.groups.shortName : (awayTeam.team.conference || null),
					conferenceId: awayTeam.team.conferenceId || awayTeam.team.conference?.id || null,
						ranked: (awayTeam.curatedRank?.current && awayTeam.curatedRank.current !== 99) ? awayTeam.curatedRank.current : null,
						receivingVotes: awayTeam.team.receivingVotes || false,
						location: awayTeam.team.location || 'Unknown',
						// Add more ranking data
						apRank: (awayTeam.curatedRank?.current && awayTeam.curatedRank.current !== 99) ? awayTeam.curatedRank.current : null,
						coachesRank: awayTeam.team.coachesRank || null,
						cfpRank: awayTeam.team.cfpRank || null
					},
					gameDetails: {
						quarter: competition.status.period,
						clock: competition.status.displayClock,
						possession: competition.situation?.possession ? 
							(competition.situation.possession === homeTeam.team.id ? homeTeam.team.displayName : 
							 competition.situation.possession === awayTeam.team.id ? awayTeam.team.displayName : 
							 competition.situation.possession) : null,
						yardLine: competition.situation?.yardLine,
						down: competition.situation?.down,
						distance: competition.situation?.distance,
						redZone: competition.situation?.isRedZone,
						weather: event.weather?.displayValue,
						temperature: event.weather?.temperature,
						highTemperature: event.weather?.highTemperature,
						condition: event.weather?.conditionId
					},
					network: (
						event.broadcasts?.[0]?.names?.[0] ||
						event.geoBroadcasts?.[0]?.media?.shortName ||
						competition.broadcasts?.[0]?.names?.[0] ||
						competition.broadcasts?.[0]?.media?.shortName ||
						'TBD'
					)
				};
			});
			
			lastUpdated = new Date();
			loading = false;
			applyFilters(); // Apply filters after loading games
		} catch (err) {
			error = err.message;
			loading = false;
		}
	}
	
	// Conference definitions
	const conferences = {
		'Power 5': ['SEC', 'Big Ten', 'ACC', 'Pac-12', 'Big 12'],
		'SEC': ['SEC'],
		'Big Ten': ['Big Ten'],
		'ACC': ['ACC'],
		'Pac-12': ['Pac-12'],
		'Big 12': ['Big 12'],
		'Group of 5': ['AAC', 'C-USA', 'MAC', 'Mountain West', 'Sun Belt'],
		'AAC': ['AAC'],
		'C-USA': ['C-USA'],
		'MAC': ['MAC'],
		'Mountain West': ['Mountain West'],
		'Sun Belt': ['Sun Belt'],
		'Independent': ['Independent'],
		'FCS': ['FCS']
	};
	
	// Game status options
	const statusOptions = [
		{ value: 'STATUS_SCHEDULED', label: 'Scheduled', color: '#3b82f6' },
		{ value: 'STATUS_IN_PROGRESS', label: 'Live', color: '#22c55e' },
		{ value: 'STATUS_FINAL', label: 'Final', color: '#6b7280' },
		{ value: 'STATUS_POSTPONED', label: 'Postponed', color: '#f59e0b' },
		{ value: 'STATUS_CANCELLED', label: 'Cancelled', color: '#ef4444' }
	];
	
	// TV Networks
	const networks = [
		'ESPN', 'ESPN2', 'ESPNU', 'ESPN+', 'ABC', 'CBS', 'FOX', 'NBC', 'FS1', 'SEC Network',
		'Big Ten Network', 'ACC Network', 'Pac-12 Network', 'CBS Sports Network',
		'CBSSN', 'NFL Network', 'Peacock', 'Paramount+', 'TBD'
	];
	
	// Ranking tiers
	const rankingTiers = [
		{ value: 'top5', label: 'Top 5' },
		{ value: 'top10', label: 'Top 10' },
		{ value: 'top15', label: 'Top 15' },
		{ value: 'top25', label: 'Top 25' },
		{ value: 'receiving_votes', label: 'Receiving Votes' },
		{ value: 'unranked', label: 'Unranked' }
	];
	
	// Rivalry games (simplified list)
	const rivalryGames = [
		'Alabama vs Auburn', 'Ohio State vs Michigan', 'Oregon vs Washington',
		'USC vs UCLA', 'Florida vs Georgia', 'Texas vs Oklahoma',
		'Notre Dame vs USC', 'Army vs Navy', 'Harvard vs Yale'
	];
	
	// Apply all filters
	function applyFilters() {
		if (!games || games.length === 0) {
			filteredGames = [];
			return;
		}
		
		filteredGames = games.filter(game => {
			// Conference filter
			if (selectedConferences.length > 0) {
				const gameConferences = [
					game.homeTeam.conference,
					game.awayTeam.conference
				].filter(Boolean);
				
				const hasSelectedConference = selectedConferences.some(conf => {
					const confList = conferences[conf];
					if (!confList) return false;
					return gameConferences.some(gameConf => confList.includes(gameConf));
				});
				
				if (!hasSelectedConference) return false;
			}
			
			// Status filter
			if (selectedStatuses.length > 0 && !selectedStatuses.includes(game.statusType)) {
				return false;
			}
			
			// Network filter
			if (selectedNetworks.length > 0 && !selectedNetworks.includes(game.network)) {
				return false;
			}
			
			// Ranking filter
			if (selectedRankings.length > 0) {
				const homeRanked = game.homeTeam.ranked;
				const awayRanked = game.awayTeam.ranked;
				const homeReceivingVotes = game.homeTeam.receivingVotes;
				const awayReceivingVotes = game.awayTeam.receivingVotes;
				
				const hasSelectedRanking = selectedRankings.some(ranking => {
					switch (ranking) {
						case 'top5':
							return (homeRanked && homeRanked <= 5) || (awayRanked && awayRanked <= 5);
						case 'top10':
							return (homeRanked && homeRanked <= 10) || (awayRanked && awayRanked <= 10);
						case 'top15':
							return (homeRanked && homeRanked <= 15) || (awayRanked && awayRanked <= 15);
						case 'top25':
							return (homeRanked && homeRanked <= 25) || (awayRanked && awayRanked <= 25);
						case 'receiving_votes':
							return homeReceivingVotes || awayReceivingVotes;
						case 'unranked':
							return !homeRanked && !awayRanked && !homeReceivingVotes && !awayReceivingVotes;
						default:
							return true;
					}
				});
				
				if (!hasSelectedRanking) return false;
			}
			
			// Record filter
			if (minRecord || maxRecord) {
				const homeRecord = parseRecord(game.homeTeam.record);
				const awayRecord = parseRecord(game.awayTeam.record);
				
				const minWins = parseInt(minRecord);
				if (!isNaN(minWins) && homeRecord.wins < minWins && awayRecord.wins < minWins) return false;
				
				const maxWins = parseInt(maxRecord);
				if (!isNaN(maxWins) && homeRecord.wins > maxWins && awayRecord.wins > maxWins) return false;
				
				const minLs = parseInt(minLosses);
				if (!isNaN(minLs) && homeRecord.losses < minLs && awayRecord.losses < minLs) return false;
				
				const maxLs = parseInt(maxLosses);
				if (!isNaN(maxLs) && homeRecord.losses > maxLs && awayRecord.losses > maxLs) return false;
			}
			
			// Date range filter (inclusive, same-day friendly)
			if (dateRange.start || dateRange.end) {
				const gameDate = new Date(game.date);
				if (isNaN(gameDate.getTime())) return true;
				
				if (dateRange.start) {
					const startDate = new Date(dateRange.start);
					startDate.setHours(0,0,0,0);
					if (!isNaN(startDate.getTime()) && gameDate < startDate) return false;
				}
				
				if (dateRange.end) {
					const endDate = new Date(dateRange.end);
					endDate.setHours(23,59,59,999);
					if (!isNaN(endDate.getTime()) && gameDate > endDate) return false;
				}
			}
			
			// Search term filter
			if (searchTerm && searchTerm.trim()) {
				const searchLower = searchTerm.toLowerCase().trim();
				const matchesSearch = 
					(game.name || '').toLowerCase().includes(searchLower) ||
					(game.homeTeam.name || '').toLowerCase().includes(searchLower) ||
					(game.awayTeam.name || '').toLowerCase().includes(searchLower) ||
					(game.homeTeam.abbreviation || '').toLowerCase().includes(searchLower) ||
					(game.awayTeam.abbreviation || '').toLowerCase().includes(searchLower);
				
				if (!matchesSearch) return false;
			}
			
			// Quick filters
			if (showOnlyRanked && !game.homeTeam.ranked && !game.awayTeam.ranked) return false;
			if (showOnlyPower5) {
				const homeP5 = isPower5(game.homeTeam.conference) || isPower5(game.homeTeam.conferenceId);
				const awayP5 = isPower5(game.awayTeam.conference) || isPower5(game.awayTeam.conferenceId);
				if (!homeP5 && !awayP5) return false;
			}
			if (showOnlyLive && game.statusType !== 'STATUS_IN_PROGRESS') return false;
			
			return true;
		});
		
		// Sort games
		filteredGames.sort((a, b) => {
			let aValue, bValue;
			
			switch (sortBy) {
				case 'date':
					aValue = new Date(a.date);
					bValue = new Date(b.date);
					break;
				case 'score':
					aValue = (a.homeTeam.score || 0) + (a.awayTeam.score || 0);
					bValue = (b.homeTeam.score || 0) + (b.awayTeam.score || 0);
					break;
				case 'rank': {
					const aRanks = [a.homeTeam.ranked, a.awayTeam.ranked].filter(r => typeof r === 'number' && r > 0 && r < 99);
					const bRanks = [b.homeTeam.ranked, b.awayTeam.ranked].filter(r => typeof r === 'number' && r > 0 && r < 99);
					// For games, pick the best (lowest) rank among the two teams; unranked -> 99
					aValue = aRanks.length > 0 ? Math.min(...aRanks) : 99;
					bValue = bRanks.length > 0 ? Math.min(...bRanks) : 99;
					break;
				}
				case 'name':
					aValue = a.name;
					bValue = b.name;
					break;
				case 'status':
					aValue = a.statusType;
					bValue = b.statusType;
					break;
				default:
					aValue = new Date(a.date);
					bValue = new Date(b.date);
			}
			
			if (sortOrder === 'asc') {
				return aValue > bValue ? 1 : -1;
			} else {
				return aValue < bValue ? 1 : -1;
			}
		});
	}
	
	// Helper functions
	// Keep cards minimal height: disable equalization
	async function equalizeCardHeights() {
		await tick();
	if (!browser) return;
	/** @type {HTMLElement | null} */
	const grid = document.querySelector('.games-grid');
		if (!grid) return;
		grid.style.removeProperty('--card-equal-height');
		/** @type {NodeListOf<HTMLElement>} */
		const cards = grid.querySelectorAll('.game-card');
		cards.forEach(card => (card.style.minHeight = '')); // natural
	}

	function openGameModal(game) {
		selectedGame = game;
		showModal = true;
	}

	function closeGameModal() {
		showModal = false;
		selectedGame = null;
	}

	function onCardKeydown(event, game) {
		if (event.key === 'Enter' || event.key === ' ') {
			event.preventDefault();
			openGameModal(game);
		}
	}

	function formatStatusOrTime(game) {
		if (game.statusType === 'STATUS_IN_PROGRESS') {
			const q = game.gameDetails?.quarter ? `Q${game.gameDetails.quarter}` : '';
			const clk = game.gameDetails?.clock || '';
			return [q, clk].filter(Boolean).join(' ');
		}
		if (game.statusType === 'STATUS_FINAL') return 'Final';
		return formatGameTime(game.date);
	}

	function formatCompactMeta(game) {
		const when = formatStatusOrTime(game);
		const net = game.network || 'TBD';
		return `${net} • ${when}`;
	}
	/**
	 * Compute the current CFB week range in local time.
	 * Treat early Sunday (before graceHours) as part of the previous week.
	 * @param {number} graceHours
	 * @returns {{ start: Date, end: Date }}
	 */
	function getCurrentWeekRange(graceHours = 6) {
		const now = new Date();
		// If it's early Sunday morning (before grace), still treat as previous week
		const reference = new Date(now);
		if (reference.getDay() === 0 && reference.getHours() < graceHours) {
			reference.setDate(reference.getDate() - 1); // use Saturday as reference
		}

		// Start of week: Sunday 00:00 local
		const start = new Date(reference);
		start.setHours(0, 0, 0, 0);
		start.setDate(start.getDate() - start.getDay());

		// End of week with grace: next Sunday at graceHours:00 local
		const end = new Date(start);
		end.setDate(end.getDate() + 7); // move to next Sunday
		end.setHours(graceHours, 0, 0, 0);

		return { start, end };
	}

	/**
	 * Format a Date as YYYYMMDD using local time (avoids UTC date-shift).
	 * @param {Date} date
	 * @returns {string}
	 */
	function formatDateAsYYYYMMDDLocal(date) {
		const year = date.getFullYear();
		const month = String(date.getMonth() + 1).padStart(2, '0');
		const day = String(date.getDate()).padStart(2, '0');
		return `${year}${month}${day}`;
	}

	function parseRecord(recordStr) {
		if (!recordStr || typeof recordStr !== 'string') return { wins: 0, losses: 0 };
		const parts = recordStr.split('-');
		if (parts.length < 2) return { wins: 0, losses: 0 };
		return {
			wins: parseInt(parts[0]) || 0,
			losses: parseInt(parts[1]) || 0
		};
	}
	
	function isPower5(conference) {
		// Accept either conference name or ESPN conferenceId (Power 5: 5 Big Ten, 2 SEC, 1 ACC, 3 Big 12, 9 Pac-12 historically)
		const power5Names = ['SEC', 'Big Ten', 'ACC', 'Pac-12', 'Big 12'];
		const power5Ids = ['5','2','1','3','9'];
		if (!conference) return false;
		if (typeof conference === 'string') {
			return power5Names.includes(conference);
		}
		if (typeof conference === 'number') {
			return power5Ids.includes(String(conference));
		}
		if (typeof conference === 'object') {
			const id = conference.id?.toString?.();
			const name = conference.name;
			return (id && power5Ids.includes(id)) || (name && power5Names.includes(name));
		}
		return false;
	}
	
	function isRivalryGame(game) {
		if (!game || !game.name) return false;
		const gameName = game.name.toLowerCase();
		return rivalryGames.some(rivalry => {
			const rivalryLower = rivalry.toLowerCase();
			const parts = rivalryLower.split(' vs ');
			if (parts.length !== 2) return false;
			return gameName.includes(parts[0]) && gameName.includes(parts[1]);
		});
	}
	
	function clearAllFilters() {
		selectedConferences = [];
		selectedStatuses = [];
		selectedNetworks = [];
		selectedRankings = [];
		minRecord = '';
		maxRecord = '';
		minLosses = '';
		maxLosses = '';
		dateRange = { start: '', end: '' };
		searchTerm = '';
		sortBy = 'date';
		sortOrder = 'asc';
		showOnlyRanked = false;
		showOnlyPower5 = false;
		showOnlyLive = false;
		// no-op (rivalry removed)
		applyFilters();
	}
	
	// Make filters reactive to changes: explicitly depend on all filter states and games
	$: {
		selectedConferences;
		selectedStatuses;
		selectedNetworks;
		selectedRankings;
		minRecord;
		maxRecord;
		minLosses;
		maxLosses;
		dateRange.start;
		dateRange.end;
		searchTerm;
		showOnlyRanked;
		showOnlyPower5;
		showOnlyLive;
		sortBy;
		sortOrder;
		games;
		applyFilters();
	}

	// Auto-refresh every 30 seconds for live games
	let refreshInterval;

	onMount(() => {
		fetchGames();
		
		// Set up auto-refresh
		refreshInterval = setInterval(() => {
			fetchGames();
		}, 30000);
		
		const handleResize = () => equalizeCardHeights();
		window.addEventListener('resize', handleResize);
		window.addEventListener('orientationchange', handleResize);
		
		return () => {
			if (refreshInterval) {
				clearInterval(refreshInterval);
			}
			window.removeEventListener('resize', handleResize);
			window.removeEventListener('orientationchange', handleResize);
		};
	});

	// Recompute equal heights whenever the visible cards change
	$: {
		filteredGames;
		equalizeCardHeights();
	}

	// Function to get game status color
	function getStatusColor(statusType) {
		switch (statusType) {
			case 'STATUS_IN_PROGRESS':
				return '#22c55e';
			case 'STATUS_SCHEDULED':
				return '#3b82f6';
			case 'STATUS_FINAL':
				return '#6b7280';
			default:
				return '#6b7280';
		}
	}

	// Function to format game time
	function formatGameTime(dateString) {
		const date = new Date(dateString);
		return date.toLocaleTimeString('en-US', { 
			hour: 'numeric', 
			minute: '2-digit',
			timeZoneName: 'short'
		});
	}
	
	// Function to format temperature
	function formatTemperature(gameDetails) {
		const weatherText = gameDetails.weather || '';
		// If temperature is null/undefined, just show the weather text
		if (gameDetails.temperature == null) return weatherText;
		
		const fahrenheit = gameDetails.temperature;
		return `${fahrenheit}°F${weatherText ? ` - ${weatherText}` : ''}`;
	}
</script>

<svelte:head>
	<title>CFB Games - Live Scores & Stats</title>
	<meta name="description" content="Live college football scores, stats, and game details" />
</svelte:head>

<main class="container">
	<header class="header">
		<h1>🏈 College Football Live Scores</h1>
		<div class="header-info">
			{#if lastUpdated}
				<p class="last-updated">Last updated: {lastUpdated.toLocaleTimeString()}</p>
			{/if}
			{#if games.length > 0}
				<p class="game-count">{filteredGames.length} of {games.length} games</p>
			{/if}
			<div class="header-actions">
				<button on:click={() => showFilters = !showFilters} class="filter-toggle-btn">
					🔍 {showFilters ? 'Hide' : 'Show'} Filters
				</button>
				<button on:click={fetchGames} class="refresh-btn" disabled={loading}>
					{loading ? '🔄' : '↻'} {loading ? 'Loading...' : 'Refresh'}
				</button>
			</div>
		</div>
	</header>

	{#if showFilters}
		<div class="filters-panel">
			<div class="filters-header">
				<h3>🔍 Advanced Filters</h3>
				<button on:click={clearAllFilters} class="clear-filters-btn">Clear All</button>
			</div>
			
			<div class="filters-grid">
				<!-- Search -->
				<div class="filter-group">
					<label>Search Teams/Games</label>
					<input 
						type="text" 
						bind:value={searchTerm} 
						placeholder="Search by team name or game..."
						class="search-input"
					/>
				</div>

				<!-- Quick Filters -->
				<div class="filter-group">
					<label>Quick Filters</label>
					<div class="quick-filters">
						<label class="checkbox-label">
							<input type="checkbox" bind:checked={showOnlyRanked}>
							🏆 Ranked Teams Only
						</label>
						<label class="checkbox-label">
							<input type="checkbox" bind:checked={showOnlyPower5}>
							⚡ Power 5 Only
						</label>
						<label class="checkbox-label">
							<input type="checkbox" bind:checked={showOnlyLive}>
							🔴 Live Games Only
						</label>
					</div>
				</div>

				<!-- Conference Filter -->
				<div class="filter-group">
					<label>Conference</label>
					<div class="checkbox-grid">
						{#each Object.keys(conferences) as conf}
							<label class="checkbox-label">
								<input 
									type="checkbox" 
									value={conf}
									bind:group={selectedConferences}
								>
								{conf}
							</label>
						{/each}
					</div>
				</div>

				<!-- Game Status -->
				<div class="filter-group">
					<label>Game Status</label>
					<div class="checkbox-grid">
						{#each statusOptions as status}
							<label class="checkbox-label">
								<input 
									type="checkbox" 
									value={status.value}
									bind:group={selectedStatuses}
								>
								<span style="color: {status.color}">●</span> {status.label}
							</label>
						{/each}
					</div>
				</div>

				<!-- TV Network -->
				<div class="filter-group">
					<label>TV Network</label>
					<div class="checkbox-grid">
						{#each networks as network}
							<label class="checkbox-label">
								<input 
									type="checkbox" 
									value={network}
									bind:group={selectedNetworks}
								>
								{network}
							</label>
						{/each}
					</div>
				</div>

				<!-- Ranking Tiers -->
				<div class="filter-group">
					<label>Team Rankings</label>
					<div class="checkbox-grid">
						{#each rankingTiers as tier}
							<label class="checkbox-label">
								<input 
									type="checkbox" 
									value={tier.value}
									bind:group={selectedRankings}
								>
								{tier.label}
							</label>
						{/each}
					</div>
				</div>

				<!-- Record Filter -->
				<div class="filter-group">
					<label>Team Record</label>
			<div class="record-filters four-cols">
				<div>
					<label>Min Wins</label>
					<input 
						type="number"
						min="0"
						bind:value={minRecord}
						placeholder="0"
						class="record-input"
					/>
				</div>
				<div>
					<label>Max Wins</label>
					<input 
						type="number"
						min="0"
						bind:value={maxRecord}
						placeholder="12"
						class="record-input"
					/>
				</div>
				<div>
					<label>Min Losses</label>
					<input 
						type="number"
						min="0"
						bind:value={minLosses}
						placeholder="0"
						class="record-input"
					/>
				</div>
				<div>
					<label>Max Losses</label>
					<input 
						type="number"
						min="0"
						bind:value={maxLosses}
						placeholder="12"
						class="record-input"
					/>
				</div>
			</div>
				</div>

				<!-- Date Range -->
				<div class="filter-group">
					<label>Date Range</label>
					<div class="date-filters">
						<div>
							<label>Start Date:</label>
							<input 
								type="date" 
								bind:value={dateRange.start}
								class="date-input"
							/>
						</div>
						<div>
							<label>End Date:</label>
							<input 
								type="date" 
								bind:value={dateRange.end}
								class="date-input"
							/>
						</div>
					</div>
				</div>

				<!-- Sort Options -->
				<div class="filter-group">
					<label>Sort By</label>
					<div class="sort-options">
						<select bind:value={sortBy} class="sort-select">
							<option value="date">Date</option>
							<option value="score">Total Score</option>
					<option value="rank">Rank</option>
							<option value="name">Game Name</option>
							<option value="status">Status</option>
						</select>
						<select bind:value={sortOrder} class="sort-select">
							<option value="asc">Ascending</option>
							<option value="desc">Descending</option>
						</select>
					</div>
				</div>
			</div>
		</div>
	{/if}

	{#if error}
		<div class="error">
			<p>❌ Error loading games: {error}</p>
			<button on:click={fetchGames} class="retry-btn">Try Again</button>
		</div>
	{:else if loading && games.length === 0}
		<div class="loading">
			<div class="spinner"></div>
			<p>Loading games...</p>
		</div>
	{:else if games.length === 0}
		<div class="no-games">
			<p>No games scheduled for this week</p>
		</div>
	{:else if filteredGames.length === 0}
		<div class="no-games">
			<p>No games match your current filters</p>
			<button on:click={clearAllFilters} class="clear-filters-btn">Clear Filters</button>
		</div>
	{:else}
		<div class="games-grid">
			{#each filteredGames as game (game.id)}
				<div class="game-card" class:live={game.statusType === 'STATUS_IN_PROGRESS'} role="button" tabindex="0" on:click={() => openGameModal(game)} on:keydown={(e) => onCardKeydown(e, game)}>
					<div class="game-compact">
						<img src={game.awayTeam.logo} alt={game.awayTeam.name} class="team-logo small" />
						<span class="abbr">
							{#if game.awayTeam.ranked}
								<span class="rank-mini">#{game.awayTeam.ranked}</span>
							{/if}
							{game.awayTeam.abbreviation || game.awayTeam.name}
						</span>
						<span class="team-score compact">{game.awayTeam.score ?? '-'}</span>
						<span class="vs">@</span>
						<span class="team-score compact">{game.homeTeam.score ?? '-'}</span>
						<span class="abbr">
							{#if game.homeTeam.ranked}
								<span class="rank-mini">#{game.homeTeam.ranked}</span>
							{/if}
							{game.homeTeam.abbreviation || game.homeTeam.name}
						</span>
						<img src={game.homeTeam.logo} alt={game.homeTeam.name} class="team-logo small" />
						<span class="meta">{formatCompactMeta(game)}</span>
					</div>
				</div>
			{/each}
		</div>
	{/if}
</main>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
		background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
		min-height: 100vh;
	}

	.container {
		max-width: 1400px;
		margin: 0 auto;
		padding: 20px;
	}

	.header {
		text-align: center;
		margin-bottom: 30px;
		color: white;
	}

	.header h1 {
		font-size: 2.5rem;
		margin: 0 0 10px 0;
		text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
	}

	.header-info {
		display: flex;
		justify-content: center;
		align-items: center;
		gap: 20px;
		flex-wrap: wrap;
	}
	
	.header-actions {
		display: flex;
		gap: 10px;
		align-items: center;
	}

	.last-updated, .game-count {
		margin: 0;
		opacity: 0.9;
	}
	
	.game-count {
		font-weight: 600;
		color: #22c55e;
	}

	.refresh-btn {
		background: rgba(255,255,255,0.2);
		border: 2px solid rgba(255,255,255,0.3);
		color: white;
		padding: 8px 16px;
		border-radius: 8px;
		cursor: pointer;
		font-weight: 600;
		transition: all 0.3s ease;
	}

	.refresh-btn:hover:not(:disabled) {
		background: rgba(255,255,255,0.3);
		transform: translateY(-2px);
	}

	.refresh-btn:disabled {
		opacity: 0.6;
		cursor: not-allowed;
	}

	.games-grid {
		display: grid;
		grid-template-columns: repeat(6, 1fr);
		gap: 10px;
	}

	.game-card {
		background: white;
		border-radius: 6px;
		padding: 6px 8px;
		box-shadow: 0 1px 2px rgba(0,0,0,0.06);
		transition: background 0.15s ease, transform 0.1s ease;
		border: 1px solid #e5e7eb;
		min-height: 44px;
		display: flex;
		align-items: center;
		cursor: pointer;
		user-select: none;
	}

	.game-card:hover {
		background: #f9fafb;
	}
	.game-compact {
		display: grid;
		grid-template-columns: 20px auto auto auto auto auto 20px minmax(110px, 1fr);
		align-items: center;
		gap: 6px;
		width: 100%;
	}

	.team-logo.small {
		width: 20px;
		height: 20px;
		object-fit: contain;
	}

	.abbr {
		font-weight: 600;
		font-size: 0.82rem;
		color: #111827;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		display: flex;
		align-items: center;
		gap: 4px;
	}

	.rank-mini {
		background: #f59e0b;
		color: white;
		font-size: 0.64rem;
		font-weight: 700;
		padding: 1px 4px;
		border-radius: 8px;
	}

	.team-score.compact {
		font-size: 0.95rem;
		font-weight: 700;
		text-align: center;
		min-width: 16px;
	}

	.meta {
		text-align: right;
		font-size: 0.78rem;
		color: #6b7280;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.game-card.live {
		border-color: #22c55e;
		box-shadow: 0 4px 6px rgba(34, 197, 94, 0.2);
	}

	.game-header {
		display: flex;
		justify-content: space-between;
		align-items: flex-start;
		margin-bottom: 15px;
	}



	.network {
		font-size: 0.85rem;
		color: #6b7280;
		background: #f3f4f6;
		padding: 2px 8px;
		border-radius: 4px;
	}

	.game-status {
		font-weight: 600;
		font-size: 0.9rem;
	}

	.teams {
		display: flex;
		align-items: center;
		gap: 15px;
		margin-bottom: 15px;
	}

	.team {
		display: flex;
		align-items: center;
		gap: 10px;
		flex: 1;
	}

	.away-team {
		justify-content: flex-start;
	}

	.home-team {
		justify-content: flex-end;
		flex-direction: row-reverse;
	}

	.team-logo {
		width: 48px;
		height: 48px;
		object-fit: contain;
	}

	.team-info {
		display: flex;
		flex-direction: column;
		align-items: center;
		text-align: center;
	}

	.team-brand {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 6px;
		flex: 1;
	}

	.team-name {
		font-weight: 600;
		font-size: 0.95rem;
		color: #1f2937;
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 6px;
	}
	.team-rank {
		margin-top: 2px;
	}
	
	.ranking-badge {
		background: #f59e0b;
		color: white;
		font-size: 0.7rem;
		font-weight: bold;
		padding: 2px 6px;
		border-radius: 10px;
		min-width: 20px;
		text-align: center;
	}

	.team-record {
		font-size: 0.8rem;
		color: #6b7280;
	}

	.team-score {
		font-size: 1.5rem;
		font-weight: bold;
		color: #1f2937;
		min-width: 30px;
		text-align: center;
	}

	.vs {
		font-size: 1.2rem;
		font-weight: bold;
		color: #6b7280;
	}

	.game-details {
		background: #f8fafc;
		border-radius: 8px;
		padding: 12px;
		margin-bottom: 10px;
	}

	.game-clock {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 8px;
	}

	.quarter {
		font-weight: 600;
		color: #1f2937;
	}

	.time {
		font-weight: bold;
		color: #dc2626;
		font-size: 1.1rem;
	}

	.possession, .yard-line {
		display: flex;
		justify-content: space-between;
		margin-bottom: 4px;
		font-size: 0.9rem;
	}

	.label {
		color: #6b7280;
	}

	.yard {
		font-weight: 600;
		color: #1f2937;
	}

	.down-distance {
		display: flex;
		gap: 5px;
		font-weight: 600;
		color: #1f2937;
	}

	.red-zone {
		color: #dc2626;
		font-weight: 600;
		font-size: 0.9rem;
	}

	.game-time {
		text-align: center;
		font-weight: 600;
		color: #3b82f6;
		padding: 8px;
		background: #eff6ff;
		border-radius: 6px;
	}

	.weather {
		text-align: center;
		font-size: 0.9rem;
		color: #6b7280;
		margin-top: 0;
	}

	.card-footer {
		margin-top: auto;
		display: flex;
		flex-direction: column;
		gap: 8px;
	}

	.loading, .error, .no-games {
		text-align: center;
		padding: 40px;
		background: white;
		border-radius: 12px;
		box-shadow: 0 4px 6px rgba(0,0,0,0.1);
	}

	.spinner {
		width: 40px;
		height: 40px;
		border: 4px solid #f3f4f6;
		border-top: 4px solid #3b82f6;
		border-radius: 50%;
		animation: spin 1s linear infinite;
		margin: 0 auto 20px;
	}

	@keyframes spin {
		0% { transform: rotate(0deg); }
		100% { transform: rotate(360deg); }
	}

	.retry-btn {
		background: #3b82f6;
		color: white;
		border: none;
		padding: 10px 20px;
		border-radius: 6px;
		cursor: pointer;
		font-weight: 600;
		margin-top: 10px;
	}

	.retry-btn:hover {
		background: #2563eb;
	}

	/* Filter Panel Styles */
	.filters-panel {
		background: white;
		border-radius: 12px;
		padding: 20px;
		margin-bottom: 20px;
		box-shadow: 0 4px 6px rgba(0,0,0,0.1);
		border: 2px solid #e5e7eb;
	}

	.filters-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 20px;
		padding-bottom: 15px;
		border-bottom: 2px solid #f3f4f6;
	}

	.filters-header h3 {
		margin: 0;
		color: #1f2937;
		font-size: 1.2rem;
	}

	.clear-filters-btn {
		background: #ef4444;
		color: white;
		border: none;
		padding: 8px 16px;
		border-radius: 6px;
		cursor: pointer;
		font-weight: 600;
		transition: background 0.3s ease;
	}

	.clear-filters-btn:hover {
		background: #dc2626;
	}

	.filters-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
		gap: 20px;
	}

	.filter-group {
		background: #f8fafc;
		padding: 15px;
		border-radius: 8px;
		border: 1px solid #e5e7eb;
	}

	.filter-group label {
		display: block;
		font-weight: 600;
		color: #374151;
		margin-bottom: 8px;
		font-size: 0.9rem;
	}

	.search-input {
		width: 100%;
		padding: 10px;
		border: 2px solid #d1d5db;
		border-radius: 6px;
		font-size: 0.9rem;
		transition: border-color 0.3s ease;
	}

	.search-input:focus {
		outline: none;
		border-color: #3b82f6;
	}

	.quick-filters {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
		gap: 10px;
	}

	.checkbox-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(120px, 1fr));
		gap: 8px;
		max-height: 200px;
		overflow-y: auto;
	}

	.checkbox-label {
		display: flex;
		align-items: center;
		gap: 6px;
		font-size: 0.85rem;
		cursor: pointer;
		padding: 4px 8px;
		border-radius: 4px;
		transition: background-color 0.2s ease;
	}

	.checkbox-label:hover {
		background-color: #f3f4f6;
	}

	.checkbox-label input[type="checkbox"] {
		margin: 0;
		cursor: pointer;
	}

	.record-filters, .date-filters {
		display: grid;
		grid-template-columns: 1fr 1fr;
		gap: 10px;
	}

	.record-filters.four-cols {
		grid-template-columns: repeat(4, minmax(100px, 1fr));
	}

	.record-filters > div, .date-filters > div {
		display: flex;
		flex-direction: column;
		gap: 4px;
	}

	.record-filters label, .date-filters label {
		font-size: 0.8rem;
		color: #6b7280;
		margin-bottom: 4px;
	}

	.record-input, .date-input {
		padding: 8px;
		border: 2px solid #d1d5db;
		border-radius: 4px;
		font-size: 0.85rem;
		transition: border-color 0.3s ease;
	}

	.record-input:focus, .date-input:focus {
		outline: none;
		border-color: #3b82f6;
	}

	.sort-options {
		display: flex;
		gap: 10px;
	}

	.sort-select {
		flex: 1;
		padding: 8px;
		border: 2px solid #d1d5db;
		border-radius: 4px;
		font-size: 0.85rem;
		background: white;
		cursor: pointer;
	}

	.sort-select:focus {
		outline: none;
		border-color: #3b82f6;
	}

	.filter-toggle-btn {
		background: rgba(255,255,255,0.2);
		border: 2px solid rgba(255,255,255,0.3);
		color: white;
		padding: 8px 16px;
		border-radius: 8px;
		cursor: pointer;
		font-weight: 600;
		transition: all 0.3s ease;
	}

	.filter-toggle-btn:hover {
		background: rgba(255,255,255,0.3);
		transform: translateY(-2px);
	}

	/* Enhanced game card styles for filtered view */
	.game-card.ranked {
		border-left: 4px solid #f59e0b;
	}

	.game-card.power5 {
		border-left: 4px solid #3b82f6;
	}

	.game-card.rivalry {
		border-left: 4px solid #ef4444;
		background: linear-gradient(135deg, #fef2f2 0%, #ffffff 100%);
	}

	/* Responsive design for filters */
	@media (max-width: 1280px) {
		.games-grid { grid-template-columns: repeat(4, 1fr); }
	}

	@media (max-width: 900px) {
		.games-grid { grid-template-columns: repeat(2, 1fr); }
	}

	@media (max-width: 768px) {
		.games-grid { grid-template-columns: 1fr; }
		
		.header h1 {
			font-size: 2rem;
		}
		
		.header-info {
			flex-direction: column;
			gap: 10px;
		}
		
		.filters-grid {
			grid-template-columns: 1fr;
		}
		
		.quick-filters {
			grid-template-columns: 1fr;
		}
		
		.checkbox-grid {
			grid-template-columns: 1fr;
		}
		
		.record-filters, .date-filters {
			grid-template-columns: 1fr;
		}
		
		.sort-options {
			flex-direction: column;
		}
	}

/* Simple modal */
.modal-backdrop {
	position: fixed;
	top: 0;
	left: 0;
	width: 100vw;
	height: 100vh;
	background: rgba(0,0,0,0.5);
	display: flex;
	align-items: center;
	justify-content: center;
	z-index: 1000;
}

.modal {
	background: #fff;
	border-radius: 12px;
	width: min(900px, 95vw);
	max-height: 85vh;
	overflow: auto;
	box-shadow: 0 10px 30px rgba(0,0,0,0.2);
	border: 1px solid #e5e7eb;
}

.modal-header {
	display: flex;
	justify-content: space-between;
	align-items: center;
	padding: 12px 16px;
	border-bottom: 1px solid #f3f4f6;
}

.modal-body { padding: 16px; }

.close-btn {
	background: transparent;
	border: none;
	font-size: 1.25rem;
	cursor: pointer;
}

.modal-card {
	background: white;
	border-radius: 12px;
	padding: 16px;
	border: 1px solid #e5e7eb;
	box-shadow: 0 4px 8px rgba(0,0,0,0.06);
}

.modal-row {
	display: grid;
	grid-template-columns: 1fr auto 1fr;
	align-items: center;
	gap: 16px;
}

.modal-team {
	display: flex;
	align-items: center;
	gap: 10px;
}

.modal-team.away { justify-content: flex-start; }
.modal-team.home { justify-content: flex-end; flex-direction: row-reverse; }

.modal-team-info { display: flex; flex-direction: column; align-items: center; text-align: center; }
.modal-team-info .name { font-weight: 700; color: #1f2937; }
.modal-team-info .record { font-size: 0.85rem; color: #6b7280; }
.modal-team-info .rank { margin-top: 4px; }

.modal-score { text-align: center; min-width: 220px; }
.modal-score .status { font-weight: 700; margin-bottom: 6px; }
.modal-score .scores { display: flex; align-items: center; gap: 10px; justify-content: center; font-size: 1.25rem; font-weight: 800; }
.modal-score .submeta { margin-top: 6px; color: #6b7280; font-size: 0.9rem; display: flex; gap: 6px; justify-content: center; align-items: center; }

.modal-network { font-weight: 700; color: #374151; }

.live-extras { margin-top: 16px; display: grid; gap: 6px; }
</style>

{#if showModal && selectedGame}
	<div class="modal-backdrop" on:click={closeGameModal}>
		<div class="modal" on:click|stopPropagation>
			<div class="modal-header">
				<div class="modal-network">{selectedGame.network}</div>
				<button class="close-btn" aria-label="Close" on:click={closeGameModal}>✕</button>
			</div>
			<div class="modal-body">
				<div class="modal-card">
					<div class="modal-row">
						<div class="modal-team away">
							<img src={selectedGame.awayTeam.logo} alt={selectedGame.awayTeam.name} class="team-logo" />
							<div class="modal-team-info">
								<div class="name">{selectedGame.awayTeam.name}</div>
								<div class="record">{selectedGame.awayTeam.record}</div>
								{#if selectedGame.awayTeam.ranked}
									<div class="rank"><span class="ranking-badge">#{selectedGame.awayTeam.ranked}</span></div>
								{/if}
							</div>
						</div>
						<div class="modal-score">
							<div class="status" style="color: {getStatusColor(selectedGame.statusType)}">{selectedGame.status}</div>
							<div class="scores">
								<span class="away-score">{selectedGame.awayTeam.score}</span>
								<span class="at">@</span>
								<span class="home-score">{selectedGame.homeTeam.score}</span>
							</div>
							<div class="submeta">
								<span>{formatGameTime(selectedGame.date)}</span>
								<span>•</span>
								<span>{formatTemperature(selectedGame.gameDetails)}</span>
							</div>
						</div>
						<div class="modal-team home">
							<img src={selectedGame.homeTeam.logo} alt={selectedGame.homeTeam.name} class="team-logo" />
							<div class="modal-team-info">
								<div class="name">{selectedGame.homeTeam.name}</div>
								<div class="record">{selectedGame.homeTeam.record}</div>
								{#if selectedGame.homeTeam.ranked}
									<div class="rank"><span class="ranking-badge">#{selectedGame.homeTeam.ranked}</span></div>
								{/if}
							</div>
						</div>
					</div>

					{#if selectedGame.statusType === 'STATUS_IN_PROGRESS'}
						<div class="live-extras">
							{#if selectedGame.gameDetails.possession}
								<div><strong>Possession:</strong> {selectedGame.gameDetails.possession}</div>
							{/if}
							{#if selectedGame.gameDetails.yardLine}
								<div><strong>Yard Line:</strong> {selectedGame.gameDetails.yardLine}</div>
							{/if}
							{#if selectedGame.gameDetails.down}
								<div><strong>Down & Distance:</strong> {selectedGame.gameDetails.down}{#if selectedGame.gameDetails.distance} & {selectedGame.gameDetails.distance}{/if}</div>
							{/if}
							{#if selectedGame.gameDetails.redZone}
								<div class="red-zone">🔴 Red Zone</div>
							{/if}
						</div>
					{/if}
				</div>
			</div>
		</div>
	</div>
{/if}
