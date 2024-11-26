<script lang="ts">
	import { getBackendConfig, getModelFilterConfig, updateModelFilterConfig } from '$lib/apis';
	import { getSignUpEnabledStatus, toggleSignUpEnabledStatus } from '$lib/apis/auths';
	import { getUserPermissions, updateUserPermissions } from '$lib/apis/users';
	import { compareUserModels } from '$lib/utils';

	import { onMount, getContext } from 'svelte';
	import { models, config } from '$lib/stores';
	import Switch from '$lib/components/common/Switch.svelte';
	import { setDefaultModels } from '$lib/apis/configs';
	import { getUsersModels, getUsers, setUsersModels } from '../../../apis/users';
	import { user } from '../../../stores';

	const i18n = getContext('i18n');

	export let saveHandler: Function;

	let defaultModelId = '';

	let whitelistEnabled = false;
	let whitelistModels = [''];
	let permissions = {
		chat: {
			deletion: true,
			edit: true,
			temporary: true
		}
	};

	let users = [''];
	let chatDeletion = true;
	let chatEdit = true;
	let chatTemporary = true;

	let openDropdowns = {};
	let userSelections = {};
	let usersModels = {};

	let userWhitelistEnabled = false;

	onMount(async () => {
		users = await getUsers(localStorage.token);
		permissions = await getUserPermissions(localStorage.token);
		usersModels = await getUsersModels(localStorage.token);

		chatDeletion = permissions?.chat?.deletion ?? true;
		chatEdit = permissions?.chat?.editing ?? true;
		chatTemporary = permissions?.chat?.temporary ?? true;

		const res = await getModelFilterConfig(localStorage.token);
		if (res) {
			whitelistEnabled = res.enabled;
			whitelistModels = res.models.length > 0 ? res.models : [''];
		}

		if (usersModels) userSelections = { ...usersModels };

		defaultModelId = $config.default_models ? $config?.default_models.split(',')[0] : '';
	});

	function toggleOption(userId, option) {
		if (userSelections[userId].includes(option))
			userSelections[userId] = userSelections[userId].filter((item) => item !== option);
		else userSelections[userId] = [...(userSelections[userId] || []), option];

		userSelections = userSelections; // Trigger reactivity
	}

	function toggleDropdown(userId) {
		// Toggle the current dropdown
		openDropdowns[userId] = !openDropdowns[userId];

		// Close all other dropdowns by setting their state to false
		Object.keys(openDropdowns).forEach((key) => {
			if (key !== userId.toString()) openDropdowns[key] = false;
		});

		openDropdowns = { ...openDropdowns }; // Trigger reactivity
	}

	// 	// Close all other dropdowns
	// 	openDropdowns = Object.fromEntries(
	// 	Object.keys(openDropdowns).map((key) => [key, false]) // Set all dropdowns to false
	// );

	// // Toggle the clicked dropdown
	// openDropdowns[userId] = !openDropdowns[userId];
	// openDropdowns = openDropdowns; // Trigger reactivity

	let showGlobalDropdown = false;

	// Function to handle selecting a model for all users
	function selectModelForAllUsers(modelId) {}

	// Close dropdowns when clicking outside
	function handleClickOutside(event, userId) {
		const dropdown = event.target.closest('.dropdown-container');
		const globalDropdown = event.target.closest('.global-dropdown-container');

		// Close individual user dropdowns if clicked outside
		if (!dropdown && openDropdowns[userId]) {
			openDropdowns[userId] = false;
			openDropdowns = openDropdowns; // Trigger reactivity
		}

		// Close the global dropdown if clicked outside it
		if (!globalDropdown && showGlobalDropdown) {
			showGlobalDropdown = false;
		}
	}

	let val = null;
	let compute;
	$: compute = compareUserModels(usersModels, userSelections);
</script>

<svelte:window
	on:click={(event) => {
		users.forEach((user) => handleClickOutside(event, user.id));
		handleClickOutside(event, null); // Handle global dropdown "outside click"
	}}
/>

<form
	class="flex flex-col h-full justify-between space-y-3 text-sm"
	on:submit|preventDefault={async () => {
		await setDefaultModels(localStorage.token, defaultModelId);
		await updateUserPermissions(localStorage.token, {
			chat: {
				deletion: chatDeletion,
				editing: chatEdit,
				temporary: chatTemporary
			}
		});
		await updateModelFilterConfig(localStorage.token, whitelistEnabled, whitelistModels);
		saveHandler();

		val = await config.set(await getBackendConfig());
		await setUsersModels(localStorage.token, compareUserModels(usersModels, userSelections));
	}}
