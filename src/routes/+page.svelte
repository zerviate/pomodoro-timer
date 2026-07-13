<script lang="ts">
	import { onMount } from 'svelte';

	type Phase = 'work' | 'break';
	type Preset = { label: string; work: number; break: number };

	const STORAGE_KEY = 'pomodoro-settings:v1';
	const RING_RADIUS = 152;
	const RING_CIRCUMFERENCE = 2 * Math.PI * RING_RADIUS;
	const presets: Preset[] = [
		{ label: '25+5', work: 25, break: 5 },
		{ label: '50+10', work: 50, break: 10 }
	];

	let workMinutes = $state(25);
	let breakMinutes = $state(5);
	let activePhase = $state<Phase>('work');
	let remainingMs = $state(25 * 60_000);
	let isRunning = $state(false);

	let frameId: number | null = null;
	let phaseEndsAt = 0;
	let hasMounted = false;

	const clampMinutes = (value: number) => {
		if (!Number.isFinite(value)) return 1;
		return Math.min(180, Math.max(1, Math.round(value)));
	};

	const durationFor = (phase: Phase) => (phase === 'work' ? workMinutes : breakMinutes) * 60_000;

	const cancelFrame = () => {
		if (frameId !== null) {
			cancelAnimationFrame(frameId);
			frameId = null;
		}
	};

	const persistSettings = () => {
		if (!hasMounted) return;
		localStorage.setItem(
			STORAGE_KEY,
			JSON.stringify({
				workMinutes,
				breakMinutes
			})
		);
	};

	const resetTimer = (phase: Phase = 'work') => {
		cancelFrame();
		isRunning = false;
		activePhase = phase;
		remainingMs = durationFor(phase);
		phaseEndsAt = 0;
	};

	const queueNextPhase = (phase: Phase) => {
		activePhase = phase;
		remainingMs = durationFor(phase);
		phaseEndsAt = performance.now() + remainingMs;
		frameId = requestAnimationFrame(tick);
	};

	const advancePhase = () => {
		const nextPhase = activePhase === 'work' ? 'break' : 'work';
		queueNextPhase(nextPhase);
	};

	const tick = () => {
		if (!isRunning) return;

		const nextRemaining = Math.max(0, phaseEndsAt - performance.now());
		remainingMs = nextRemaining;

		if (nextRemaining <= 0) {
			advancePhase();
			return;
		}

		frameId = requestAnimationFrame(tick);
	};

	const startTimer = () => {
		if (isRunning) return;
		if (remainingMs <= 0) {
			remainingMs = durationFor(activePhase);
		}

		isRunning = true;
		phaseEndsAt = performance.now() + remainingMs;
		frameId = requestAnimationFrame(tick);
	};

	const stopTimer = () => {
		if (!isRunning) return;

		cancelFrame();
		remainingMs = Math.max(0, phaseEndsAt - performance.now());
		phaseEndsAt = 0;
		isRunning = false;
	};

	const applySettings = (nextWork: number, nextBreak: number) => {
		workMinutes = clampMinutes(nextWork);
		breakMinutes = clampMinutes(nextBreak);
		persistSettings();

		if (!isRunning) {
			resetTimer('work');
		}
	};

	const handleIntervalChange = (phase: Phase, event: Event) => {
		const input = event.currentTarget as HTMLInputElement;
		const nextValue = clampMinutes(input.valueAsNumber);

		if (phase === 'work') {
			applySettings(nextValue, breakMinutes);
			return;
		}

		applySettings(workMinutes, nextValue);
	};

	const applyPreset = (preset: Preset) => {
		if (isRunning) return;
		applySettings(preset.work, preset.break);
	};

	const formatTime = (value: number) => {
		const safeValue = Math.max(0, Math.floor(value));
		const totalSeconds = Math.floor(safeValue / 1000);
		const minutes = Math.floor(totalSeconds / 60);
		const seconds = totalSeconds % 60;
		const milliseconds = safeValue % 1000;

		return {
			main: `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`,
			ms: `.${milliseconds.toString().padStart(3, '0')}`
		};
	};

	const timeDisplay = $derived.by(() => formatTime(remainingMs));
	const pauseMs = $derived.by(() => (activePhase === 'break' ? remainingMs : durationFor('break')));
	const pauseDisplay = $derived.by(() => formatTime(pauseMs));
	const remainingRatio = $derived.by(() => {
		const currentDuration = durationFor(activePhase);
		if (currentDuration <= 0) return 0;
		return Math.min(1, Math.max(0, remainingMs / currentDuration));
	});
	const ringOffset = $derived(RING_CIRCUMFERENCE * (1 - remainingRatio));
	const ringColor = $derived.by(() => {
		if (remainingRatio > 0.66) return 'var(--success)';
		if (remainingRatio > 0.33) return 'var(--warning)';
		return 'var(--danger)';
	});

	onMount(() => {
		hasMounted = true;

		const storedSettings = localStorage.getItem(STORAGE_KEY);
		if (storedSettings) {
			try {
				const parsed = JSON.parse(storedSettings) as {
					workMinutes?: number;
					breakMinutes?: number;
				};

				workMinutes = clampMinutes(parsed.workMinutes ?? workMinutes);
				breakMinutes = clampMinutes(parsed.breakMinutes ?? breakMinutes);
				remainingMs = workMinutes * 60_000;
			} catch (error) {
				console.error('Could not restore pomodoro settings.', error);
			}
		}

		return () => {
			cancelFrame();
		};
	});
