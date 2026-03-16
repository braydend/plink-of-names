<script lang="ts">
	import { browser } from '$app/environment';
	import NameListInput from '$lib/components/NameListInput.svelte';
	import PlinkoBoard from '$lib/components/PlinkoBoard.svelte';

	const STORAGE_KEY = 'plink-of-names';

	interface SavedState {
		names: string[];
		pegDensity: number;
		boardScale: number;
		chaos: number;
	}

	function loadState(): Partial<SavedState> {
		if (!browser) return {};
		try {
			const raw = localStorage.getItem(STORAGE_KEY);
			if (raw) return JSON.parse(raw);
		} catch {
			// ignore corrupt data
		}
		return {};
	}

	function saveState(state: SavedState) {
		if (!browser) return;
		try {
			localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
		} catch {
			// ignore quota errors
		}
	}

	const saved = loadState();

	let names = $state<string[]>(saved.names ?? []);
	let selectedName = $state<string | null>(null);
	let dropping = $state(false);
	let board = $state<ReturnType<typeof PlinkoBoard>>();
	let pegDensity = $state(saved.pegDensity ?? 5);
	let boardScale = $state(saved.boardScale ?? 1);
	let chaos = $state(saved.chaos ?? 0);

	// Persist on changes
	$effect(() => {
		saveState({ names, pegDensity, boardScale, chaos });
	});

	function handleDrop() {
		if (!board) return;
		selectedName = null;
		dropping = true;
		board.dropBall();
	}

	function handleResult(name: string) {
		selectedName = name;
		dropping = false;
	}

	function handleDropAgain() {
		if (!board) return;
		board.resetBall();
		selectedName = null;
		dropping = false;
	}

	let canDrop = $derived(names.length >= 2 && !dropping && !selectedName);
</script>

<div class="flex h-screen bg-gradient-to-br from-slate-900 via-purple-950 to-slate-900">
	<!-- Sidebar: Name Input -->
	<aside class="flex w-72 shrink-0 flex-col border-r border-white/10 p-4">
		<div class="min-h-0 flex-1">
			<NameListInput bind:names />
		</div>

		<!-- Settings -->
		<div class="flex flex-col gap-3 border-t border-white/10 pt-3">
			<div>
				<div class="mb-1 flex items-baseline justify-between">
					<label for="peg-density" class="text-sm font-medium text-purple-300">
						Peg Density
					</label>
					<span class="text-base font-bold text-white">{pegDensity}</span>
				</div>
				<input
					id="peg-density"
					type="range"
					min="1"
					max="10"
					step="1"
					bind:value={pegDensity}
					class="h-2 w-full cursor-pointer appearance-none rounded-lg bg-white/10 accent-purple-500"
				/>
				<div class="mt-1 flex justify-between text-xs text-white/30">
					<span>Sparse</span>
					<span>Dense</span>
				</div>
			</div>

			<div>
				<div class="mb-1 flex items-baseline justify-between">
					<label for="board-scale" class="text-sm font-medium text-purple-300">
						Board Height
					</label>
					<span class="text-base font-bold text-white">{boardScale}x</span>
				</div>
				<input
					id="board-scale"
					type="range"
					min="1"
					max="3"
					step="0.25"
					bind:value={boardScale}
					class="h-2 w-full cursor-pointer appearance-none rounded-lg bg-white/10 accent-purple-500"
				/>
				<div class="mt-1 flex justify-between text-xs text-white/30">
					<span>Fit</span>
					<span>Tall</span>
				</div>
			</div>

			<div>
				<div class="mb-1 flex items-baseline justify-between">
					<label for="chaos" class="text-sm font-medium text-purple-300">
						Chaos
					</label>
					<span class="text-base font-bold text-white">{chaos}</span>
				</div>
				<input
					id="chaos"
					type="range"
					min="0"
					max="5"
					step="1"
					bind:value={chaos}
					class="h-2 w-full cursor-pointer appearance-none rounded-lg bg-white/10 accent-purple-500"
				/>
				<div class="mt-1 flex justify-between text-xs text-white/30">
					<span>Calm</span>
					<span>Wild</span>
				</div>
			</div>

			<button
				onclick={() => board?.randomise()}
				disabled={names.length < 2 || dropping}
				class="rounded-lg bg-purple-600 px-4 py-2 text-sm font-medium text-white transition-colors hover:bg-purple-500 disabled:opacity-40 disabled:hover:bg-purple-600"
			>
				Randomise
			</button>
		</div>
	</aside>

	<!-- Main: Plinko Board -->
	<main class="flex min-w-0 flex-1 flex-col overflow-hidden">
		<!-- Top bar: Drop button + result -->
		{#if names.length >= 2}
			<div class="flex shrink-0 items-center justify-center gap-6 px-4 py-3">
				{#if selectedName}
					<div
						class="rounded-xl border border-amber-400/30 bg-amber-400/10 px-6 py-2 text-center"
					>
						<p class="text-sm font-medium text-amber-300/70">Selected</p>
						<p class="text-2xl font-bold text-amber-300">{selectedName}</p>
					</div>
					<button
						onclick={handleDropAgain}
						class="rounded-xl bg-gradient-to-r from-emerald-500 to-cyan-500 px-8 py-3 text-lg font-bold text-white shadow-lg transition-all hover:scale-105 hover:shadow-emerald-500/25"
					>
						Drop Again
					</button>
				{:else}
					<button
						onclick={handleDrop}
						disabled={!canDrop}
						class="rounded-xl bg-gradient-to-r from-pink-500 to-purple-500 px-8 py-3 text-lg font-bold text-white shadow-lg transition-all hover:scale-105 hover:shadow-pink-500/25 disabled:scale-100 disabled:opacity-50 disabled:shadow-none"
					>
						{dropping ? 'Dropping...' : 'Drop!'}
					</button>
				{/if}
			</div>
		{/if}

		<!-- Board fills remaining space -->
		<div class="relative min-h-0 flex-1">
			<div class="absolute inset-0">
				<PlinkoBoard bind:this={board} {names} {pegDensity} {boardScale} {chaos} onresult={handleResult} />
			</div>
		</div>
	</main>
</div>
