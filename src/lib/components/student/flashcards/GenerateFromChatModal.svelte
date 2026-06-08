<!-- GenerateFromChatModal.svelte -->
<!--
  Sprint 3, Step 1 — in-place flashcard generation from a chat.
  Mounted in the student tutor chat navbar (ChatNavbar.svelte). Source is
  locked to the current chat; the user picks model + generation params.
  Reuses POST /api/v1/flashcards/generate; no backend change.
-->
<script lang="ts">
	import { getContext } from 'svelte';
	import { goto } from '$app/navigation';
	import { toast } from 'svelte-sonner';
	import type { Writable } from 'svelte/store';

	import Modal from '$lib/components/common/Modal.svelte';
	import Sparkles from '$lib/components/icons/Sparkles.svelte';
	import XMark from '$lib/components/icons/XMark.svelte';
	import { generateFlashcards } from '$lib/apis/flashcards';
	import { models } from '$lib/stores';

	interface I18n { t: (key: string) => string }
	const i18n = getContext<Writable<I18n>>('i18n');

	export let show = false;
	// The chat object as ChatNavbar receives it: { id, chat: { title, history: {...} } }
	export let chat: any = null;
	// Pre-selected model id from the chat's model selector.
	export let initialModel = '';

	let selectedModel = '';
	let cardCount = 8;
	let languageHint = '';
	let difficultyHint: '' | 'beginner' | 'intermediate' | 'advanced' = '';
	let customTitle = '';
	let generating = false;

	$: if (show) initForm();
	function initForm() {
		selectedModel = initialModel || $models[0]?.id || '';
		cardCount = 8;
		languageHint = '';
		difficultyHint = '';
		customTitle = '';
	}

	// Same extraction logic as Flashcards.svelte uses for a loaded chat — keeps
	// the message shape consistent across both entry points.
	function extractMessages(chatObj: any): { role: string; content: string }[] {
		const msgMap =
			chatObj?.chat?.history?.messages ?? chatObj?.chat?.history ?? {};
		if (typeof msgMap !== 'object' || Array.isArray(msgMap)) return [];
		return Object.values(msgMap)
			.filter(
				(m: any) =>
					(m.role === 'user' || m.role === 'assistant') && m.content?.trim()
			)
			.map((m: any) => ({ role: m.role, content: m.content.trim() }));
	}

	async function submit() {
		if (!selectedModel) {
			toast.error($i18n.t('Please select a model first'));
			return;
		}
		const messages = extractMessages(chat);
		if (!messages.length) {
			toast.error($i18n.t('This chat has no messages yet'));
			return;
		}

		const title = customTitle.trim() || chat?.chat?.title || $i18n.t('Chat set');
		const sourceLabel = `${$i18n.t('Chat')}: ${chat?.chat?.title ?? $i18n.t('Untitled')}`;

		generating = true;
		try {
			const token = localStorage.getItem('token') ?? '';
			const newSet = await generateFlashcards(token, {
				messages,
				model: selectedModel,
				title,
				source_label: sourceLabel,
				card_count: cardCount,
				language: languageHint.trim() || undefined,
				difficulty: difficultyHint || undefined
			});
			toast.success(
				`"${newSet.title}" — ${newSet.card_count} ${$i18n.t('cards')}`,
				{
					action: {
						label: $i18n.t('View set'),
						onClick: () => goto('/student/flashcards')
					}
				}
			);
			show = false;
		} catch (e: any) {
			toast.error(e?.message ?? $i18n.t('Failed to generate flashcards'));
		} finally {
			generating = false;
		}
	}
</script>

