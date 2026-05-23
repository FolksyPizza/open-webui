<script lang="ts">
	import { onDestroy, onMount, getContext } from 'svelte';
	import { goto } from '$app/navigation';

	import { WEBUI_NAME, config, showSidebar, user } from '$lib/stores';
	import { page } from '$app/stores';
	import Tooltip from '$lib/components/common/Tooltip.svelte';

	const i18n = getContext('i18n');

	let loaded = false;

	// Admin is a dedicated full-screen surface — match /settings: hide the
	// chat sidebar on entry, restore on exit.
	let priorSidebar: boolean | undefined;

	onMount(async () => {
		if ($user?.role !== 'admin') {
			await goto('/');
			return;
		}
		priorSidebar = ($showSidebar as unknown) as boolean;
		showSidebar.set(false);
		loaded = true;
	});

	onDestroy(() => {
		if (priorSidebar !== undefined) showSidebar.set(priorSidebar);
	});
</script>

<svelte:head>
	<title>
		{$i18n.t('Admin Panel')} • {$WEBUI_NAME}
	</title>
</svelte:head>

{#if loaded}
	<div
		class="fixed inset-0 z-[60] flex flex-col h-screen max-h-[100dvh] overflow-hidden bg-white dark:bg-gray-900 text-gray-700 dark:text-gray-100"
	>
		<div
			class="flex items-center justify-between gap-4 px-4 md:px-6 py-3 border-b border-gray-50 dark:border-gray-850/50 shrink-0"
		>
			<div class="flex items-center gap-2 min-w-0">
				<Tooltip content={$i18n.t('Back to chat')} placement="bottom">
					<button
						class="p-1 rounded-md hover:bg-gray-100 dark:hover:bg-gray-850 transition"
						aria-label={$i18n.t('Back')}
						on:click={() => goto('/')}
					>
						<svg
							xmlns="http://www.w3.org/2000/svg"
							viewBox="0 0 20 20"
							fill="currentColor"
							class="w-4 h-4"
						>
							<path
								fill-rule="evenodd"
								d="M11.78 5.22a.75.75 0 0 1 0 1.06L8.06 10l3.72 3.72a.75.75 0 1 1-1.06 1.06l-4.25-4.25a.75.75 0 0 1 0-1.06l4.25-4.25a.75.75 0 0 1 1.06 0Z"
								clip-rule="evenodd"
							/>
						</svg>
					</button>
				</Tooltip>
				<div class="text-lg font-medium truncate">{$i18n.t('Admin Panel')}</div>
			</div>

			<nav class="flex-1 min-w-0">
				<div
					class="flex gap-1 scrollbar-none overflow-x-auto justify-end text-sm font-medium rounded-full bg-transparent"
				>
					<a
						draggable="false"
						class="px-2.5 py-1 rounded-xl transition select-none {$page.url.pathname.includes(
							'/admin/users'
						) || $page.url.pathname === '/admin'
							? 'bg-gray-100 dark:bg-gray-850/50'
							: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-white'}"
						href="/admin">{$i18n.t('Users')}</a
					>

					{#if $config?.features.enable_admin_analytics ?? true}
						<a
							draggable="false"
							class="px-2.5 py-1 rounded-xl transition select-none {$page.url.pathname.includes(
								'/admin/analytics'
							)
								? 'bg-gray-100 dark:bg-gray-850/50'
								: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-white'}"
							href="/admin/analytics">{$i18n.t('Analytics')}</a
						>
					{/if}

					<a
						draggable="false"
						class="px-2.5 py-1 rounded-xl transition select-none {$page.url.pathname.includes(
							'/admin/evaluations'
						)
							? 'bg-gray-100 dark:bg-gray-850/50'
							: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-white'}"
						href="/admin/evaluations">{$i18n.t('Evaluations')}</a
					>

					<a
						draggable="false"
						class="px-2.5 py-1 rounded-xl transition select-none {$page.url.pathname.includes(
							'/admin/functions'
						)
							? 'bg-gray-100 dark:bg-gray-850/50'
							: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-white'}"
						href="/admin/functions">{$i18n.t('Functions')}</a
					>

					<a
						draggable="false"
						class="px-2.5 py-1 rounded-xl transition select-none {$page.url.pathname.includes(
							'/admin/settings'
						)
							? 'bg-gray-100 dark:bg-gray-850/50'
							: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-white'}"
						href="/admin/settings">{$i18n.t('Settings')}</a
					>
				</div>
			</nav>
		</div>

		<div class="flex-1 min-h-0 overflow-y-auto py-4">
			<slot />
		</div>
	</div>
{/if}
