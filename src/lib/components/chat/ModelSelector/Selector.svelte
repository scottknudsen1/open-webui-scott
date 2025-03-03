<script lang="ts">
	import { Select } from 'bits-ui';

	import { flyAndScale } from '$lib/utils/transitions';
	import { createEventDispatcher, onMount, getContext, tick } from 'svelte';

	import ChevronDown from '$lib/components/icons/ChevronDown.svelte';
	import Check from '$lib/components/icons/Check.svelte';
	import Search from '$lib/components/icons/Search.svelte';
	import Star from '$lib/components/icons/Star.svelte';

	import { cancelOllamaRequest, deleteModel, getOllamaVersion, pullModel } from '$lib/apis/ollama';

	import { user, MODEL_DOWNLOAD_POOL, models, starredModels } from '$lib/stores';
	import { writable, derived }  from 'svelte/store';
	import { toast } from 'svelte-sonner';
	import { capitalizeFirstLetter, getModels, splitStream } from '$lib/utils';
	import Tooltip from '$lib/components/common/Tooltip.svelte';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	export let value = '';
	export let placeholder = 'Select a model';
	export let searchEnabled = true;
	export let searchPlaceholder = 'Search a model';

	export let items = [{ value: 'mango', label: 'Mango' }];
	starredModels.set({
    	mango: false,  // Example structure
	});

	let searchValue = '';
	let ollamaVersion = null;

	//const sortedItems = derived([items, starredModels], ([$items, $starredModels]) => {
	//	console.log("Starred Models:", $starredModels);
    //    return $items.slice().sort((a, b) => {
    //        const aIsStarred = $starredModels[a.value] === true;
    //        const bIsStarred = $starredModels[b.value] === true;
    //        if (aIsStarred && !bIsStarred) {
    //            return -1; // `a` comes before `b`
    //        } else if (!aIsStarred && bIsStarred) {
    //            return 1; // `b` comes before `a`
    //        }
    //        return 0; // Maintain original order if both are starred or not starred
    //    });
    //});

	$: filteredItems = <butt>
		? items.filter((item) => item.value.includes(searchValue.toLowerCase()))
		: items;

	//const combinedItems = derived(
    //    [items, starredModels, searchValue],
    //    ([$items, $starredModels, $searchValue]) => {
	//		console.log("Items:", $items);  // Debugging output
    //    	console.log("Starred Models:", $starredModels);  // Debugging output
    //    	console.log("Search Value:", $searchValue);  // Debugging output
    //        const sortedItems = $items.slice().sort((a, b) => {
    //            const aIsStarred = $starredModels[a.value] === true;
    //            const bIsStarred = $starredModels[b.value] === true;
    //            return (bIsStarred - aIsStarred) || a.label.localeCompare(b.label);
    //        });
    //        if ($searchValue) {
    //            return sortedItems.filter(item => item.value.toLowerCase().includes($searchValue.toLowerCase()));
    //        }
	//		console.log("sorted items: " + sortedItems);
    //        return sortedItems;
    //    }
    //);

	const pullModelHandler = async () => {
		const sanitizedModelTag = searchValue.trim().replace(/^ollama\s+(run|pull)\s+/, '');

		console.log($MODEL_DOWNLOAD_POOL);
		if ($MODEL_DOWNLOAD_POOL[sanitizedModelTag]) {
			toast.error(
				$i18n.t(`Model '{{modelTag}}' is already in queue for downloading.`, {
					modelTag: sanitizedModelTag
				})
			);
			return;
		}
		if (Object.keys($MODEL_DOWNLOAD_POOL).length === 3) {
			toast.error(
				$i18n.t('Maximum of 3 models can be downloaded simultaneously. Please try again later.')
			);
			return;
		}

		const res = await pullModel(localStorage.token, sanitizedModelTag, '0').catch((error) => {
			toast.error(error);
			return null;
		});

		if (res) {
			const reader = res.body
				.pipeThrough(new TextDecoderStream())
				.pipeThrough(splitStream('\n'))
				.getReader();

			while (true) {
				try {
					const { value, done } = await reader.read();
					if (done) break;

					let lines = value.split('\n');

					for (const line of lines) {
						if (line !== '') {
							let data = JSON.parse(line);
							console.log(data);
							if (data.error) {
								throw data.error;
							}
							if (data.detail) {
								throw data.detail;
							}

							if (data.id) {
								MODEL_DOWNLOAD_POOL.set({
									...$MODEL_DOWNLOAD_POOL,
									[sanitizedModelTag]: {
										...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
										requestId: data.id,
										reader,
										done: false
									}
								});
								console.log(data);
							}

							if (data.status) {
								if (data.digest) {
									let downloadProgress = 0;
									if (data.completed) {
										downloadProgress = Math.round((data.completed / data.total) * 1000) / 10;
									} else {
										downloadProgress = 100;
									}

									MODEL_DOWNLOAD_POOL.set({
										...$MODEL_DOWNLOAD_POOL,
										[sanitizedModelTag]: {
											...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
											pullProgress: downloadProgress,
											digest: data.digest
										}
									});
								} else {
									toast.success(data.status);

									MODEL_DOWNLOAD_POOL.set({
										...$MODEL_DOWNLOAD_POOL,
										[sanitizedModelTag]: {
											...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
											done: data.status === 'success'
										}
									});
								}
							}
						}
					}
				} catch (error) {
					console.log(error);
					if (typeof error !== 'string') {
						error = error.message;
					}

					toast.error(error);
					// opts.callback({ success: false, error, modelName: opts.modelName });
				}
			}

			if ($MODEL_DOWNLOAD_POOL[sanitizedModelTag].done) {
				toast.success(
					$i18n.t(`Model '{{modelName}}' has been successfully downloaded.`, {
						modelName: sanitizedModelTag
					})
				);

				models.set(await getModels(localStorage.token));
			} else {
				toast.error('Download canceled');
			}

			delete $MODEL_DOWNLOAD_POOL[sanitizedModelTag];

			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL
			});
		}
	};

	onMount(async () => {
		ollamaVersion = await getOllamaVersion(localStorage.token).catch((error) => false);
	});

	const cancelModelPullHandler = async (model: string) => {
		const { reader, requestId } = $MODEL_DOWNLOAD_POOL[model];
		if (reader) {
			await reader.cancel();

			await cancelOllamaRequest(localStorage.token, requestId);
			delete $MODEL_DOWNLOAD_POOL[model];
			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL
			});
			await deleteModel(localStorage.token, model);
			toast.success(`${model} download has been canceled`);
		}
	};

	function toggleStar(value) {
        starredModels.update(currentStars => {
            const newState = { ...currentStars };
            newState[value] = !newState[value];
            return newState;
        });
    }
