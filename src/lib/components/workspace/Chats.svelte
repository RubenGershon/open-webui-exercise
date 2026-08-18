<script lang="ts">
	import dayjs from 'dayjs';
	import { onMount, getContext } from 'svelte';
	import { goto } from '$app/navigation';
	import { toast } from 'svelte-sonner';
	import { getChatById, getChatList, updateChatById } from '$lib/apis/chats';
	import { WEBUI_NAME } from '$lib/stores';
	import Spinner from '$lib/components/common/Spinner.svelte';

	const i18n = getContext('i18n');

	type SortKey = 'updated_at' | 'title' | 'model';
	type ManagedChat = {
		id: string;
		title?: string;
		updated_at?: number;
		created_at?: number;
		model?: string;
	};

	let chats: ManagedChat[] = [];
	let loading = true;
	let error = '';
	let sortKey: SortKey = 'updated_at';
	let sortDirection: 'asc' | 'desc' = 'desc';
	let editingId: string | null = null;
	let editingTitle = '';
	let savingId: string | null = null;

	$: sortedChats = [...chats].sort((first, second) => {
		let comparison = 0;
		if (sortKey === 'title') {
			comparison = (first.title ?? '').localeCompare(second.title ?? '');
		} else if (sortKey === 'model') {
			comparison = (first.model ?? '').localeCompare(second.model ?? '');
		} else {
			comparison = (first.updated_at ?? 0) - (second.updated_at ?? 0);
		}
		return sortDirection === 'asc' ? comparison : -comparison;
	});

	const getModelName = (chat) => {
		const models = chat?.chat?.models;
		if (Array.isArray(models) && models.length > 0) return models[0];
		if (typeof chat?.chat?.model === 'string') return chat.chat.model;
		const assistant = Object.values(chat?.chat?.history?.messages ?? {}).find(
			(message: any) => message?.role === 'assistant' && message?.model
		);
		return assistant?.model ?? 'Unknown';
	};

	const loadChats = async () => {
		loading = true;
		error = '';
		try {
			const items = await getChatList(localStorage.token, null, false, false, 'updated_at', 'desc');
			const details = await Promise.all(
				items.map(async (item) => {
					const detail = await getChatById(localStorage.token, item.id).catch(() => null);
					return { ...item, model: getModelName(detail) };
				})
			);
			chats = details;
		} catch (loadError) {
			error = loadError?.message ?? $i18n.t('Failed to load chats');
		} finally {
			loading = false;
		}
	};

	const setSort = (nextKey: SortKey) => {
		if (sortKey === nextKey) {
			sortDirection = sortDirection === 'asc' ? 'desc' : 'asc';
		} else {
			sortKey = nextKey;
			sortDirection = nextKey === 'updated_at' ? 'desc' : 'asc';
		}
	};

	const startEditing = (chat: ManagedChat) => {
		editingId = chat.id;
		editingTitle = chat.title ?? '';
	};

	const cancelEditing = () => {
		editingId = null;
		editingTitle = '';
	};

	const saveTitle = async (chat: ManagedChat) => {
		const title = editingTitle.trim();
		if (!title) {
			toast.error($i18n.t('Title cannot be an empty string.'));
			return;
		}

		savingId = chat.id;
		const updated = await updateChatById(localStorage.token, chat.id, { title }).catch(
			(saveError) => {
				toast.error(saveError?.message ?? $i18n.t('Failed to update chat title'));
				return null;
			}
		);
		savingId = null;
		if (updated) {
			chats = chats.map((item) => (item.id === chat.id ? { ...item, title } : item));
			cancelEditing();
		}
	};

	onMount(loadChats);
</script>

<svelte:head>
	<title>{$i18n.t('Chats')} / {$WEBUI_NAME}</title>
</svelte:head>

