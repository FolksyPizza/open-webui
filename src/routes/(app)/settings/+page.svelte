<script lang="ts">
	import { getContext, onDestroy, onMount, tick } from 'svelte';
	import { toast } from 'svelte-sonner';
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';

	import { config, models, settings, showSidebar, user } from '$lib/stores';
	import { updateUserSettings } from '$lib/apis/users';
	import { getModels as _getModels } from '$lib/apis';

	import Capabilities from '$lib/components/chat/Settings/Capabilities.svelte';
	import General from '$lib/components/chat/Settings/General.svelte';
	import Interface from '$lib/components/chat/Settings/Interface.svelte';
	import Connections from '$lib/components/chat/Settings/Connections.svelte';
	import Integrations from '$lib/components/chat/Settings/Integrations.svelte';
	import Audio from '$lib/components/chat/Settings/Audio.svelte';
	import DataControls from '$lib/components/chat/Settings/DataControls.svelte';
	import Account from '$lib/components/chat/Settings/Account.svelte';
	import About from '$lib/components/chat/Settings/About.svelte';

	import WrenchAlt from '$lib/components/icons/WrenchAlt.svelte';
	import SettingsAlt from '$lib/components/icons/SettingsAlt.svelte';
	import Link from '$lib/components/icons/Link.svelte';
	import AppNotification from '$lib/components/icons/AppNotification.svelte';
	import SoundHigh from '$lib/components/icons/SoundHigh.svelte';
	import DatabaseSettings from '$lib/components/icons/DatabaseSettings.svelte';
	import UserCircle from '$lib/components/icons/UserCircle.svelte';
	import InfoCircle from '$lib/components/icons/InfoCircle.svelte';
	import UserBadgeCheck from '$lib/components/icons/UserBadgeCheck.svelte';

	const i18n = getContext('i18n');

	interface Tab {
		id: string;
		title: string;
		icon: any;
		visible?: () => boolean;
	}

	const tabs: Tab[] = [
		{ id: 'general', title: 'General', icon: SettingsAlt },
		{
			id: 'interface',
			title: 'Interface',
			icon: AppNotification,
			visible: () => $user?.role === 'admin' || ($user?.permissions?.settings?.interface ?? true)
		},
		{ id: 'capabilities', title: 'Capabilities', icon: WrenchAlt },
		{
			id: 'connections',
			title: 'Connections',
			icon: Link,
			visible: () => !!$config?.features?.enable_direct_connections
		},
		{
			id: 'tools',
			title: 'Integrations',
			icon: WrenchAlt,
			visible: () =>
				$user?.role === 'admin' ||
				($user?.role === 'user' && ($user?.permissions?.features?.direct_tool_servers ?? false))
		},
		{ id: 'audio', title: 'Audio', icon: SoundHigh },
		{ id: 'data_controls', title: 'Data Controls', icon: DatabaseSettings },
		{ id: 'account', title: 'Account', icon: UserCircle },
		{ id: 'about', title: 'About', icon: InfoCircle }
	];

	$: visibleTabs = tabs.filter((t) => (t.visible ? t.visible() : true));

	let selectedTab: string = 'general';

	const getModels = async () => {
		return await _getModels(
			localStorage.token,
			$config?.features?.enable_direct_connections && ($settings?.directConnections ?? null)
		);
	};

	// Silent saveSettings — no toast notifications on individual setting
	// changes (toggles, inputs auto-save on change). The value still
	// persists to the user record via updateUserSettings.
	const saveSettings = async (updated: any) => {
		await settings.set({ ...$settings, ...updated });
		await models.set(await getModels());
		await updateUserSettings(localStorage.token, { ui: $settings });
	};

	// Explicit-Save confirmation. Only fires when a tab's "Save" button is
	// clicked (General / Interface / Audio dispatch a `save` event;
	// Account uses a `saveHandler` callback). Per-toggle changes stay silent.
	const savedToast = () => {
		toast.success($i18n.t('Settings saved successfully!'));
	};

	const selectTab = async (id: string) => {
		selectedTab = id;
		const url = new URL(window.location.href);
		url.searchParams.set('tab', id);
		await goto(url.pathname + url.search, { replaceState: true, noScroll: true });
	};

	// Hide the chat sidebar while on /settings — Settings is a dedicated
	// full-screen surface. Restore the user's prior state on leave.
	let priorSidebar: boolean | undefined;
	onMount(async () => {
		priorSidebar = ($showSidebar as unknown) as boolean;
		showSidebar.set(false);

		const initial = $page.url.searchParams.get('tab');
		if (initial && tabs.find((t) => t.id === initial)) {
			selectedTab = initial;
		}
		await tick();
	});
	onDestroy(() => {
		if (priorSidebar !== undefined) showSidebar.set(priorSidebar);
	});
