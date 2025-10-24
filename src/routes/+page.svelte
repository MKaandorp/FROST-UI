<script lang="ts">
	export const ssr = false;

	import { onMount } from 'svelte';

	type EntitySet = {
		name: string;
		url: string;
		description?: string;
	};

	type Breadcrumb = {
		label: string;
		url: string;
	};

	type CollectionView = {
		kind: 'collection';
		items: Record<string, unknown>[];
		nextLink: string | null;
		raw: Record<string, unknown>;
	};

	type EntityView = {
		kind: 'entity';
		entity: Record<string, unknown>;
	};

	type View = CollectionView | EntityView;

	const DEFAULT_SERVER = 'https://ogc-demo.k8s.ilt-dmz.iosb.fraunhofer.de/v1.1/';

	let serverUrl = DEFAULT_SERVER;
	let username = '';
	let password = '';

	let rootLoading = false;
	let rootError: string | null = null;
	let entitySets: EntitySet[] = [];

	let activeServer = '';
	let navigationStack: Breadcrumb[] = [];
	let currentView: View | null = null;
	let dataLoading = false;
	let dataError: string | null = null;

	onMount(() => {
		// Try to connect to the default server once the component is mounted.
		connect().catch(() => {
			/* handled through rootError */
		});
	});

	function ensureBase(url: string) {
		const trimmed = url.trim();
		if (!trimmed) {
			return '';
		}
		return trimmed.endsWith('/') ? trimmed : `${trimmed}/`;
	}

	function resolveUrl(url: string) {
		if (!url) {
			return url;
		}
		const base = activeServer || ensureBase(serverUrl);
		try {
			return new URL(url, base).toString();
		} catch {
			return url;
		}
	}

	function buildHeaders() {
		const headers = new Headers({ Accept: 'application/json' });
		if (username || password) {
			const credentials = `${username}:${password}`;
			try {
				headers.set('Authorization', `Basic ${btoa(credentials)}`);
			} catch {
				headers.set(
					'Authorization',
					`Basic ${btoa(unescape(encodeURIComponent(credentials)))}`
				);
			}
		}
		return headers;
	}

	async function fetchJson(url: string) {
		const response = await fetch(url, { headers: buildHeaders() });
		if (!response.ok) {
			const text = await response.text().catch(() => '');
			const message = text ? ` - ${text}` : '';
			throw new Error(`Request failed (${response.status} ${response.statusText})${message}`);
		}
		return response.json();
	}

	async function connect() {
		rootError = null;
		rootLoading = true;
		dataError = null;
		currentView = null;
		navigationStack = [];

		const base = ensureBase(serverUrl || DEFAULT_SERVER);
		try {
			const json = await fetchJson(base);
			const rawSets = Array.isArray(json?.value) ? json.value : [];
			if (!rawSets.length) {
				throw new Error('No entity sets returned. Check the server URL.');
			}
			activeServer = base;
			entitySets = rawSets
				.map((entry: Record<string, unknown>) => {
					const name = typeof entry.name === 'string' ? entry.name : '';
					const rawUrl =
						typeof entry.url === 'string'
							? entry.url
							: typeof entry.href === 'string'
								? entry.href
								: null;
					const url =
						typeof rawUrl === 'string'
							? rawUrl
							: name
								? `${base}${name}`
								: '';
					const description =
						typeof entry.description === 'string' ? entry.description : undefined;
					if (!name || !url) {
						return null;
					}
					return {
						name,
						url: resolveUrl(url),
						description
					};
				})
				.filter(Boolean) as EntitySet[];
			if (!entitySets.length) {
				throw new Error('Server returned an empty entity list.');
			}
		} catch (error) {
			rootError = error instanceof Error ? error.message : 'Unable to reach the server.';
			entitySets = [];
			activeServer = base;
		} finally {
			rootLoading = false;
		}
	}

	async function fetchView(url: string): Promise<View> {
		const resolved = resolveUrl(url);
		const json = await fetchJson(resolved);
		if (Array.isArray(json?.value)) {
			return {
				kind: 'collection',
				items: json.value as Record<string, unknown>[],
				nextLink:
					typeof json['@iot.nextLink'] === 'string'
						? resolveUrl(json['@iot.nextLink'] as string)
						: null,
				raw: json as Record<string, unknown>
			};
		}
		return {
			kind: 'entity',
			entity: json as Record<string, unknown>
		};
	}

	async function navigateTo(params: { url: string; label: string; reset?: boolean; replaceFrom?: number }) {
		const { url, label, reset = false, replaceFrom } = params;
		dataError = null;
		dataLoading = true;
		try {
			const view = await fetchView(url);
			currentView = view;
			const targetUrl = resolveUrl(url);
			if (reset) {
				navigationStack = [{ label, url: targetUrl }];
			} else if (typeof replaceFrom === 'number') {
				navigationStack = [...navigationStack.slice(0, replaceFrom), { label, url: targetUrl }];
			} else {
				navigationStack = [...navigationStack, { label, url: targetUrl }];
			}
		} catch (error) {
			dataError = error instanceof Error ? error.message : 'Failed to load data.';
		} finally {
			dataLoading = false;
		}
	}

	function openEntitySet(set: EntitySet) {
		navigateTo({ url: set.url, label: set.name, reset: true });
	}

	function openNavigationLink(link: string, label: string) {
		if (!link) return;
		navigateTo({ url: link, label });
	}

	function loadNextPage(link: string) {
		if (!link || !navigationStack.length) return;
		const last = navigationStack[navigationStack.length - 1];
		navigateTo({ url: link, label: last.label, replaceFrom: navigationStack.length - 1 });
	}

	function revisitCrumb(index: number) {
		const crumb = navigationStack[index];
		if (!crumb || index === navigationStack.length - 1) {
			return;
		}
		navigateTo({ url: crumb.url, label: crumb.label, replaceFrom: index });
	}

	function navLabelFromKey(key: string) {
		return (
			key
				.replace('@iot.navigationLink', '')
				.replace(/([a-z0-9])([A-Z])/g, '$1 $2')
				.replace(/_/g, ' ')
				.trim() || 'Related'
		);
	}

	function isComplex(value: unknown): value is Record<string, unknown> | unknown[] {
		return typeof value === 'object' && value !== null;
	}

	function formatPrimitive(value: unknown) {
		if (value === null || value === undefined) return '—';
		if (typeof value === 'boolean') return value ? 'true' : 'false';
		return String(value);
	}

	function itemTitle(item: Record<string, unknown>, fallback: string) {
		if (typeof item.name === 'string' && item.name.trim()) return item.name;
		if (typeof item['@iot.id'] === 'number' || typeof item['@iot.id'] === 'string') {
			return `@iot.id ${item['@iot.id']}`;
		}
		if (typeof item.description === 'string' && item.description.trim()) return item.description;
		return fallback;
	}