<div class="mx-auto flex w-full max-w-5xl flex-col gap-5 py-5">
	<div class="flex flex-wrap items-end justify-between gap-3">
		<div>
			<h1 class="text-xl font-semibold text-gray-900 dark:text-gray-100">{$i18n.t('Chats')}</h1>
			<p class="mt-1 text-sm text-gray-500 dark:text-gray-400">
				{$i18n.t('Manage your conversations')}
			</p>
		</div>
		<div class="flex items-center gap-1 rounded-lg border border-gray-200 p-1 dark:border-gray-800">
			{#each [{ key: 'updated_at', label: $i18n.t('Date') }, { key: 'model', label: $i18n.t('Model') }, { key: 'title', label: $i18n.t('Title') }] as option}
				<button
					type="button"
					class="rounded-md px-2.5 py-1.5 text-xs transition {sortKey === option.key
						? 'bg-gray-100 text-gray-900 dark:bg-gray-800 dark:text-gray-100'
						: 'text-gray-500 hover:bg-gray-50 dark:text-gray-400 dark:hover:bg-gray-900'}"
					on:click={() => setSort(option.key)}
				>
					{option.label}{sortKey === option.key ? ` ${sortDirection === 'asc' ? '↑' : '↓'}` : ''}
				</button>
			{/each}
		</div>
	</div>

	{#if loading}
		<div class="flex min-h-48 items-center justify-center"><Spinner className="size-5" /></div>
	{:else if error}
		<div
			class="rounded-lg border border-red-200 bg-red-50 p-4 text-sm text-red-700 dark:border-red-900/50 dark:bg-red-950/20 dark:text-red-300"
		>
			{error}
		</div>
	{:else if sortedChats.length === 0}
		<div
			class="rounded-lg border border-dashed border-gray-200 p-10 text-center text-sm text-gray-500 dark:border-gray-800 dark:text-gray-400"
		>
			{$i18n.t('No chats found')}
		</div>
	{:else}
		<div class="overflow-hidden rounded-xl border border-gray-200 dark:border-gray-800">
			<div
				class="grid grid-cols-[minmax(0,1fr)_minmax(8rem,0.35fr)_minmax(8rem,0.3fr)_auto] gap-3 border-b border-gray-200 px-4 py-2.5 text-xs font-medium text-gray-500 dark:border-gray-800 dark:text-gray-400"
			>
				<span>{$i18n.t('Title')}</span>
				<span>{$i18n.t('Model')}</span>
				<span>{$i18n.t('Updated')}</span>
				<span class="sr-only">{$i18n.t('Actions')}</span>
			</div>
			{#each sortedChats as chat (chat.id)}
				<div
					class="grid grid-cols-[minmax(0,1fr)_minmax(8rem,0.35fr)_minmax(8rem,0.3fr)_auto] items-center gap-3 border-b border-gray-100 px-4 py-3 last:border-b-0 hover:bg-gray-50 dark:border-gray-900 dark:hover:bg-gray-900/50"
				>
					{#if editingId === chat.id}
						<input
							class="min-w-0 rounded-md border border-gray-200 bg-transparent px-2 py-1 text-sm outline-none focus:border-gray-400 dark:border-gray-700"
							bind:value={editingTitle}
							on:keydown={(event) => event.key === 'Enter' && saveTitle(chat)}
							on:keydown={(event) => event.key === 'Escape' && cancelEditing()}
						/>
					{:else}
						<button
							type="button"
							class="min-w-0 truncate text-left text-sm text-gray-900 hover:underline dark:text-gray-100"
							on:click={() => goto(`/c/${chat.id}`)}
						>
							{chat.title || $i18n.t('Untitled chat')}
						</button>
					{/if}
					<span class="truncate text-sm text-gray-600 dark:text-gray-400">{chat.model}</span>
					<span class="text-xs text-gray-500 dark:text-gray-500"
						>{dayjs.unix(chat.updated_at ?? 0).format('YYYY-MM-DD HH:mm')}</span
					>
					<div class="flex items-center justify-end gap-1">
						{#if editingId === chat.id}
							<button
								type="button"
								class="rounded px-2 py-1 text-xs text-gray-500 hover:bg-gray-100 dark:hover:bg-gray-800"
								disabled={savingId === chat.id}
								on:click={() => saveTitle(chat)}>{$i18n.t('Save')}</button
							>
							<button
								type="button"
								class="rounded px-2 py-1 text-xs text-gray-500 hover:bg-gray-100 dark:hover:bg-gray-800"
								on:click={cancelEditing}>{$i18n.t('Cancel')}</button
							>
						{:else}
							<button
								type="button"
								class="rounded px-2 py-1 text-xs text-gray-500 hover:bg-gray-100 dark:hover:bg-gray-800"
								on:click={() => startEditing(chat)}>{$i18n.t('Edit')}</button
							>
						{/if}
					</div>
				</div>
			{/each}
		</div>
	{/if}
</div>