</script>

<svelte:head>
	<title>Pomodoro Timer</title>
	<meta name="theme-color" content="#0b0b0c" />
	<meta
		name="description"
		content="A minimal dark pomodoro timer with interval presets and millisecond precision."
	/>
</svelte:head>

<div class="page-shell">
	<main class="timer-stack">
		<div class="timer-frame" data-running={isRunning} style={`--ring-color:${ringColor};`}>
			<div class="timer-glow"></div>
			<svg class="progress-ring" viewBox="0 0 360 360" aria-hidden="true">
				<circle class="ring-track" cx="180" cy="180" r={RING_RADIUS}></circle>
				<circle
					class="ring-progress"
					cx="180"
					cy="180"
					r={RING_RADIUS}
					style={`stroke-dasharray:${RING_CIRCUMFERENCE}; stroke-dashoffset:${ringOffset};`}
				></circle>
			</svg>

			<div
				class="timer-readout"
			>
				<div
					class="timer-primary"
					role="timer"
					aria-label={`${activePhase === 'work' ? 'Work' : 'Break'} ${timeDisplay.main}${timeDisplay.ms}`}
				>
					<span class="sr-only">{activePhase === 'work' ? 'Work phase' : 'Break phase'}</span>
					<span class="timer-main">{timeDisplay.main}</span>
					<span class="timer-ms">{timeDisplay.ms}</span>
				</div>
				<div class="timer-secondary" aria-label="Pause timer">
					<span class="timer-secondary-label">pause</span>
					<span class="timer-secondary-value">
						{pauseDisplay.main}<span class="timer-secondary-ms">{pauseDisplay.ms}</span>
					</span>
				</div>
			</div>
		</div>

		<div class="controls">
			<div class="preset-row" aria-label="Interval presets">
				{#each presets as preset (preset.label)}
					<button
						type="button"
						class="control-button"
						data-active={workMinutes === preset.work && breakMinutes === preset.break}
						aria-pressed={workMinutes === preset.work && breakMinutes === preset.break}
						disabled={isRunning}
						onclick={() => applyPreset(preset)}
					>
						{preset.label}
					</button>
				{/each}
			</div>

			<div class="interval-row">
				<input
					type="number"
					class="interval-input"
					min="1"
					max="180"
					step="1"
					inputmode="numeric"
					aria-label="Work minutes"
					value={workMinutes}
					disabled={isRunning}
					onchange={(event) => handleIntervalChange('work', event)}
				/>
				<span class="interval-divider" aria-hidden="true">+</span>
				<input
					type="number"
					class="interval-input"
					min="1"
					max="180"
					step="1"
					inputmode="numeric"
					aria-label="Break minutes"
					value={breakMinutes}
					disabled={isRunning}
					onchange={(event) => handleIntervalChange('break', event)}
				/>
			</div>

			<div class="action-row">
				<button
					type="button"
					class="control-button control-button-primary"
					disabled={isRunning}
					onclick={startTimer}
				>
					Start
				</button>
				<button type="button" class="control-button" onclick={() => resetTimer()}>
					Reset
				</button>
				<button type="button" class="control-button" disabled={!isRunning} onclick={stopTimer}>
					Stop
				</button>
			</div>
		</div>
	</main>
</div>
