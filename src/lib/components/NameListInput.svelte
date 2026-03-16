<script lang="ts">
	let { names = $bindable<string[]>([]) } = $props();

	let singleName = $state('');
	let bulkText = $state('');
	let showBulk = $state(false);

	function addName() {
		const trimmed = singleName.trim();
		if (trimmed) {
			names = [...names, trimmed];
			singleName = '';
		}
	}

	function addBulkNames() {
		const newNames = bulkText
			.split('\n')
			.map((n) => n.trim())
			.filter((n) => n.length > 0);
		if (newNames.length > 0) {
			names = [...names, ...newNames];
			bulkText = '';
			showBulk = false;
		}
	}

	function removeName(index: number) {
		names = names.filter((_, i) => i !== index);
	}

	function handleKeydown(e: KeyboardEvent) {
		if (e.key === 'Enter') {
			e.preventDefault();
			addName();
		}
	}
</script>

<div class="flex h-full flex-col gap-4">
	<h2 class="text-xl font-bold text-purple-300">Plink Of Names</h2>

	<!-- Single name input -->
	<div class="flex gap-2">
		<input
			type="text"
			bind:value={singleName}
			onkeydown={handleKeydown}
			placeholder="Enter a name..."
			class="flex-1 rounded-lg border border-purple-500/30 bg-white/10 px-3 py-2 text-white placeholder-white/40 focus:border-purple-400 focus:outline-none focus:ring-1 focus:ring-purple-400"
		/>
		<button
			onclick={addName}
			disabled={!singleName.trim()}
			class="rounded-lg bg-purple-600 px-4 py-2 font-medium text-white transition-colors hover:bg-purple-500 disabled:opacity-40 disabled:hover:bg-purple-600"
		>
			Add
		</button>
	</div>

	<!-- Toggle bulk entry -->
	<button
		onclick={() => (showBulk = !showBulk)}
		class="text-left text-sm text-purple-400 transition-colors hover:text-purple-300"
	>
		{showBulk ? '− Hide bulk entry' : '+ Bulk entry (paste multiple names)'}
	</button>

	<!-- Bulk entry textarea -->
	{#if showBulk}
		<div class="flex flex-col gap-2">
			<textarea
				bind:value={bulkText}
				placeholder="Paste names here, one per line..."
				rows="5"
				class="rounded-lg border border-purple-500/30 bg-white/10 px-3 py-2 text-white placeholder-white/40 focus:border-purple-400 focus:outline-none focus:ring-1 focus:ring-purple-400"
			></textarea>
			<button
				onclick={addBulkNames}
				disabled={!bulkText.trim()}
				class="rounded-lg bg-emerald-600 px-4 py-2 font-medium text-white transition-colors hover:bg-emerald-500 disabled:opacity-40 disabled:hover:bg-emerald-600"
			>
				Add All
			</button>
		</div>
	{/if}

	<!-- Name list -->
	<div class="flex-1 overflow-y-auto">
		{#if names.length === 0}
			<p class="py-4 text-center text-sm text-white/40">No names added yet</p>
		{:else}
			<ul class="flex flex-col gap-1">
				{#each names as name, i}
					<li
						class="flex items-center justify-between rounded-lg bg-white/5 px-3 py-2 transition-colors hover:bg-white/10"
					>
						<span class="text-white">{name}</span>
						<button
							onclick={() => removeName(i)}
							class="text-red-400/60 transition-colors hover:text-red-400"
							aria-label="Remove {name}"
						>
							✕
						</button>
					</li>
				{/each}
			</ul>
		{/if}
	</div>

	<!-- Name count -->
	{#if names.length > 0}
		<div class="border-t border-white/10 pt-2 text-sm text-white/50">
			{names.length} name{names.length === 1 ? '' : 's'}
			{#if names.length < 2}
				<span class="text-amber-400"> — need at least 2 to drop</span>
			{/if}
		</div>
	{/if}
</div>
