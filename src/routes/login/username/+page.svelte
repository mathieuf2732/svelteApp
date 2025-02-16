<script lang="ts">
	import { goto } from "$app/navigation";
	import AuthCheck from "$lib/components/AuthCheck.svelte";
	import { db, user, userData } from "$lib/firebase";
	import { doc, getDoc, writeBatch } from "firebase/firestore";

	const USERNAME_VALIDATION_REGEX = /^(?=[a-zA-Z0-9._]{3,16}$)(?!.*[_.]{2})[^_.].*[^_.]$/;

	let inputValue = $state("");
	let username = $derived(inputValue.toLocaleLowerCase());
	let loading = $state(false);
	let isAvailable = $state(false);
	let debounceTimer: NodeJS.Timeout;

	let isValid = $derived(
		username.length > 2 && username.length < 16 && USERNAME_VALIDATION_REGEX.test(username)
	);
	let isTouched = $derived(username.length > 0);
	let isTaken = $derived(isValid && !isAvailable && !loading);

	function checkAvailability() {
		isAvailable = false;
		clearTimeout(debounceTimer);
		loading = true;

		debounceTimer = setTimeout(async () => {
			const ref = doc(db, "usernames", username);
			const exists = await getDoc(ref).then((doc) => doc.exists());

			isAvailable = !exists;
			loading = false;
		}, 500);
	}

	async function confirmUsername(e: SubmitEvent) {
		e.preventDefault();
		const batch = writeBatch(db);
		batch.set(doc(db, "usernames", username), { uid: $user?.uid });
		batch.set(doc(db, "users", $user!.uid), {
			username,
			photoURL: $user?.photoURL ?? null,
			published: true,
			bio: "",
			links: []
		});

		await batch.commit();
		goto("/login/photo");
	}
</script>

<AuthCheck>
	{#if $userData?.username}
		<p class="text-lg">
			Your username is <span class="text-success font-bold">@{$userData.username}</span>
		</p>
	{:else}
		<h1>Choose Your Username</h1>
		<form class="w-2/5" onsubmit={confirmUsername}>
			<input
				type="text"
				placeholder="Username"
				class="input w-full lowercase"
				bind:value={inputValue}
				oninput={checkAvailability}
				class:input-error={!isValid && isTouched}
				class:input-warning={isTaken}
				class:input-success={isAvailable && isValid && !loading}
			/>
			<div class="my-4 min-h-16 px-8 w-full">
				{#if loading && isValid}
					<p class="text-secondary">Checking availability of @{username}...</p>
				{/if}

				{#if !isValid && isTouched}
					<p class="text-error text-sm">Must be 3-15 characters long, alphanumeric only</p>
				{/if}

				{#if isValid && !isAvailable && !loading}
					<p class="text-warning text-sm">
						@{username} is not available
					</p>
				{/if}

				{#if isAvailable && isValid}
					<button class="btn btn-success" onsubmit={confirmUsername}>
						Confirm username @{username}
					</button>
				{/if}
			</div>
		</form>
	{/if}
</AuthCheck>
