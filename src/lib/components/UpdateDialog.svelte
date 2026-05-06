<script lang="ts">
	import { fade, scale } from 'svelte/transition';
	import { updateStore } from '../stores/update.svelte.js';

	let dialogEl = $state<HTMLDivElement>();
	let previousActiveElement: HTMLElement | null = null;

	// Capture / restore focus when the dialog opens and closes. The actual
	// focus-on-open handoff happens in `introend` below, after the scale
	// transition finishes — that avoids a 50 ms timer racing the 200 ms
	// transition on slow machines or throttled tabs.
	$effect(() => {
		if (updateStore.show) {
			previousActiveElement = document.activeElement as HTMLElement | null;
		} else if (previousActiveElement) {
			previousActiveElement.focus();
			previousActiveElement = null;
		}
	});

	function close() {
		updateStore.close();
	}

	function retry() {
		// Retry always re-runs the check, even after a download or install
		// error. Re-checking is the safer default — the latest.json may have
		// been updated to point at a fixed binary in the meantime.
		updateStore.runCheck();
	}

	function startDownload() {
		updateStore.startDownload();
	}

	function handleKeydown(e: KeyboardEvent) {
		// Tab focus trap — keep keyboard focus inside the dialog while it's
		// open, matching the existing Settings.svelte modal pattern. Without
		// this, Tab can move focus to elements behind the backdrop. Tab is
		// trapped in every phase including `downloading` — even when the
		// only footer control is the disabled Cancel button, we don't want
		// focus to escape.
		if (e.key === 'Tab') {
			const focusable = dialogEl?.querySelectorAll<HTMLElement>(
				'button:not([disabled]), [href], input, select, textarea, summary, [tabindex]:not([tabindex="-1"])',
			);
			if (!focusable || focusable.length === 0) {
				// No interactive children (e.g. only a disabled Cancel during
				// downloading) — keep focus on the dialog container instead
				// of letting it escape to the underlying UI.
				e.preventDefault();
				dialogEl?.focus();
				return;
			}
			const first = focusable[0];
			const last = focusable[focusable.length - 1];
			if (e.shiftKey && document.activeElement === first) {
				e.preventDefault();
				last.focus();
			} else if (!e.shiftKey && document.activeElement === last) {
				e.preventDefault();
				first.focus();
			}
			return;
		}

		// Phase-specific keys — Escape closes (except during downloading,
		// which would tear down an in-flight install), and Enter activates
		// the visible primary action when focus is on the dialog container,
		// not on a button or summary (those should handle Enter natively).
		if (updateStore.phase === 'downloading') return;

		if (e.key === 'Escape') {
			e.preventDefault();
			close();
			return;
		}

		if (
			e.key === 'Enter' &&
			!(e.target instanceof HTMLButtonElement) &&
			!(e.target instanceof HTMLElement && e.target.tagName === 'SUMMARY')
		) {
			e.preventDefault();
			if (updateStore.phase === 'available') startDownload();
			else if (updateStore.phase === 'error') retry();
			else if (updateStore.phase === 'up-to-date') close();
		}
	}

	function handleBackdrop() {
		if (updateStore.phase !== 'downloading') close();
	}

	function handleIntroEnd() {
		dialogEl?.focus();
	}

	function fmtMB(bytes: number) {
		return (bytes / (1024 * 1024)).toFixed(1);
	}

	let progressPct = $derived(
		updateStore.total > 0 ? Math.min(100, (updateStore.downloaded / updateStore.total) * 100) : 0,
	);

	let errorHeading = $derived(
		updateStore.errorSource === 'download'
			? 'Update download failed'
			: updateStore.errorSource === 'install'
				? 'Restart failed'
				: 'Update check failed',
	);

	let errorBodyLead = $derived(
		updateStore.errorSource === 'download'
			? 'Could not download or install the update.'
			: updateStore.errorSource === 'install'
				? 'The update was downloaded but Markpad could not restart automatically. Please quit and reopen Markpad to finish installing.'
				: 'Could not check for updates.',
	);
</script>