</script>

<Select.Root
	{items}
	onOpenChange={async () => {
		searchValue = '';
		window.setTimeout(() => document.getElementById('model-search-input')?.focus(), 0);
	}}
	selected={items.find((item) => item.value === value) ?? ''}
	onSelectedChange={(selectedItem) => {
		value = selectedItem.value;
	}}
>
	<Select.Trigger class="relative w-full" aria-label={placeholder}>
		<Select.Value
			class="flex text-left px-0.5 outline-none bg-transparent truncate text-lg font-semibold placeholder-gray-400  focus:outline-none"
			{placeholder}
		/>
		<ChevronDown className="absolute end-2 top-1/2 -translate-y-[45%] size-3.5" strokeWidth="2.5" />
	</Select.Trigger>
	<Select.Content
		class=" z-40 w-full rounded-lg  bg-white dark:bg-gray-900 dark:text-white shadow-lg border border-gray-300/30 dark:border-gray-700/50  outline-none"
		transition={flyAndScale}
		sideOffset={4}
	>
		<slot>
			{#if searchEnabled}
				<div class="flex items-center gap-2.5 px-5 mt-3.5 mb-3">
					<Search className="size-4" strokeWidth="2.5" />

					<input
						id="model-search-input"
						bind:value={searchValue}
						class="w-full text-sm bg-transparent outline-none"
						placeholder={searchPlaceholder}
					/>
				</div>

				<hr class="border-gray-100 dark:border-gray-800" />
			{/if}

			<div class="px-3 my-2 max-h-72 overflow-y-auto">
				{#each filteredItems as item}
					<Select.Item
						class="flex w-full font-medium line-clamp-1 select-none items-center rounded-button py-2 pl-3 pr-1.5 text-sm  text-gray-700 dark:text-gray-100  outline-none transition-all duration-75 hover:bg-gray-100 dark:hover:bg-gray-850 rounded-lg cursor-pointer data-[highlighted]:bg-muted"
						value={item.value}
						label={item.label}
					>
						<div class="flex items-center gap-2">
							<div class="line-clamp-1">
								{item.label}

								<span class=" text-xs font-medium text-gray-600 dark:text-gray-400"
									>{item.info?.details?.parameter_size ?? ''}</span
								>
							</div>

							<!-- {JSON.stringify(item.info)} -->

							{#if item.info.external}
								<Tooltip content={item.info?.source ?? 'External'}>
									<div class=" mr-2">
										<svg
											xmlns="http://www.w3.org/2000/svg"
											viewBox="0 0 16 16"
											fill="currentColor"
											class="size-3"
										>
											<path
												fill-rule="evenodd"
												d="M8.914 6.025a.75.75 0 0 1 1.06 0 3.5 3.5 0 0 1 0 4.95l-2 2a3.5 3.5 0 0 1-5.396-4.402.75.75 0 0 1 1.251.827 2 2 0 0 0 3.085 2.514l2-2a2 2 0 0 0 0-2.828.75.75 0 0 1 0-1.06Z"
												clip-rule="evenodd"
											/>
											<path
												fill-rule="evenodd"
												d="M7.086 9.975a.75.75 0 0 1-1.06 0 3.5 3.5 0 0 1 0-4.95l2-2a3.5 3.5 0 0 1 5.396 4.402.75.75 0 0 1-1.251-.827 2 2 0 0 0-3.085-2.514l-2 2a2 2 0 0 0 0 2.828.75.75 0 0 1 0 1.06Z"
												clip-rule="evenodd"
											/>
										</svg>
									</div>
								</Tooltip>
							{:else}
								<Tooltip
									content={`${
										item.info?.details?.quantization_level
											? item.info?.details?.quantization_level + ' '
											: ''
									}${item.info.size ? `(${(item.info.size / 1024 ** 3).toFixed(1)}GB)` : ''}`}
								>
									<div class=" mr-2">
										<svg
											xmlns="http://www.w3.org/2000/svg"
											fill="none"
											viewBox="0 0 24 24"
											stroke-width="1.5"
											stroke="currentColor"
											class="w-4 h-4"
										>
											<path
												stroke-linecap="round"
												stroke-linejoin="round"
												d="m11.25 11.25.041-.02a.75.75 0 0 1 1.063.852l-.708 2.836a.75.75 0 0 0 1.063.853l.041-.021M21 12a9 9 0 1 1-18 0 9 9 0 0 1 18 0Zm-9-3.75h.008v.008H12V8.25Z"
											/>
										</svg>
									</div>
								</Tooltip>
							{/if}
						</div>

						{#if value === item.value}
							<div class="ml-auto">
								<Check />
							</div>
						{/if}
						<button
							class="ml-auto p-2"
							on:click|stopPropagation={() => toggleStar(item.value)}
						>
							<Star filled={$starredModels[item.value] || false} />
						</button>
					</Select.Item>
				{:else}
					<div>
						<div class="block px-3 py-2 text-sm text-gray-700 dark:text-gray-100">
							No results found
						</div>
					</div>
				{/each}

				{#if !(searchValue.trim() in $MODEL_DOWNLOAD_POOL) && searchValue && ollamaVersion && $user.role === 'admin'}
					<button
						class="flex w-full font-medium line-clamp-1 select-none items-center rounded-button py-2 pl-3 pr-1.5 text-sm text-gray-700 dark:text-gray-100 outline-none transition-all duration-75 hover:bg-gray-100 dark:hover:bg-gray-850 rounded-lg cursor-pointer data-[highlighted]:bg-muted"
						on:click={() => {
							pullModelHandler();
						}}
					>
						Pull "{searchValue}" from Ollama.com
					</button>
				{/if}

				{#each Object.keys($MODEL_DOWNLOAD_POOL) as model}
					<div
						class="flex w-full justify-between font-medium select-none rounded-button py-2 pl-3 pr-1.5 text-sm text-gray-700 dark:text-gray-100 outline-none transition-all duration-75 rounded-lg cursor-pointer data-[highlighted]:bg-muted"
					>
						<div class="flex">
							<div class="-ml-2 mr-2.5 translate-y-0.5">
								<svg
									class="size-4"
									viewBox="0 0 24 24"
									fill="currentColor"
									xmlns="http://www.w3.org/2000/svg"
									><style>
										.spinner_ajPY {
											transform-origin: center;
											animation: spinner_AtaB 0.75s infinite linear;
										}
										@keyframes spinner_AtaB {
											100% {
												transform: rotate(360deg);
											}
										}
									</style><path
										d="M12,1A11,11,0,1,0,23,12,11,11,0,0,0,12,1Zm0,19a8,8,0,1,1,8-8A8,8,0,0,1,12,20Z"
										opacity=".25"
									/><path
										d="M10.14,1.16a11,11,0,0,0-9,8.92A1.59,1.59,0,0,0,2.46,12,1.52,1.52,0,0,0,4.11,10.7a8,8,0,0,1,6.66-6.61A1.42,1.42,0,0,0,12,2.69h0A1.57,1.57,0,0,0,10.14,1.16Z"
										class="spinner_ajPY"
									/></svg
								>
							</div>

							<div class="flex flex-col self-start">
								<div class="line-clamp-1">
									Downloading "{model}" {'pullProgress' in $MODEL_DOWNLOAD_POOL[model]
										? `(${$MODEL_DOWNLOAD_POOL[model].pullProgress}%)`
										: ''}
								</div>

								{#if 'digest' in $MODEL_DOWNLOAD_POOL[model] && $MODEL_DOWNLOAD_POOL[model].digest}
									<div class="-mt-1 h-fit text-[0.7rem] dark:text-gray-500 line-clamp-1">
										{$MODEL_DOWNLOAD_POOL[model].digest}
									</div>
								{/if}
							</div>
						</div>

						<div class="mr-2 translate-y-0.5">
							<Tooltip content="Cancel">
								<button
									class="text-gray-800 dark:text-gray-100"
									on:click={() => {
										cancelModelPullHandler(model);
									}}
								>
									<svg
										class="w-4 h-4 text-gray-800 dark:text-white"
										aria-hidden="true"
										xmlns="http://www.w3.org/2000/svg"
										width="24"
										height="24"
										fill="currentColor"
										viewBox="0 0 24 24"
									>
										<path
											stroke="currentColor"
											stroke-linecap="round"
											stroke-linejoin="round"
											stroke-width="2"
											d="M6 18 17.94 6M18 18 6.06 6"
										/>
									</svg>
								</button>
							</Tooltip>
						</div>
					</div>
				{/each}
			</div>
		</slot>
	</Select.Content>
</Select.Root>