>
	<div class="space-y-3 max-h-full">
		<div>
			<div class=" mb-2 text-sm font-medium">{$i18n.t('User Permissions')}</div>

			<div class="  flex w-full justify-between my-2 pr-2">
				<div class=" self-center text-xs font-medium">{$i18n.t('Allow Chat Deletion')}</div>

				<Switch bind:state={chatDeletion} />
			</div>

			<div class="  flex w-full justify-between my-2 pr-2">
				<div class=" self-center text-xs font-medium">{$i18n.t('Allow Chat Editing')}</div>

				<Switch bind:state={chatEdit} />
			</div>

			<div class="  flex w-full justify-between my-2 pr-2">
				<div class=" self-center text-xs font-medium">{$i18n.t('Allow Temporary Chat')}</div>

				<Switch bind:state={chatTemporary} />
			</div>
		</div>

		<!-- {JSON.stringify(usersModels ?? 'usersModels')}

		<hr class=" dark:border-gray-850 my-2" />

		{JSON.stringify(userSelections)}

		<hr class=" dark:border-gray-850 my-2" />

		{JSON.stringify(compute)} -->

		<div class="mt-2 space-y-3">
			<div>
				<div class="mb-2">
					<div class="flex justify-between items-center text-xs">
						<div class=" text-sm font-medium">{$i18n.t('Manage Models')}</div>
					</div>
				</div>
				<div class=" space-y-1 mb-3">
					<div class="mb-2">
						<div class="flex justify-between items-center text-xs">
							<div class=" text-xs font-medium">{$i18n.t('Default Model')}</div>
						</div>
					</div>
					<div class="flex-1 mr-2">
						<select
							class="w-full rounded-lg py-2 px-4 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-none"
							bind:value={defaultModelId}
							placeholder="Select a model"
						>
							<!-- all models here  -->
							<option value="" disabled selected>{$i18n.t('Select a model')}</option>
							{#each $models.filter((model) => model.id) as model}
								<option value={model.id} class="bg-gray-100 dark:bg-gray-700">{model.name}</option>
							{/each}
						</select>
					</div>
				</div>

				<div class=" space-y-1">
					<div class="mb-2">
						<div class="flex justify-between items-center text-xs my-3 pr-2">
							<div class=" text-xs font-medium">{$i18n.t('Model Whitelisting')}</div>

							<Switch bind:state={whitelistEnabled} />
						</div>
					</div>

					{#if whitelistEnabled}
						<div>
							<div class=" space-y-1.5">
								{#each whitelistModels as modelId, modelIdx}
									<div class="flex w-full">
										<div class="flex-1 mr-2">
											<select
												class="w-full rounded-lg py-2 px-4 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-none"
												bind:value={modelId}
												placeholder="Select a model"
											>
												<option value="" disabled selected>{$i18n.t('Select a model')}</option>
												{#each $models.filter((model) => model.id) as model}
													<option value={model.id} class="bg-gray-100 dark:bg-gray-700"
														>{model.name}</option
													>
												{/each}
											</select>
										</div>

										{#if modelIdx === 0}
											<button
												class="px-2.5 bg-gray-100 hover:bg-gray-200 text-gray-800 dark:bg-gray-900 dark:text-white rounded-lg transition"
												type="button"
												on:click={() => {
													if (whitelistModels.at(-1) !== '') {
														whitelistModels = [...whitelistModels, ''];
													}
												}}
											>
												<svg
													xmlns="http://www.w3.org/2000/svg"
													viewBox="0 0 16 16"
													fill="currentColor"
													class="w-4 h-4"
												>
													<path
														d="M8.75 3.75a.75.75 0 0 0-1.5 0v3.5h-3.5a.75.75 0 0 0 0 1.5h3.5v3.5a.75.75 0 0 0 1.5 0v-3.5h3.5a.75.75 0 0 0 0-1.5h-3.5v-3.5Z"
													/>
												</svg>
											</button>
										{:else}
											<button
												class="px-2.5 bg-gray-100 hover:bg-gray-200 text-gray-800 dark:bg-gray-900 dark:text-white rounded-lg transition"
												type="button"
												on:click={() => {
													whitelistModels.splice(modelIdx, 1);
													whitelistModels = whitelistModels;
												}}
											>
												<svg
													xmlns="http://www.w3.org/2000/svg"
													viewBox="0 0 16 16"
													fill="currentColor"
													class="w-4 h-4"
												>
													<path d="M3.75 7.25a.75.75 0 0 0 0 1.5h8.5a.75.75 0 0 0 0-1.5h-8.5Z" />
												</svg>
											</button>
										{/if}
									</div>
								{/each}
							</div>

							<div class="flex justify-end items-center text-xs mt-1.5 text-right">
								<div class=" text-xs font-medium">
									{whitelistModels.length}
									{$i18n.t('Model(s) Whitelisted')}
								</div>
							</div>
						</div>

						<!-- <div class="h-4"></div> -->
					{/if}

					<div class="mb-2">
						<div class="flex justify-between items-center text-xs my-3 pr-2">
							<div class=" text-xs font-medium">{$i18n.t('User Whitelisting')}</div>
						</div>
					</div>

					<!-- User Select-->
					<div class="grid grid-cols-2 gap-x-12 gap-y-4 p-2">
						<div
							class="absolute inset-y-0 w-px bg-gray-200 left-1/2 transform -translate-x-1/2 scale-x-50"
						></div>
						{#each users as user (user.id)}
							<div class="flex items-center space-x-4">
								<!-- User Name -->
								<div class="w-1/3">
									{user.name} <span class="text-xs text-gray-400">{`(${user.email})`}</span>
								</div>

								<!-- Dropdown Container -->
								<div class="dropdown-container relative w-2/3">
									<!-- Selected Count Button -->
									<button
										type="button"
										class="w-full p-2 text-left bg-gray-50 rounded flex justify-between items-center hover:border-gray-400"
										on:click|stopPropagation={() => toggleDropdown(user.id)}
									>
										<span>
											{userSelections[user.id]?.length || 0} models selected
										</span>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											fill="none"
											viewBox="0 0 24 24"
											stroke="currentColor"
											class="w-4 h-4 text-gray-500"
										>
											<path
												stroke-linecap="round"
												stroke-linejoin="round"
												stroke-width="2"
												d="M19 9l-7 7-7-7"
											/>
										</svg>
									</button>

									<!-- Dropdown Options -->
									{#if openDropdowns[user.id]}
										<div
											class="absolute top-11 w-full mt-1 max-h-48 overflow-y-auto bg-white border-gray-50 rounded shadow-lg z-10"
										>
											{#each $models.filter((model) => model.id) as model}
												<div
													class="p-2 cursor-pointer hover:bg-gray-100 flex items-center gap-2"
													class:bg-blue-50={userSelections[user.id]?.includes(model.id)}
													on:click|stopPropagation={() => toggleOption(user.id, model.id)}
												>
													<div
														class="w-4 h-4 border rounded flex items-center justify-center {userSelections[
															user.id
														]?.includes(model.id)
															? 'bg-blue-500 border-blue-500'
															: 'border-gray-300'}"
													>
														{#if userSelections[user.id]?.includes(model)}
															<span class="text-white text-xs">✓</span>
														{/if}
													</div>
													{model.name}
												</div>
											{/each}
										</div>
									{/if}
								</div>
							</div>
						{/each}
					</div>
				</div>
			</div>
		</div>
	</div>

	<div class="flex justify-end pt-3 text-sm font-medium">
		<button
			class="px-3.5 py-1.5 text-sm font-medium bg-black hover:bg-gray-900 text-white dark:bg-white dark:text-black dark:hover:bg-gray-100 transition rounded-full"
			type="submit"
		>
			{$i18n.t('Save')}
		</button>
	</div>
</form>

<!-- Global Select -->
<!-- <div class="mr-2 grid grid-cols-1 col-span-2 mb-6 global-dropdown-container relative">
	<div class="relative w-full">
		<button
			type="button"
			class="rounded-lg w-full p-2 text-left bg-gray-50 border-gray-300 rounded flex justify-between items-center hover:border-gray-400"
			on:click={() => (showGlobalDropdown = !showGlobalDropdown)}
		>
			<span>&nbsp;&nbsp;Select a model</span>
			<svg
				xmlns="http://www.w3.org/2000/svg"
				fill="none"
				viewBox="0 0 24 24"
				stroke="currentColor"
				class="w-4 h-4 text-gray-500"
			>
				<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
			</svg>
		</button>

		{#if showGlobalDropdown}
			<div
				class="absolute top-11 w-full mt-1 max-h-48 overflow-y-auto bg-white border-gray-300 rounded shadow-lg z-10"
			>
				{#each $models.filter((model) => model.id) as model}
					<div
						class="p-2 cursor-pointer hover:bg-gray-100 flex items-center gap-2"
						on:click={() => selectModelForAllUsers(model.id)}
					>
						<div class="w-4 h-4 border rounded flex items-center justify-center">

							{#if users.every((user) => userSelections[user.id]?.includes(model.id))}
								<span class="text-blue-500 text-xs">✓</span>
							{/if}
						</div>
						{model.name}
					</div>
				{/each}
			</div>
		{/if}
	</div>
</div> -->