{#if updateStore.show}
	<div
		class="update-backdrop"
		transition:fade={{ duration: 150 }}
		onclick={handleBackdrop}
		role="presentation">
		<div
			class="update-content"
			bind:this={dialogEl}
			transition:scale={{ duration: 200, start: 0.95 }}
			onintroend={handleIntroEnd}
			onclick={(e) => e.stopPropagation()}
			role="dialog"
			aria-modal="true"
			aria-labelledby="update-title"
			aria-describedby="update-body"
			tabindex="-1"
			onkeydown={handleKeydown}>
			<div class="update-header">
				<h3 id="update-title">
					{#if updateStore.phase === 'checking'}
						Checking for updates…
					{:else if updateStore.phase === 'up-to-date'}
						You're up to date
					{:else if updateStore.phase === 'available'}
						Update available
					{:else if updateStore.phase === 'downloading'}
						Downloading update…
					{:else if updateStore.phase === 'error'}
						{errorHeading}
					{:else}
						Markpad
					{/if}
				</h3>
			</div>

			<div class="update-body" id="update-body">
				{#if updateStore.phase === 'checking'}
					<div class="centered-row">
						<span class="spinner" aria-hidden="true"></span>
						<p>Looking for the latest version of Markpad…</p>
					</div>
				{:else if updateStore.phase === 'up-to-date'}
					<p>
						You're using the latest version of Markpad
						{#if updateStore.current}(v{updateStore.current}){/if}.
					</p>
				{:else if updateStore.phase === 'available'}
					<p class="lead">
						Markpad <strong>v{updateStore.latest}</strong> is available.
						You're on v{updateStore.current}.
					</p>
					{#if updateStore.notes}
						<details class="notes">
							<summary>Release notes</summary>
							<pre>{updateStore.notes}</pre>
						</details>
					{/if}
				{:else if updateStore.phase === 'downloading'}
					<p class="lead">Downloading Markpad v{updateStore.latest}…</p>
					{#if updateStore.total > 0}
						<progress max={updateStore.total} value={updateStore.downloaded}></progress>
					{:else}
						<progress></progress>
					{/if}
					<p class="progress-text">
						{#if updateStore.total > 0}
							{fmtMB(updateStore.downloaded)} MB of {fmtMB(updateStore.total)} MB ({progressPct.toFixed(0)}%)
						{:else}
							{fmtMB(updateStore.downloaded)} MB downloaded
						{/if}
					</p>
					<p class="hint">Markpad will restart automatically when the update is ready.</p>
				{:else if updateStore.phase === 'error'}
					<p>{errorBodyLead}</p>
					<pre class="error-detail">{updateStore.errorMsg}</pre>
				{/if}
			</div>

			<div class="update-footer">
				{#if updateStore.phase === 'checking' || updateStore.phase === 'downloading'}
					<button
						class="btn secondary"
						onclick={close}
						disabled={updateStore.phase === 'downloading'}>
						Cancel
					</button>
				{:else if updateStore.phase === 'up-to-date'}
					<button class="btn primary" onclick={close}>OK</button>
				{:else if updateStore.phase === 'available'}
					<button class="btn secondary" onclick={close}>Cancel</button>
					<button class="btn primary" onclick={startDownload}>Download &amp; Install</button>
				{:else if updateStore.phase === 'error'}
					<button class="btn secondary" onclick={close}>Close</button>
					<button class="btn primary" onclick={retry}>Retry</button>
				{/if}
			</div>
		</div>
	</div>
{/if}

<style>
	.update-backdrop {
		position: fixed;
		inset: 0;
		background: rgba(0, 0, 0, 0.4);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 30000;
	}

	.update-content {
		background: var(--color-canvas-default);
		border: 1px solid var(--color-border-default);
		border-radius: 6px;
		width: 460px;
		max-width: 92vw;
		box-shadow: 0 20px 50px rgba(0, 0, 0, 0.3);
		overflow: hidden;
		font-family: var(--win-font);
	}

	.update-header {
		padding: 20px 24px 12px 24px;
	}

	.update-header h3 {
		margin: 0;
		font-size: 16px;
		font-weight: 600;
		color: var(--color-fg-default);
	}

	.update-body {
		padding: 0 24px 20px 24px;
		font-size: 14px;
		line-height: 1.5;
		color: var(--color-fg-muted);
	}

	.update-body p {
		margin: 0 0 8px 0;
	}

	.update-body p.lead {
		color: var(--color-fg-default);
	}

	.update-body p.hint {
		font-size: 12px;
		color: var(--color-fg-muted);
		margin-top: 8px;
	}

	.update-body p.progress-text {
		font-size: 12px;
		color: var(--color-fg-muted);
		font-variant-numeric: tabular-nums;
	}

	progress {
		width: 100%;
		height: 8px;
		margin: 12px 0 4px 0;
		appearance: none;
	}
	progress::-webkit-progress-bar {
		background: var(--color-neutral-muted);
		border-radius: 4px;
	}
	progress::-webkit-progress-value {
		background: var(--color-accent-fg);
		border-radius: 4px;
		transition: width 0.1s linear;
	}

	.notes {
		margin-top: 12px;
		font-size: 13px;
	}
	.notes summary {
		cursor: pointer;
		color: var(--color-fg-default);
	}
	.notes pre {
		margin: 8px 0 0 0;
		padding: 12px;
		background: var(--color-canvas-subtle);
		border: 1px solid var(--color-border-muted);
		border-radius: 6px;
		max-height: 180px;
		overflow: auto;
		white-space: pre-wrap;
		word-break: break-word;
		font-family: var(--win-font);
		font-size: 13px;
		color: var(--color-fg-muted);
	}

	.error-detail {
		margin: 8px 0 0 0;
		padding: 8px 10px;
		background: var(--color-canvas-subtle);
		border: 1px solid var(--color-border-muted);
		border-radius: 4px;
		font-size: 12px;
		font-family: var(--win-font);
		white-space: pre-wrap;
		word-break: break-word;
		color: var(--color-fg-muted);
	}

	.centered-row {
		display: flex;
		align-items: center;
		gap: 12px;
	}

	.spinner {
		width: 16px;
		height: 16px;
		border: 2px solid var(--color-neutral-muted);
		border-top-color: var(--color-accent-fg);
		border-radius: 50%;
		animation: spin 0.8s linear infinite;
		display: inline-block;
		flex-shrink: 0;
	}

	@keyframes spin {
		to {
			transform: rotate(360deg);
		}
	}

	.update-footer {
		padding: 16px 24px;
		background: var(--color-canvas-subtle);
		display: flex;
		align-items: center;
		justify-content: flex-end;
		gap: 8px;
		border-top: 1px solid var(--color-border-muted);
	}

	.btn {
		padding: 6px 16px;
		border-radius: 6px;
		font-size: 14px;
		font-weight: 500;
		cursor: pointer;
		transition: all 0.1s;
		border: 1px solid transparent;
		font-family: inherit;
	}

	.btn.secondary {
		background: transparent;
		color: var(--color-fg-default);
		border-color: var(--color-border-default);
	}
	.btn.secondary:hover:not(:disabled) {
		background: var(--color-neutral-muted);
	}

	.btn.primary {
		background: var(--color-accent-fg);
		color: var(--color-btn-fg);
	}
	.btn.primary:hover:not(:disabled) {
		filter: brightness(1.1);
	}

	.btn:disabled {
		opacity: 0.5;
		cursor: not-allowed;
	}
</style>