</script>

<div class="page">
	<section class="sidebar">
		<h1>FROST UI Explorer</h1>
		<form class="connection" on:submit|preventDefault={connect}>
			<label>
				<span>Server URL</span>
				<input
					type="url"
					placeholder="https://example.com/v1.1/"
					bind:value={serverUrl}
					required
				/>
			</label>
			<label>
				<span>Username</span>
				<input type="text" autocomplete="username" bind:value={username} />
			</label>
			<label>
				<span>Password</span>
				<input type="password" autocomplete="current-password" bind:value={password} />
			</label>
			<button type="submit" class="primary" disabled={rootLoading}>
				{rootLoading ? 'Connecting…' : 'Connect'}
			</button>
		</form>

		{#if rootError}
			<p class="error">{rootError}</p>
		{/if}

		<section class="entities">
			<header>
				<h2>Entity Sets</h2>
				{#if activeServer}
					<small>{activeServer}</small>
				{:else}
					<small>Connect to a FROST server</small>
				{/if}
			</header>

			{#if rootLoading}
				<p class="status">Loading entity sets…</p>
			{:else if entitySets.length}
				<div class="entity-grid">
					{#each entitySets as set (set.url)}
						<button
							type="button"
							class="entity-card"
							class:active={navigationStack[0]?.url === set.url}
							on:click={() => openEntitySet(set)}
						>
							<strong>{set.name}</strong>
							{#if set.description}
								<p>{set.description}</p>
							{/if}
							<code>{set.url}</code>
						</button>
					{/each}
				</div>
			{:else if !rootError}
				<p class="status">Connect to load available entities.</p>
			{/if}
		</section>
	</section>

	<section class="content">
		{#if navigationStack.length}
			<nav class="breadcrumbs" aria-label="Navigation trail">
				{#each navigationStack as crumb, index (crumb.url)}
					<button
						type="button"
						on:click={() => revisitCrumb(index)}
						disabled={index === navigationStack.length - 1}
					>
						{crumb.label}
					</button>
					{#if index < navigationStack.length - 1}
						<span class="separator">›</span>
					{/if}
				{/each}
			</nav>
		{/if}

		{#if dataError}
			<p class="error">{dataError}</p>
		{/if}

		{#if dataLoading && !currentView}
			<div class="placeholder">
				<p>Loading data…</p>
			</div>
		{:else if currentView}
			{#if currentView.kind === 'collection'}
				<section class="collection">
					<header class="collection-header">
						<h2>{navigationStack[navigationStack.length - 1]?.label ?? 'Collection'}</h2>
						{#if typeof currentView.raw['@iot.count'] === 'number'}
							<span>{currentView.raw['@iot.count']} items reported</span>
						{/if}
					</header>

					{#if currentView.items.length === 0}
						<p class="status">No entities found for this set.</p>
					{:else}
						<div class="item-list">
							{#each currentView.items as item, index (item['@iot.id'] ?? index)}
								<details class="item-card" open={index === 0}>
									<summary>
										{itemTitle(item, `Item ${index + 1}`)}
									</summary>
									<div class="field-grid">
										{#each Object.entries(item) as [key, value]}
											{#if key.endsWith('@iot.navigationLink') && typeof value === 'string'}
												<div class="field action">
													<span class="field-key">{navLabelFromKey(key)}</span>
													<button type="button" on:click={() => openNavigationLink(value, navLabelFromKey(key))}>
														Explore
													</button>
												</div>
											{:else if key === '@iot.selfLink' && typeof value === 'string'}
												<div class="field link">
													<span class="field-key">@iot.selfLink</span>
													<div class="field-value">
														<a href={value} target="_blank" rel="noreferrer">{value}</a>
														<button type="button" on:click={() => openNavigationLink(value, 'Details')}>
															Open here
														</button>
													</div>
												</div>
											{:else}
												<div class="field">
													<span class="field-key">{key}</span>
													{#if isComplex(value)}
														<pre>{JSON.stringify(value, null, 2)}</pre>
													{:else}
														<span class="field-value">{formatPrimitive(value)}</span>
													{/if}
												</div>
											{/if}
										{/each}
									</div>
								</details>
							{/each}
						</div>
					{/if}

					{#if currentView.nextLink}
						<div class="pagination">
							<button type="button" on:click={() => loadNextPage(currentView.nextLink)}>
								Load next page
							</button>
						</div>
					{/if}
				</section>
			{:else}
				<section class="entity-detail">
					<header>
						<h2>{itemTitle(currentView.entity, 'Entity')}</h2>
					</header>
					<div class="field-grid">
						{#each Object.entries(currentView.entity) as [key, value]}
							{#if key.endsWith('@iot.navigationLink') && typeof value === 'string'}
								<div class="field action">
									<span class="field-key">{navLabelFromKey(key)}</span>
									<button type="button" on:click={() => openNavigationLink(value, navLabelFromKey(key))}>
										Explore
									</button>
								</div>
							{:else if key === '@iot.selfLink' && typeof value === 'string'}
								<div class="field link">
									<span class="field-key">@iot.selfLink</span>
									<div class="field-value">
										<a href={value} target="_blank" rel="noreferrer">{value}</a>
									</div>
								</div>
							{:else}
								<div class="field">
									<span class="field-key">{key}</span>
									{#if isComplex(value)}
										<pre>{JSON.stringify(value, null, 2)}</pre>
									{:else}
										<span class="field-value">{formatPrimitive(value)}</span>
									{/if}
								</div>
							{/if}
						{/each}
					</div>
				</section>
			{/if}
		{:else}
			<div class="placeholder">
				<p>Select an entity set to start exploring.</p>
			</div>
		{/if}
	</section>
</div>

<style>
	:global(body) {
		background: #f6f8fb;
		color: #1e2430;
		font-family: 'Inter', system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
	}

	.page {
		display: flex;
		flex-wrap: wrap;
		gap: 1.5rem;
		align-items: flex-start;
		padding: 2rem 0;
	}

	.sidebar {
		flex: 0 0 320px;
		max-width: 360px;
		background: #fff;
		padding: 1.5rem;
		border-radius: 12px;
		box-shadow: 0 10px 30px rgba(15, 23, 42, 0.08);
	}

	h1 {
		margin: 0 0 1rem;
		font-size: 1.5rem;
		font-weight: 600;
	}

	h2 {
		margin: 0;
		font-size: 1rem;
		font-weight: 600;
		text-transform: uppercase;
		letter-spacing: 0.08em;
	}

	.connection {
		display: grid;
		gap: 0.9rem;
		margin-bottom: 1.5rem;
	}

	label {
		display: grid;
		gap: 0.35rem;
		font-size: 0.85rem;
		font-weight: 500;
	}

	input {
		border: 1px solid #cbd5f5;
		border-radius: 8px;
		padding: 0.55rem 0.75rem;
		font-size: 0.95rem;
		transition: border-color 150ms ease, box-shadow 150ms ease;
	}

	input:focus {
		outline: none;
		border-color: #4c6ef5;
		box-shadow: 0 0 0 3px rgba(76, 110, 245, 0.2);
	}

	button {
		cursor: pointer;
	}

	button.primary {
		background: #4c6ef5;
		color: #fff;
		border: none;
		padding: 0.6rem 1rem;
		font-weight: 600;
		border-radius: 8px;
		transition: background 150ms ease, transform 150ms ease;
	}

	button.primary:disabled {
		background: #8ea3ff;
		cursor: wait;
	}

	button.primary:not(:disabled):hover {
		background: #3b5bdb;
		transform: translateY(-1px);
	}

	.entities header {
		display: flex;
		flex-direction: column;
		gap: 0.25rem;
		margin-bottom: 0.75rem;
	}

	.entities small {
		color: #6b7280;
		font-size: 0.75rem;
	}

	.entity-grid {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
		gap: 0.75rem;
	}

	.entity-card {
		display: flex;
		flex-direction: column;
		align-items: flex-start;
		gap: 0.35rem;
		padding: 0.9rem;
		border-radius: 10px;
		border: 1px solid #e2e8f0;
		background: linear-gradient(180deg, #fff, #f8faff);
		text-align: left;
		transition: border-color 150ms ease, box-shadow 150ms ease, transform 150ms ease;
		color: inherit;
	}

	.entity-card strong {
		font-size: 0.95rem;
	}

	.entity-card p {
		margin: 0;
		font-size: 0.82rem;
		color: #4b5563;
	}

	.entity-card code {
		font-size: 0.7rem;
		color: #6366f1;
		background: rgba(99, 102, 241, 0.08);
		border-radius: 6px;
		padding: 0.15rem 0.35rem;
	}

	.entity-card:hover {
		border-color: #6366f1;
		box-shadow: 0 8px 20px rgba(99, 102, 241, 0.18);
		transform: translateY(-2px);
	}

	.entity-card.active {
		border-color: #4c6ef5;
		box-shadow: 0 0 0 2px rgba(76, 110, 245, 0.35);
	}

	.content {
		flex: 1 1 620px;
		min-width: min(640px, 100%);
		background: #fff;
		padding: 1.5rem;
		border-radius: 12px;
		box-shadow: 0 12px 40px rgba(15, 23, 42, 0.08);
		display: grid;
		gap: 1.25rem;
	}

	.breadcrumbs {
		display: flex;
		flex-wrap: wrap;
		align-items: center;
		gap: 0.5rem;
	}

	.breadcrumbs button {
		border: 1px solid #d0d4f0;
		background: #f6f8ff;
		color: #374151;
		border-radius: 999px;
		padding: 0.35rem 0.75rem;
		font-size: 0.75rem;
		transition: background 150ms ease, border-color 150ms ease;
	}

	.breadcrumbs button:disabled {
		border-color: #6366f1;
		background: rgba(99, 102, 241, 0.15);
		color: #312e81;
		cursor: default;
	}

	.breadcrumbs button:not(:disabled):hover {
		background: rgba(99, 102, 241, 0.12);
		border-color: #818cf8;
	}

	.separator {
		color: #9ca3af;
	}

	.collection-header {
		display: flex;
		flex-wrap: wrap;
		align-items: baseline;
		justify-content: space-between;
		gap: 0.75rem;
	}

	.collection-header span {
		font-size: 0.85rem;
		color: #6b7280;
	}

	.item-list {
		display: grid;
		gap: 0.9rem;
	}

	.item-card {
		border: 1px solid #e5e7eb;
		border-radius: 10px;
		padding: 0.75rem 1rem;
		background: linear-gradient(180deg, #ffffff 0%, #f9fbff 100%);
		box-shadow: 0 4px 12px rgba(15, 23, 42, 0.06);
	}

	.item-card summary {
		cursor: pointer;
		font-weight: 600;
		color: #1f2937;
	}

	.field-grid {
		display: grid;
		gap: 0.75rem;
		margin-top: 0.85rem;
		grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
	}

	.field {
		display: flex;
		flex-direction: column;
		gap: 0.35rem;
		font-size: 0.85rem;
		background: #f8fafc;
		padding: 0.65rem;
		border-radius: 8px;
		border: 1px solid #e2e8f0;
	}

	.field.action {
		background: #eef2ff;
		border-color: #c7d2fe;
	}

	.field.link {
		background: #f1f5f9;
		border-color: #d8dee9;
	}

	.field-key {
		font-size: 0.75rem;
		font-weight: 600;
		color: #475569;
		text-transform: uppercase;
		letter-spacing: 0.06em;
	}

	.field-value {
		font-family: 'Source Code Pro', 'Fira Code', monospace;
		word-break: break-word;
		font-size: 0.8rem;
	}

	.field.action button {
		align-self: start;
		border: none;
		background: #4c6ef5;
		color: #fff;
		padding: 0.4rem 0.7rem;
		border-radius: 6px;
		font-size: 0.75rem;
		font-weight: 600;
		transition: background 150ms ease, transform 150ms ease;
	}

	.field.action button:hover {
		background: #3b5bdb;
		transform: translateY(-1px);
	}

	.field.link a {
		color: #2563eb;
		text-decoration: none;
		font-size: 0.8rem;
	}

	.field.link a:hover {
		text-decoration: underline;
	}

	pre {
		margin: 0;
		padding: 0.5rem;
		border-radius: 6px;
		background: #0f172a;
		color: #e2e8f0;
		font-size: 0.75rem;
		overflow-x: auto;
	}

	.entity-detail header {
		display: flex;
		align-items: baseline;
		justify-content: space-between;
	}

	.pagination {
		display: flex;
		justify-content: flex-end;
	}

	.pagination button {
		border: 1px solid #4c6ef5;
		color: #fff;
		background: #4c6ef5;
		padding: 0.5rem 0.9rem;
		border-radius: 6px;
		font-weight: 600;
		transition: background 150ms ease, transform 150ms ease;
	}

	.pagination button:hover {
		background: #3b5bdb;
		transform: translateY(-1px);
	}

	.placeholder {
		border: 2px dashed #cbd5f5;
		padding: 2rem;
		border-radius: 12px;
		text-align: center;
		color: #6b7280;
	}

	.error {
		background: rgba(239, 68, 68, 0.12);
		border: 1px solid rgba(239, 68, 68, 0.3);
		color: #b91c1c;
		padding: 0.75rem;
		border-radius: 8px;
		font-size: 0.85rem;
	}

	.status {
		font-size: 0.85rem;
		color: #6b7280;
	}

	@media (max-width: 960px) {
		.sidebar,
		.content {
			width: 100%;
			max-width: 100%;
		}

		.content {
			padding: 1rem;
		}
	}
</style>
