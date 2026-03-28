<script lang="ts">
	import { createEventDispatcher, getContext } from 'svelte';
	const dispatch = createEventDispatcher();
	const i18n = getContext('i18n');

	import XMark from '$lib/components/icons/XMark.svelte';
	import AdvancedParams from '../Settings/Advanced/AdvancedParams.svelte';
	import Valves from '$lib/components/chat/Controls/Valves.svelte';
	import FileItem from '$lib/components/common/FileItem.svelte';
	import Collapsible from '$lib/components/common/Collapsible.svelte';

	import { onMount } from 'svelte';
	import { getModelsConfig } from '$lib/apis/configs';
	import { getModelById } from '$lib/apis/models';
	import { user, settings } from '$lib/stores';
	export let models = [];
	export let chatFiles = [];
	export let params = {};
	export let embed = false;

	// ---------------------------------------------------------------------------
	// PARAM INHERITANCE LOGIC
	// ---------------------------------------------------------------------------
	// Parameters flow through four layers, each overriding the previous:
	//
	//   1. Admin global defaults  — set in Admin → Settings → Advanced Params.
	//                               Stored in DEFAULT_MODEL_PARAMS on the server.
	//                               Fetched via GET /api/configs/models (admin only).
	//
	//   2. Model-specific params  — set in Admin → Workspace → Models → [model].
	//                               Stored per-model in the DB. The /api/models list
	//                               endpoint strips these out before sending to the
	//                               client, so we must fetch them via getModelById().
	//
	//   3. User account settings  — set in the user's own Settings → Advanced Params.
	//                               Available in the $settings store.
	//
	//   4. Chat-specific params   — set in the chat Controls panel (right sidebar).
	//                               Stored in `params` and sent directly with each request.
	//                               These win over everything else.
	//
	// `inheritedParams` (layers 1-3) is passed to AdvancedParams so it can:
	//   a) Show the effective inherited value next to each "Default" button.
	//   b) Pre-fill the value when the user clicks "Default" to enable a param.
	//
	// The backend (main.py) applies the same priority independently, so even when
	// a param is left on "Default" in the chat UI, the correct inherited value
	// is still forwarded to the model.
	// ---------------------------------------------------------------------------

	// Layer 1: Admin global defaults (admin users only — others get an empty object).
	let adminDefaultParams: Record<string, any> = {};

	// Layer 2: Model-specific params fetched via getModelById because /api/models
	// deliberately strips `params` from the model list response.
	let modelSpecificParams: Record<string, any> = {};

	// Tracks the last fetched model ID to avoid redundant API calls when the
	// models array reference changes but the selected model is the same.
	let lastFetchedModelId: string | null = null;

	onMount(async () => {
		if ($user?.role === 'admin') {
			const config = await getModelsConfig(localStorage.token).catch(() => null);
			adminDefaultParams = config?.DEFAULT_MODEL_PARAMS ?? {};
		}
	});

	// Re-fetch layer 2 only when the selected model actually changes.
	// `fetchId` is captured at call time so the async callback can verify the model
	// hasn't changed while the request was in-flight (prevents out-of-order param leakage).
	$: if (models[0]?.id && models[0].id !== lastFetchedModelId) {
		lastFetchedModelId = models[0].id;
		const fetchId = models[0].id;
		getModelById(localStorage.token, fetchId)
			.then((m: any) => {
				// Discard stale responses if the user switched models mid-flight.
				if (fetchId === models[0]?.id) {
					modelSpecificParams = m?.params ?? {};
				}
			})
			.catch(() => {
				if (fetchId === models[0]?.id) {
					modelSpecificParams = {};
				}
			});
	} else if (!models[0]?.id) {
		lastFetchedModelId = null;
		modelSpecificParams = {};
	}

	// Merged layers 1-3. Layer 3 (user settings) wins over layer 2 (model) wins over layer 1 (admin).
	// Chat-specific params (layer 4) live in `params` and are not part of this object.
	$: inheritedParams = {
		...adminDefaultParams,   // layer 1 — lowest priority
		...modelSpecificParams,  // layer 2
		...($settings?.params ?? {}) // layer 3 — highest inherited priority
	} as Record<string, any>;

	// User's global system prompt (Settings → General). Stored at $settings.system, not $settings.params.system.
	// Typed separately because the settings store type doesn't declare `system` as a known field.
	$: userGlobalSystem = ($settings as Record<string, any>)?.system as string | undefined;

	// Persist collapsible section open/close state
	const getOpen = (key: string, fallback = true): boolean => {
		const v = localStorage.getItem(`chatControls.${key}`);
		return v !== null ? v === 'true' : fallback;
	};
	const setOpen = (key: string) => (open: boolean) => {
		localStorage.setItem(`chatControls.${key}`, String(open));
	};

	let showFiles = getOpen('files');
	let showValves = getOpen('valves', false);
	let showSystemPrompt = getOpen('systemPrompt');
	let showAdvancedParams = getOpen('advancedParams');
