<script lang="ts">
	import { getContext, onMount } from 'svelte';
	import { toast } from 'svelte-sonner';
	import { goto } from '$app/navigation';

	import { config, settings, user } from '$lib/stores';
	import { getBackendConfig } from '$lib/apis';
	import { getAdminConfig, updateAdminConfig } from '$lib/apis/auths';
	import { getOllamaConfig, updateOllamaConfig } from '$lib/apis/ollama';
	import { getOpenAIConfig, updateOpenAIConfig } from '$lib/apis/openai';
	import { getRAGConfig, updateRAGConfig } from '$lib/apis/retrieval';
	import { updateUserSettings } from '$lib/apis/users';

	import Switch from '$lib/components/common/Switch.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';

	const i18n = getContext('i18n');

	type Step =
		| 'welcome'
		| 'connections'
		| 'web_search'
		| 'code_execution'
		| 'capabilities'
		| 'done';

	let step: Step = 'welcome';
	let saving = false;
	let loaded = false;

	// Step state
	let ollamaEnabled = false;
	let ollamaUrl = 'http://localhost:11434';
	let openaiEnabled = false;
	let openaiKey = '';
	let openaiUrl = 'https://api.openai.com/v1';

	let webSearchEnabled = false;
	let webSearchEngine = 'duckduckgo';
	let bypassEmbedding = true;

	let adminConfig: any = null;
	let codeExecutionEnabled = false;
	let codeInterpreterEnabled = false;

	let enableMemories = false;
	let enableImageGeneration = false;
	let enableCalendar = false;
	let enableWebSearchCap = false;

	const stepsOrder: Step[] = [
		'welcome',
		'connections',
		'web_search',
		'code_execution',
		'capabilities',
		'done'
	];
	$: stepIndex = stepsOrder.indexOf(step);
	$: progressPct = Math.round((stepIndex / (stepsOrder.length - 1)) * 100);

	const skip = () => {
		if (stepIndex < stepsOrder.length - 1) step = stepsOrder[stepIndex + 1];
	};

	const back = () => {
		if (stepIndex > 0) step = stepsOrder[stepIndex - 1];
	};

	const finish = async () => {
		// Record completion on the admin's user settings so the wizard
		// doesn't re-trigger on next sign-in. The wizard already runs only
		// when the user has no `adminOnboardingCompleted` flag set.
		await settings.set({ ...$settings, adminOnboardingCompleted: true });
		await updateUserSettings(localStorage.token, { ui: $settings });
		await goto('/');
	};

	const saveConnections = async () => {
		saving = true;
		try {
			if (ollamaEnabled) {
				const cur = await getOllamaConfig(localStorage.token);
				await updateOllamaConfig(localStorage.token, {
					...cur,
					ENABLE_OLLAMA_API: true,
					OLLAMA_BASE_URLS: ollamaUrl ? [ollamaUrl] : cur.OLLAMA_BASE_URLS
				} as any);
			}
			if (openaiEnabled && openaiKey) {
				const cur = await getOpenAIConfig(localStorage.token);
				await updateOpenAIConfig(localStorage.token, {
					...cur,
					ENABLE_OPENAI_API: true,
					OPENAI_API_KEYS: [openaiKey],
					OPENAI_API_BASE_URLS: [openaiUrl]
				} as any);
			}
			step = 'web_search';
		} catch (e: any) {
			toast.error(`${e?.detail ?? e}`);
		} finally {
			saving = false;
		}
	};

	const saveWebSearch = async () => {
		saving = true;
		try {
			const cur = await getRAGConfig(localStorage.token);
			await updateRAGConfig(localStorage.token, {
				...cur,
				web: {
					...(cur.web ?? {}),
					ENABLE_WEB_SEARCH: webSearchEnabled,
					WEB_SEARCH_ENGINE: webSearchEngine,
					BYPASS_WEB_SEARCH_EMBEDDING_AND_RETRIEVAL: bypassEmbedding
				}
			} as any);
			step = 'code_execution';
		} catch (e: any) {
			toast.error(`${e?.detail ?? e}`);
		} finally {
			saving = false;
		}
	};

	const saveCodeExecution = async () => {
		saving = true;
		try {
			if (!adminConfig) adminConfig = await getAdminConfig(localStorage.token);
			adminConfig.ENABLE_CODE_EXECUTION = codeExecutionEnabled;
			adminConfig.ENABLE_CODE_INTERPRETER = codeInterpreterEnabled;
			await updateAdminConfig(localStorage.token, adminConfig);
			step = 'capabilities';
		} catch (e: any) {
			toast.error(`${e?.detail ?? e}`);
		} finally {
			saving = false;
		}
	};

	const saveCapabilities = async () => {
		saving = true;
		try {
			if (!adminConfig) adminConfig = await getAdminConfig(localStorage.token);
			adminConfig.ENABLE_MEMORIES = enableMemories;
			adminConfig.ENABLE_IMAGE_GENERATION = enableImageGeneration;
			adminConfig.ENABLE_CALENDAR = enableCalendar;
			await updateAdminConfig(localStorage.token, adminConfig);

			// Web search admin enable was set in the web_search step, but the
			// "Web search" capability toggle here mirrors it for visibility.
			if (enableWebSearchCap !== webSearchEnabled) {
				const cur = await getRAGConfig(localStorage.token);
				await updateRAGConfig(localStorage.token, {
					...cur,
					web: { ...(cur.web ?? {}), ENABLE_WEB_SEARCH: enableWebSearchCap }
				} as any);
			}

			await config.set(await getBackendConfig());
			step = 'done';
		} catch (e: any) {
			toast.error(`${e?.detail ?? e}`);
		} finally {
			saving = false;
		}
	};

	onMount(async () => {
		if ($user?.role !== 'admin') {
			await goto('/');
			return;
		}
		try {
			adminConfig = await getAdminConfig(localStorage.token);
			enableMemories = !!adminConfig?.ENABLE_MEMORIES;
			enableImageGeneration = !!adminConfig?.ENABLE_IMAGE_GENERATION;
			enableCalendar = !!adminConfig?.ENABLE_CALENDAR;
			codeExecutionEnabled = !!adminConfig?.ENABLE_CODE_EXECUTION;
			codeInterpreterEnabled = !!adminConfig?.ENABLE_CODE_INTERPRETER;

			const rag = await getRAGConfig(localStorage.token);
			webSearchEnabled = !!rag?.web?.ENABLE_WEB_SEARCH;
			enableWebSearchCap = webSearchEnabled;
			webSearchEngine = rag?.web?.WEB_SEARCH_ENGINE || 'duckduckgo';
			bypassEmbedding = rag?.web?.BYPASS_WEB_SEARCH_EMBEDDING_AND_RETRIEVAL ?? true;

			const ollama = await getOllamaConfig(localStorage.token).catch(() => null);
			if (ollama) {
				ollamaEnabled = !!ollama.ENABLE_OLLAMA_API;
				ollamaUrl = ollama.OLLAMA_BASE_URLS?.[0] ?? ollamaUrl;
			}
			const openai = await getOpenAIConfig(localStorage.token).catch(() => null);
			if (openai) {
				openaiEnabled = !!openai.ENABLE_OPENAI_API;
				openaiKey = openai.OPENAI_API_KEYS?.[0] ?? '';
				openaiUrl = openai.OPENAI_API_BASE_URLS?.[0] ?? openaiUrl;
			}
		} catch (e) {
			console.error(e);
		} finally {
			loaded = true;
		}
	});
