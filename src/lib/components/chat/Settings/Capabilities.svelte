<script lang="ts">
	import { getContext, onMount } from 'svelte';
	import { config, settings, user } from '$lib/stores';
	import Switch from '$lib/components/common/Switch.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import ManageModal from './Personalization/ManageModal.svelte';

	const i18n = getContext('i18n');

	export let saveSettings: Function;

	type CapabilityKey =
		| 'memory'
		| 'web_search'
		| 'code_execution_and_files'
		| 'image_generation'
		| 'calendar'
		| 'canvas';

	type ToolLoadingMode = 'auto' | 'always_loaded' | 'on_demand';

	const DEFAULTS: Record<CapabilityKey, boolean> = {
		memory: false,
		web_search: false,
		code_execution_and_files: false,
		image_generation: false,
		calendar: false,
		canvas: true
	};

	const ITEMS: { key: CapabilityKey; title: string; description: string }[] = [
		{
			key: 'memory',
			title: 'Memory',
			description:
				'Let the model remember details across chats. When on, the model can read and write your memory automatically.'
		},
		{
			key: 'web_search',
			title: 'Web search',
			description:
				'Allow the model to search the web when it needs current information. The model decides when to use it.'
		},
		{
			key: 'code_execution_and_files',
			title: 'Code execution and file creation',
			description:
				'Let the model run code in a sandbox and produce downloadable files (CSV, JSON, plots, etc.).'
		},
		{
			key: 'image_generation',
			title: 'Image generation',
			description: 'Allow the model to generate images via the configured image provider.'
		},
		{
			key: 'calendar',
			title: 'Calendar',
			description:
				'Let the model read and propose changes to your calendar. Every write requires your approval.'
		},
		{
			key: 'canvas',
			title: 'Canvas / Artifacts',
			description:
				'Let the model create and iteratively edit documents, code, diagrams, and HTML widgets alongside the conversation.'
		}
	];

	// A capability is "available" when both the admin-level feature flag
	// and the user-level permission allow it. If either is off, the
	// toggle is greyed out and locked in the user UI — the admin panel
	// is the only place to re-enable it.
	$: availability = ((): Record<CapabilityKey, { allowed: boolean; reason?: string }> => {
		const perms = ($user as any)?.permissions?.features ?? {};
		const cfg = $config?.features ?? ({} as any);
		const adminOn = (v: any) => v === undefined || !!v;
		const permOn = (v: any) => v === undefined || !!v;

		// Note: admins see the same gating as users so the UI accurately
		// reflects what's available system-wide. Admins still have full
		// access — they can flip the gate in /admin/settings to unblock it.
		const check = (
			feature: string,
			adminFlag: boolean,
			permFlag: boolean
		): { allowed: boolean; reason?: string } => {
			if (!adminFlag) {
				return { allowed: false, reason: `${feature} is disabled in the admin panel.` };
			}
			if (!permFlag) {
				return { allowed: false, reason: `${feature} isn't enabled for your account.` };
			}
			return { allowed: true };
		};

		return {
			memory: check('Memory', adminOn(cfg.enable_memories), permOn(perms.memories)),
			web_search: check('Web search', adminOn(cfg.enable_web_search), permOn(perms.web_search)),
			// Backend exposes both enable_code_execution and enable_code_interpreter;
			// either being on is enough to allow the user-level capability.
			code_execution_and_files: check(
				'Code execution',
				adminOn(cfg.enable_code_interpreter) || adminOn(cfg.enable_code_execution),
				permOn(perms.code_interpreter)
			),
			image_generation: check(
				'Image generation',
				adminOn(cfg.enable_image_generation),
				permOn(perms.image_generation)
			),
			calendar: check('Calendar', adminOn(cfg.enable_calendar), permOn(perms.calendar)),
			canvas: { allowed: true }
		};
	})();

	let capabilities: Record<CapabilityKey, boolean> = { ...DEFAULTS };
	let toolLoadingMode: ToolLoadingMode = 'auto';
	let showManageMemory = false;

	const persist = () => {
		saveSettings({
			capabilities: { ...capabilities },
			toolLoadingMode,
			memory: capabilities.memory
		});
	};

	const toggle = (key: CapabilityKey, next: boolean) => {
		if (!availability[key].allowed) return; // hard guard
		capabilities[key] = next;
		persist();
	};

	onMount(() => {
		const stored = ($settings as any)?.capabilities ?? {};
		capabilities = {
			...DEFAULTS,
			...stored,
			memory: stored.memory ?? ($settings as any)?.memory ?? DEFAULTS.memory
		};
		toolLoadingMode = ($settings as any)?.toolLoadingMode ?? 'auto';
	});