</script>

<div class=" dark:text-white">
	{#if !embed}
		<div class=" flex items-center justify-between dark:text-gray-100 mb-2">
			<div class=" text-md self-center font-primary">{$i18n.t('Controls')}</div>
			<button
				class="self-center"
				aria-label={$i18n.t('Close chat controls')}
				on:click={() => {
					dispatch('close');
				}}
			>
				<XMark className="size-3.5" />
			</button>
		</div>
	{/if}

	{#if $user?.role === 'admin' || ($user?.permissions.chat?.controls ?? true)}
		<div class=" dark:text-gray-200 text-sm py-0.5 px-0.5">
			{#if chatFiles.length > 0}
				<Collapsible
					title={$i18n.t('Files')}
					bind:open={showFiles}
					onChange={setOpen('files')}
					buttonClassName="w-full"
				>
					<div class="flex flex-col gap-1 mt-1.5" slot="content">
						{#each chatFiles as file, fileIdx}
							<FileItem
								className="w-full"
								item={file}
								edit={true}
								url={file?.url ? file.url : null}
								name={file.name}
								type={file.type}
								size={file?.size}
								dismissible={true}
								small={true}
								on:dismiss={() => {
									// Remove the file from the chatFiles array

									chatFiles.splice(fileIdx, 1);
									chatFiles = chatFiles;
								}}
								on:click={() => {
									console.log(file);
								}}
							/>
						{/each}
					</div>
				</Collapsible>

				<hr class="my-2 border-gray-50 dark:border-gray-700/10" />
			{/if}

			{#if $user?.role === 'admin' || ($user?.permissions.chat?.valves ?? true)}
				<Collapsible
					bind:open={showValves}
					onChange={setOpen('valves')}
					title={$i18n.t('Valves')}
					buttonClassName="w-full"
				>
					<div class="text-sm" slot="content">
						<Valves show={showValves} />
					</div>
				</Collapsible>

				<hr class="my-2 border-gray-50 dark:border-gray-700/10" />
			{/if}

			{#if $user?.role === 'admin' || ($user?.permissions.chat?.system_prompt ?? true)}
				<Collapsible
					title={$i18n.t('System Prompt')}
					bind:open={showSystemPrompt}
					onChange={setOpen('systemPrompt')}
					buttonClassName="w-full"
				>
					<div class="" slot="content">
						<textarea
							bind:value={params.system}
							class="w-full text-xs outline-hidden resize-vertical {$settings.highContrastMode
								? 'border-2 border-gray-300 dark:border-gray-700 rounded-lg bg-gray-50 dark:bg-gray-800 p-2.5'
								: 'py-1.5 bg-transparent'}"
							rows="4"
							placeholder={$i18n.t('Enter system prompt')}
						/>

						<!-- ---------------------------------------------------------------------------
						     Inherited system prompt preview (shown only when no custom prompt is set).
						     Priority matches Chat.svelte: user global settings beats model/admin params.
						     This is read-only context — the textarea above still overrides everything.
						     --------------------------------------------------------------------------- -->
						{#if !params.system}
							{#if userGlobalSystem}
								<!-- Layer 3: user's global system prompt (Settings → General) -->
								<div class="mt-1 text-xs opacity-40 flex flex-col gap-0.5">
									<span>Default (from your settings):</span>
									<span class="line-clamp-3 whitespace-pre-wrap">{userGlobalSystem}</span>
								</div>
							{:else if inheritedParams?.system}
								<!-- Layer 1/2: admin global or model-specific system prompt -->
								<div class="mt-1 text-xs opacity-40 flex flex-col gap-0.5">
									<span>Default (from model settings):</span>
									<span class="line-clamp-3 whitespace-pre-wrap">{inheritedParams.system}</span>
								</div>
							{/if}
						{/if}
					</div>
				</Collapsible>

				<hr class="my-2 border-gray-50 dark:border-gray-700/10" />
			{/if}

			{#if $user?.role === 'admin' || ($user?.permissions.chat?.params ?? true)}
				<Collapsible
					title={$i18n.t('Advanced Params')}
					bind:open={showAdvancedParams}
					onChange={setOpen('advancedParams')}
					buttonClassName="w-full"
				>
					<div class="text-sm mt-1.5" slot="content">
						<div>
							<AdvancedParams admin={$user?.role === 'admin'} custom={true} bind:params {inheritedParams} />
						</div>
					</div>
				</Collapsible>
			{/if}
		</div>
	{/if}
</div>