</script>

<svelte:head>
	<title>{$i18n.t('Set up Open WebUI')} | Open WebUI</title>
</svelte:head>

<div
	class="fixed inset-0 z-[60] flex flex-col h-screen max-h-[100dvh] overflow-hidden bg-white dark:bg-gray-900 text-gray-700 dark:text-gray-100"
>
	<div
		class="flex items-center justify-between gap-4 px-4 md:px-6 py-3 border-b border-gray-50 dark:border-gray-850/50 shrink-0"
	>
		<div class="text-lg font-medium">{$i18n.t('Set up Open WebUI')}</div>
		<button
			class="text-xs px-2.5 py-1 rounded-lg text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-white transition"
			on:click={finish}
		>
			{$i18n.t('Skip setup')}
		</button>
	</div>

	<!-- Progress bar -->
	<div class="h-0.5 bg-gray-100 dark:bg-gray-850 shrink-0">
		<div
			class="h-full bg-gray-900 dark:bg-white transition-all duration-300"
			style="width: {progressPct}%;"
		></div>
	</div>

	<div class="flex-1 min-h-0 overflow-y-auto">
		<div class="max-w-2xl w-full mx-auto px-4 md:px-6 py-8 md:py-12">
			{#if !loaded}
				<div class="flex items-center justify-center py-20">
					<Spinner className="size-5" />
				</div>
			{:else if step === 'welcome'}
				<div class="space-y-4">
					<h1 class="text-2xl font-medium">
						{$i18n.t("Welcome, {{name}}", { name: $user?.name ?? 'admin' })}
					</h1>
					<p class="text-sm text-gray-600 dark:text-gray-400">
						{$i18n.t(
							"You're the first user, so you're the admin for this Open WebUI install. The next few steps configure model connections, web search, and what your users can do. You can skip any step and change everything later from the admin panel."
						)}
					</p>
					<div class="text-sm text-gray-500 dark:text-gray-500 space-y-1.5">
						<div>1. {$i18n.t('Model connections (Ollama, OpenAI)')}</div>
						<div>2. {$i18n.t('Web search')}</div>
						<div>3. {$i18n.t('Code execution and file creation')}</div>
						<div>4. {$i18n.t('Default capabilities')}</div>
					</div>

					<div class="pt-4 flex gap-2 justify-end">
						<button
							class="px-4 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 rounded-full transition"
							on:click={() => (step = 'connections')}
						>
							{$i18n.t('Get started')}
						</button>
					</div>
				</div>
			{:else if step === 'connections'}
				<div class="space-y-5">
					<div>
						<h1 class="text-xl font-medium">{$i18n.t('Model connections')}</h1>
						<p class="text-sm text-gray-500 mt-1">
							{$i18n.t('Connect at least one model provider. You can add more later from /admin/settings/connections.')}
						</p>
					</div>

					<div class="space-y-3">
						<div class="border border-gray-100 dark:border-gray-850 rounded-xl p-4 space-y-3">
							<div class="flex justify-between items-center">
								<div>
									<div class="text-sm font-medium">{$i18n.t('Ollama')}</div>
									<div class="text-xs text-gray-500">{$i18n.t('Local models. Defaults to http://localhost:11434.')}</div>
								</div>
								<Switch bind:state={ollamaEnabled} />
							</div>
							{#if ollamaEnabled}
								<input
									type="url"
									class="w-full text-sm bg-transparent border border-gray-100 dark:border-gray-800 rounded-lg py-1.5 px-2.5 outline-hidden"
									placeholder="http://localhost:11434"
									bind:value={ollamaUrl}
								/>
							{/if}
						</div>

						<div class="border border-gray-100 dark:border-gray-850 rounded-xl p-4 space-y-3">
							<div class="flex justify-between items-center">
								<div>
									<div class="text-sm font-medium">{$i18n.t('OpenAI API')}</div>
									<div class="text-xs text-gray-500">{$i18n.t('Any OpenAI-compatible endpoint (OpenAI, Anthropic, Groq, etc.).')}</div>
								</div>
								<Switch bind:state={openaiEnabled} />
							</div>
							{#if openaiEnabled}
								<input
									type="password"
									class="w-full text-sm bg-transparent border border-gray-100 dark:border-gray-800 rounded-lg py-1.5 px-2.5 outline-hidden"
									placeholder="sk-..."
									bind:value={openaiKey}
								/>
								<input
									type="url"
									class="w-full text-sm bg-transparent border border-gray-100 dark:border-gray-800 rounded-lg py-1.5 px-2.5 outline-hidden"
									placeholder="https://api.openai.com/v1"
									bind:value={openaiUrl}
								/>
							{/if}
						</div>
					</div>

					<div class="pt-2 flex gap-2 justify-between">
						<button class="text-sm text-gray-500 hover:text-gray-700 dark:hover:text-white" on:click={back}>
							{$i18n.t('Back')}
						</button>
						<div class="flex gap-2">
							<button
								class="px-3 py-1.5 text-sm font-medium hover:bg-gray-100 dark:hover:bg-gray-850 rounded-full transition"
								on:click={skip}
								disabled={saving}
							>
								{$i18n.t('Skip')}
							</button>
							<button
								class="px-4 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 rounded-full transition disabled:opacity-50"
								on:click={saveConnections}
								disabled={saving}
							>
								{saving ? $i18n.t('Saving…') : $i18n.t('Continue')}
							</button>
						</div>
					</div>
				</div>
			{:else if step === 'web_search'}
				<div class="space-y-5">
					<div>
						<h1 class="text-xl font-medium">{$i18n.t('Web search')}</h1>
						<p class="text-sm text-gray-500 mt-1">
							{$i18n.t("When enabled, models can search the web for current information.")}
						</p>
					</div>

					<div class="border border-gray-100 dark:border-gray-850 rounded-xl p-4 space-y-4">
						<div class="flex justify-between items-center">
							<div class="text-sm font-medium">{$i18n.t('Enable web search')}</div>
							<Switch bind:state={webSearchEnabled} />
						</div>

						{#if webSearchEnabled}
							<div class="space-y-3">
								<div>
									<div class="text-xs font-medium mb-1.5">{$i18n.t('Search engine')}</div>
									<select
										class="w-full text-sm bg-transparent border border-gray-100 dark:border-gray-800 rounded-lg py-1.5 px-2.5 outline-hidden"
										bind:value={webSearchEngine}
									>
										<option value="duckduckgo">{$i18n.t('DuckDuckGo (Recommended — no API key required)')}</option>
										<option value="searxng">SearXNG</option>
										<option value="brave">Brave</option>
										<option value="google_pse">Google Programmable Search</option>
										<option value="tavily">Tavily</option>
										<option value="perplexity_search">Perplexity</option>
									</select>
								</div>

								<div class="flex justify-between items-start gap-3">
									<div class="flex-1">
										<div class="text-xs font-medium">{$i18n.t('Bypass embedding and retrieval (Recommended)')}</div>
										<div class="text-xs text-gray-500 mt-0.5">
											{$i18n.t('Feed search hits directly to the model rather than embedding them first. Faster and produces better answers for most models.')}
										</div>
									</div>
									<Switch bind:state={bypassEmbedding} />
								</div>
							</div>
						{/if}
					</div>

					<div class="pt-2 flex gap-2 justify-between">
						<button class="text-sm text-gray-500 hover:text-gray-700 dark:hover:text-white" on:click={back}>
							{$i18n.t('Back')}
						</button>
						<div class="flex gap-2">
							<button
								class="px-3 py-1.5 text-sm font-medium hover:bg-gray-100 dark:hover:bg-gray-850 rounded-full transition"
								on:click={skip}
								disabled={saving}
							>
								{$i18n.t('Skip')}
							</button>
							<button
								class="px-4 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 rounded-full transition disabled:opacity-50"
								on:click={saveWebSearch}
								disabled={saving}
							>
								{saving ? $i18n.t('Saving…') : $i18n.t('Continue')}
							</button>
						</div>
					</div>
				</div>
			{:else if step === 'code_execution'}
				<div class="space-y-5">
					<div>
						<h1 class="text-xl font-medium">{$i18n.t('Code execution and file creation')}</h1>
						<p class="text-sm text-gray-500 mt-1">
							{$i18n.t("Let models run code in a sandbox and produce downloadable files. Compute environment defaults to the in-browser Pyodide runtime — fully featured network egress + domain allowlist will land in a later phase.")}
						</p>
					</div>

					<div class="border border-gray-100 dark:border-gray-850 rounded-xl p-4 space-y-3">
						<div class="flex justify-between items-center">
							<div>
								<div class="text-sm font-medium">{$i18n.t('Code execution')}</div>
								<div class="text-xs text-gray-500">{$i18n.t('General code blocks the model can run.')}</div>
							</div>
							<Switch bind:state={codeExecutionEnabled} />
						</div>
						<div class="flex justify-between items-center">
							<div>
								<div class="text-sm font-medium">{$i18n.t('Code interpreter')}</div>
								<div class="text-xs text-gray-500">{$i18n.t('Tool-based code execution invoked by the model. Allows file creation.')}</div>
							</div>
							<Switch bind:state={codeInterpreterEnabled} />
						</div>
					</div>

					<div class="pt-2 flex gap-2 justify-between">
						<button class="text-sm text-gray-500 hover:text-gray-700 dark:hover:text-white" on:click={back}>
							{$i18n.t('Back')}
						</button>
						<div class="flex gap-2">
							<button
								class="px-3 py-1.5 text-sm font-medium hover:bg-gray-100 dark:hover:bg-gray-850 rounded-full transition"
								on:click={skip}
								disabled={saving}
							>
								{$i18n.t('Skip')}
							</button>
							<button
								class="px-4 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 rounded-full transition disabled:opacity-50"
								on:click={saveCodeExecution}
								disabled={saving}
							>
								{saving ? $i18n.t('Saving…') : $i18n.t('Continue')}
							</button>
						</div>
					</div>
				</div>
			{:else if step === 'capabilities'}
				<div class="space-y-5">
					<div>
						<h1 class="text-xl font-medium">{$i18n.t('Default capabilities')}</h1>
						<p class="text-sm text-gray-500 mt-1">
							{$i18n.t("Turn on the features you want available to your users. Each user can still opt in or out from their own Settings → Capabilities.")}
						</p>
					</div>

					<div class="border border-gray-100 dark:border-gray-850 rounded-xl p-4 divide-y divide-gray-50 dark:divide-gray-850/60">
						<div class="flex justify-between items-center py-2">
							<div>
								<div class="text-sm font-medium">{$i18n.t('Memories')}</div>
								<div class="text-xs text-gray-500">{$i18n.t('One rolling memory file per user.')}</div>
							</div>
							<Switch bind:state={enableMemories} />
						</div>
						<div class="flex justify-between items-center py-2">
							<div>
								<div class="text-sm font-medium">{$i18n.t('Web search')}</div>
								<div class="text-xs text-gray-500">{$i18n.t('Mirrors the toggle from the web search step.')}</div>
							</div>
							<Switch bind:state={enableWebSearchCap} />
						</div>
						<div class="flex justify-between items-center py-2">
							<div>
								<div class="text-sm font-medium">{$i18n.t('Image generation')}</div>
								<div class="text-xs text-gray-500">{$i18n.t('Requires a configured image provider in /admin/settings/images.')}</div>
							</div>
							<Switch bind:state={enableImageGeneration} />
						</div>
						<div class="flex justify-between items-center py-2">
							<div>
								<div class="text-sm font-medium">{$i18n.t('Calendar')}</div>
								<div class="text-xs text-gray-500">{$i18n.t('Read/write calendar events with user approval.')}</div>
							</div>
							<Switch bind:state={enableCalendar} />
						</div>
					</div>

					<div class="pt-2 flex gap-2 justify-between">
						<button class="text-sm text-gray-500 hover:text-gray-700 dark:hover:text-white" on:click={back}>
							{$i18n.t('Back')}
						</button>
						<div class="flex gap-2">
							<button
								class="px-3 py-1.5 text-sm font-medium hover:bg-gray-100 dark:hover:bg-gray-850 rounded-full transition"
								on:click={skip}
								disabled={saving}
							>
								{$i18n.t('Skip')}
							</button>
							<button
								class="px-4 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 rounded-full transition disabled:opacity-50"
								on:click={saveCapabilities}
								disabled={saving}
							>
								{saving ? $i18n.t('Saving…') : $i18n.t('Continue')}
							</button>
						</div>
					</div>
				</div>
			{:else if step === 'done'}
				<div class="space-y-4 text-center py-6">
					<div class="text-3xl">✓</div>
					<h1 class="text-2xl font-medium">{$i18n.t("You're all set")}</h1>
					<p class="text-sm text-gray-500">
						{$i18n.t('You can change any of these settings later from the admin panel.')}
					</p>
					<div class="pt-4 flex justify-center">
						<button
							class="px-5 py-2 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 rounded-full transition"
							on:click={finish}
						>
							{$i18n.t('Start chatting')}
						</button>
					</div>
				</div>
			{/if}
		</div>
	</div>
</div>
