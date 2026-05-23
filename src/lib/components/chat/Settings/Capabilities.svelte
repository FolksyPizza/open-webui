<script lang="ts">
	import { getContext, onMount } from 'svelte';
	import { settings } from '$lib/stores';
	import Switch from '$lib/components/common/Switch.svelte';

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

	let capabilities: Record<CapabilityKey, boolean> = { ...DEFAULTS };
	let toolLoadingMode: ToolLoadingMode = 'auto';

	const persist = () => {
		saveSettings({
			capabilities: { ...capabilities },
			toolLoadingMode
		});
	};

	const toggle = (key: CapabilityKey, next: boolean) => {
		capabilities[key] = next;
		persist();
	};

	onMount(() => {
		const stored = ($settings as any)?.capabilities ?? {};
		capabilities = { ...DEFAULTS, ...stored };
		toolLoadingMode = ($settings as any)?.toolLoadingMode ?? 'auto';
	});
</script>

<div class="flex flex-col h-full justify-between text-sm">
	<div class="overflow-y-scroll max-h-[28rem] lg:max-h-full pr-1.5">
		<div>
			<div class=" mb-1 text-sm font-medium">{$i18n.t('Capabilities')}</div>

			<div class=" mb-3 text-xs text-gray-500">
				{$i18n.t(
					'Turn capabilities on to let models invoke them automatically when useful. You can still override per-chat from the + menu in any chat.'
				)}
			</div>

			<div class="flex flex-col gap-1.5">
				{#each ITEMS as item (item.key)}
					<div class="py-1 border-b border-gray-50 dark:border-gray-850/50 last:border-b-0">
						<div class="flex w-full justify-between items-start gap-3">
							<div class="flex-1 min-w-0">
								<div id="cap-{item.key}-label" class="text-xs font-medium">
									{$i18n.t(item.title)}
								</div>
								<div class="text-xs text-gray-500 mt-0.5">
									{$i18n.t(item.description)}
								</div>
							</div>

							<div class="flex items-center gap-2 p-1 shrink-0">
								<Switch
									ariaLabelledbyId="cap-{item.key}-label"
									tooltip={true}
									state={capabilities[item.key]}
									on:change={(e) => toggle(item.key, e.detail)}
								/>
							</div>
						</div>
					</div>
				{/each}
			</div>
		</div>

		<div class="mt-5">
			<div class=" mb-1 text-sm font-medium">{$i18n.t('Advanced')}</div>

			<div class="py-1">
				<div class="flex w-full justify-between items-start gap-3">
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

					<div class="flex items-center gap-2 p-1 shrink-0">
						<select
							aria-labelledby="tool-loading-mode-label"
							class="text-xs bg-transparent outline-hidden border border-gray-100 dark:border-gray-850 rounded-lg py-1 px-2"
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