</script>

<svelte:head>
	<title>{$i18n.t('Settings')} | Open WebUI</title>
</svelte:head>

<div
	class="fixed inset-0 z-[60] flex flex-col h-screen max-h-[100dvh] overflow-hidden bg-white dark:bg-gray-900 text-gray-700 dark:text-gray-100"
>
	<div
		class="flex items-center justify-between px-4 md:px-6 py-3 border-b border-gray-50 dark:border-gray-850/50 shrink-0"
	>
		<div class="flex items-center gap-2 min-w-0">
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
			<div class="text-lg font-medium truncate">{$i18n.t('Settings')}</div>
		</div>

		{#if $user?.role === 'admin'}
			<a
				href="/admin/settings"
				class="text-xs px-2.5 py-1 rounded-lg border border-gray-100 dark:border-gray-850 hover:bg-gray-100 dark:hover:bg-gray-850 transition flex items-center gap-1.5"
				on:click|preventDefault={() => goto('/admin/settings')}
			>
				<UserBadgeCheck strokeWidth="2" className="size-3.5" />
				<span>{$i18n.t('Admin Settings')}</span>
			</a>
		{/if}
	</div>

	<div
		class="flex flex-col md:flex-row flex-1 min-h-0 w-full px-3 md:px-8 py-4 gap-6"
	>
		<nav
			role="tablist"
			class="flex flex-row md:flex-col overflow-x-auto md:overflow-x-visible md:overflow-y-auto gap-1 md:w-60 shrink-0 -mx-1 px-1 md:mx-0 md:px-0"
		>
			{#each visibleTabs as tab (tab.id)}
				<button
					role="tab"
					aria-controls="tabpanel-{tab.id}"
					aria-selected={selectedTab === tab.id}
					class={`px-2.5 py-1.5 min-w-fit rounded-xl flex items-center text-left transition text-sm
						${
							selectedTab === tab.id
								? ($settings?.highContrastMode ?? false)
									? 'dark:bg-gray-800 bg-gray-200'
									: 'bg-gray-100 dark:bg-gray-850/50'
								: ($settings?.highContrastMode ?? false)
									? 'hover:bg-gray-200 dark:hover:bg-gray-800'
									: 'text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-white'
						}`}
					on:click={() => selectTab(tab.id)}
				>
					<svelte:component this={tab.icon} strokeWidth="2" className="size-4 mr-2 shrink-0" />
					<span class="truncate">{$i18n.t(tab.title)}</span>
				</button>
			{/each}
		</nav>

		<div
			id="tabpanel-{selectedTab}"
			role="tabpanel"
			class="settings-page flex-1 min-h-0 overflow-y-auto pr-1"
		>
			{#if selectedTab === 'general'}
				<General {getModels} {saveSettings} on:save={savedToast} />
			{:else if selectedTab === 'interface'}
				<Interface {saveSettings} on:save={savedToast} />
			{:else if selectedTab === 'capabilities'}
				<Capabilities {saveSettings} />
			{:else if selectedTab === 'connections'}
				<Connections {saveSettings} />
			{:else if selectedTab === 'tools'}
				<Integrations {saveSettings} />
			{:else if selectedTab === 'audio'}
				<Audio {saveSettings} on:save={savedToast} />
			{:else if selectedTab === 'data_controls'}
				<DataControls {saveSettings} />
			{:else if selectedTab === 'account'}
				<Account {saveSettings} saveHandler={savedToast} />
			{:else if selectedTab === 'about'}
				<About />
			{/if}
		</div>
	</div>
</div>