</script>

<ManageModal bind:show={showManageMemory} />

<div class="flex flex-col h-full text-sm">
	<div class="overflow-y-auto pr-2">
		<div>
			<h1 class="mb-1 text-sm font-medium">{$i18n.t('Capabilities')}</h1>

			<div class="mb-3 text-xs text-gray-500">
				{$i18n.t(
					'Turn capabilities on to let models invoke them automatically when useful. You can still override per-chat from the + menu in any chat.'
				)}
			</div>

			<div class="flex flex-col">
				{#each ITEMS as item (item.key)}
					{@const avail = availability[item.key]}
					<div>
						<div class="py-0.5 flex w-full justify-between gap-3 items-start">
							<div class="flex-1 min-w-0">
								<div
									id="cap-{item.key}-label"
									class="text-xs font-medium {!avail.allowed ? 'text-gray-400 dark:text-gray-500' : ''}"
								>
									{$i18n.t(item.title)}
									{#if !avail.allowed}
										<span
											class="ml-1 text-[0.65rem] font-medium uppercase px-1.5 py-0.5 rounded-full bg-gray-100 dark:bg-gray-800 text-gray-500 dark:text-gray-400"
										>
											{$i18n.t('Disabled')}
										</span>
									{/if}
								</div>
								<div
									class="text-xs mt-0.5 {!avail.allowed ? 'text-gray-400 dark:text-gray-600' : 'text-gray-500'}"
								>
									{$i18n.t(item.description)}
								</div>
								{#if !avail.allowed && avail.reason}
									<div class="text-xs mt-1 text-gray-400 dark:text-gray-500 italic">
										{$i18n.t(avail.reason)}
									</div>
								{/if}

								{#if item.key === 'memory' && capabilities.memory && avail.allowed}
									<div class="mt-2">
										<button
											type="button"
											class="px-3 py-1 text-xs font-medium hover:bg-black/5 dark:hover:bg-white/5 outline outline-1 outline-gray-200 dark:outline-gray-800 rounded-full"
											on:click={() => (showManageMemory = true)}
										>
											{$i18n.t('View memory')}
										</button>
									</div>
								{/if}
							</div>

							<div class="flex items-center gap-2 p-1 shrink-0">
								{#if avail.allowed}
									<Switch
										ariaLabelledbyId="cap-{item.key}-label"
										tooltip={true}
										state={capabilities[item.key]}
										on:change={(e) => toggle(item.key, e.detail)}
									/>
								{:else}
									<Tooltip content={$i18n.t(avail.reason ?? '')} placement="left">
										<div class="opacity-40 pointer-events-none" aria-disabled="true">
											<Switch
												ariaLabelledbyId="cap-{item.key}-label"
												state={false}
											/>
										</div>
									</Tooltip>
								{/if}
							</div>
						</div>
					</div>
				{/each}
			</div>
		</div>

		<div class="mt-5">
			<h1 class="mb-1 text-sm font-medium">{$i18n.t('Advanced')}</h1>

			<div>
				<div class="py-0.5 flex w-full justify-between gap-3 items-start flex-col sm:flex-row">
					<div class="flex-1 min-w-0">
						<div id="tool-loading-mode-label" class="text-xs font-medium">
							{$i18n.t('Tool loading mode')}
						</div>
						<div class="text-xs text-gray-500 mt-0.5">
							{$i18n.t(
								'How tool definitions are made available to the model. Auto keeps recently-used tools loaded. On-demand asks the model to request a tool before using it.'
							)}
						</div>
					</div>

					<div class="flex items-center gap-2 p-1 shrink-0 self-stretch sm:self-start">
						<select
							aria-labelledby="tool-loading-mode-label"
							class="text-xs bg-transparent outline-hidden border border-gray-100 dark:border-gray-800 rounded-lg py-1 px-2 min-w-[10rem]"
							bind:value={toolLoadingMode}
							on:change={persist}
						>
							<option value="auto">{$i18n.t('Auto (recommended)')}</option>
							<option value="always_loaded">{$i18n.t('Always loaded')}</option>
							<option value="on_demand">{$i18n.t('On demand')}</option>
						</select>
					</div>
				</div>
			</div>
		</div>
	</div>
</div>