<Modal bind:show size="md">
	<div class="px-6 py-5">
		<div class="flex items-start justify-between mb-5">
			<div class="flex items-center gap-2">
				<Sparkles className="w-5 h-5 text-blue-600 dark:text-blue-400" strokeWidth="1.75" />
				<div>
					<h2 class="text-base font-semibold text-gray-900 dark:text-white">
						{$i18n.t('Generate flashcards from this chat')}
					</h2>
					<p class="text-xs text-gray-500 dark:text-gray-400">
						{$i18n.t('Cards will be extracted from the current conversation.')}
					</p>
				</div>
			</div>
			<button
				on:click={() => (show = false)}
				class="p-1.5 rounded-md text-gray-400 hover:text-gray-700 hover:bg-gray-100 dark:hover:text-gray-200 dark:hover:bg-gray-800 transition-colors"
				aria-label={$i18n.t('Close')}
			>
				<XMark className="w-4 h-4" strokeWidth="2" />
			</button>
		</div>

		<div class="space-y-4">
			<!-- Title -->
			<div>
				<label
					for="fc-chat-title"
					class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1.5"
				>
					{$i18n.t('Set name')}
				</label>
				<input
					id="fc-chat-title"
					type="text"
					bind:value={customTitle}
					placeholder={chat?.chat?.title ?? $i18n.t('Auto-filled from the chat title')}
					class="w-full rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-900 text-gray-900 dark:text-white px-3 py-2 text-sm placeholder:text-gray-400 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
				/>
			</div>

			<!-- Model -->
			<div>
				<label
					for="fc-chat-model"
					class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1.5"
				>
					{$i18n.t('Model')}
				</label>
				<select
					id="fc-chat-model"
					bind:value={selectedModel}
					class="w-full rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-900 text-gray-900 dark:text-white px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
				>
					{#each $models as m}
						<option value={m.id}>{m.name ?? m.id}</option>
					{/each}
				</select>
			</div>

			<!-- Card count -->
			<div>
				<div class="flex items-center justify-between mb-1.5">
					<label
						for="fc-chat-count"
						class="block text-sm font-medium text-gray-700 dark:text-gray-300"
					>
						{$i18n.t('Number of cards')}
					</label>
					<span class="text-sm tabular-nums text-gray-700 dark:text-gray-200 font-medium">
						{cardCount}
					</span>
				</div>
				<input
					id="fc-chat-count"
					type="range"
					min="3"
					max="10"
					step="1"
					bind:value={cardCount}
					class="w-full accent-blue-600"
				/>
				<div class="flex justify-between text-[10px] text-gray-400 dark:text-gray-500 px-0.5 mt-0.5 tabular-nums">
					<span>3</span><span>10</span>
				</div>
			</div>

			<!-- Language + difficulty -->
			<div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
				<div>
					<label
						for="fc-chat-language"
						class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1.5"
					>
						{$i18n.t('Language')}
					</label>
					<input
						id="fc-chat-language"
						type="text"
						bind:value={languageHint}
						maxlength="50"
						placeholder={$i18n.t('Match the source language')}
						class="w-full rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-900 text-gray-900 dark:text-white px-3 py-2 text-sm placeholder:text-gray-400 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
					/>
				</div>
				<div>
					<label
						for="fc-chat-difficulty"
						class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-1.5"
					>
						{$i18n.t('Difficulty')}
					</label>
					<select
						id="fc-chat-difficulty"
						bind:value={difficultyHint}
						class="w-full rounded-lg border border-gray-200 dark:border-gray-600 bg-white dark:bg-gray-900 text-gray-900 dark:text-white px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
					>
						<option value="">{$i18n.t('Auto')}</option>
						<option value="beginner">{$i18n.t('Beginner')}</option>
						<option value="intermediate">{$i18n.t('Intermediate')}</option>
						<option value="advanced">{$i18n.t('Advanced')}</option>
					</select>
				</div>
			</div>
		</div>

		<div class="flex justify-end gap-2 mt-6 pt-4 border-t border-gray-200 dark:border-gray-700">
			<button
				on:click={() => (show = false)}
				disabled={generating}
				class="px-4 py-2 rounded-lg text-sm font-medium text-gray-700 dark:text-gray-200 hover:bg-gray-100 dark:hover:bg-gray-800 disabled:opacity-50 transition-colors"
			>
				{$i18n.t('Cancel')}
			</button>
			<button
				on:click={submit}
				disabled={generating || !selectedModel}
				class="inline-flex items-center gap-2 px-4 py-2 rounded-lg bg-gray-900 hover:bg-gray-800 disabled:bg-gray-300 dark:bg-white dark:hover:bg-gray-100 dark:disabled:bg-gray-600 dark:text-gray-900 text-white text-sm font-medium disabled:cursor-not-allowed transition-colors"
			>
				{#if generating}
					<svg class="animate-spin h-4 w-4" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
						<circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" />
						<path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8z" />
					</svg>
					{$i18n.t('Generating…')}
				{:else}
					<Sparkles className="w-4 h-4" strokeWidth="1.75" />
					{$i18n.t('Generate')}
				{/if}
			</button>
		</div>
	</div>
</Modal>
