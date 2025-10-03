<script lang="ts">
	import { ModeToggle } from '$lib/ui/mode-toggle';
	import { navigationConfig } from '.';
	import { siteConfig } from '$lib/metadata';
	import { page } from '$app/state';
	import { LogIn, Menu, X } from 'lucide-svelte';
	import { goto } from '$app/navigation';

	let isMobileMenuOpen = $state(false);

	function toggleMobileMenu() {
		isMobileMenuOpen = !isMobileMenuOpen;
	}

	function closeMobileMenu() {
		isMobileMenuOpen = false;
	}

	function handleLogin() {
		goto('/login');
	}
</script>

<header
	class="sticky top-0 z-50 w-full border-b border-border/40 bg-background/95 backdrop-blur supports-[backdrop-filter]:bg-background/60"
>
	<nav class="container mx-auto flex h-14 w-full items-center justify-between px-4 md:px-6">
		<!-- Logo -->
		<div class="flex items-center">
			<a
				href={siteConfig.home}
				class="text-md flex items-center font-bold md:text-lg"
				data-sveltekit-preload-data
			>
				{#if siteConfig.logo}
					<img src={siteConfig.logo} alt="Logo" class="h-8 w-8 md:h-10 md:w-10" />
				{/if}
				<span class="ml-2 truncate">{siteConfig.name}</span>
			</a>
		</div>

		<!-- Desktop Navigation -->
		<div class="hidden items-center gap-6 md:flex">
			{#each navigationConfig as item (item.href)}
				{@const isActive = page.url.pathname === item.href}
				<a
					href={item.href}
					class="smooth-transition text-sm font-medium {isActive
						? 'text-gradient'
						: 'animated-underline'}"
					data-sveltekit-preload-data
					data-sveltekit-prefetch
				>
					{item.label}
				</a>
			{/each}
			<div class="ml-4 flex items-center gap-2">
				<button
					onclick={handleLogin}
					class="flex items-center gap-2 px-3 py-2 rounded-lg transition-colors hover:bg-surface-200 dark:hover:bg-surface-700"
					aria-label="Toggle theme"
				>
					<LogIn class="w-5 h-5" />
				</button>
				<ModeToggle />
			</div>
		</div>

		<!-- Mobile Menu Controls -->
		<div class="flex items-center gap-2 md:hidden">
			<button
				onclick={handleLogin}
				class="flex items-center gap-2 px-3 py-2 rounded-lg transition-colors hover:bg-surface-200 dark:hover:bg-surface-700"
				aria-label="Toggle theme"
			>
				<LogIn class="w-5 h-5" />
			</button>
			<ModeToggle />
			<button
				onclick={toggleMobileMenu}
				class="inline-flex items-center justify-center rounded-md p-2 text-foreground hover:bg-accent hover:text-accent-foreground focus:ring-2 focus:ring-primary focus:outline-none focus:ring-inset"
				aria-expanded={isMobileMenuOpen}
				aria-label="Toggle navigation menu"
			>
				{#if isMobileMenuOpen}
					<X class="h-5 w-5" />
				{:else}
					<Menu class="h-5 w-5" />
				{/if}
			</button>
		</div>
	</nav>

	<!-- Mobile Navigation Menu -->
	{#if isMobileMenuOpen}
		<div class="border-t border-border/40 bg-background/95 backdrop-blur md:hidden">
			<div class="container mx-auto space-y-2 px-4 py-3">
				{#each navigationConfig as item (item.href)}
					{@const isActive = page.url.pathname === item.href}
					<a
						href={item.href}
						class="block rounded-md px-3 py-2 text-base font-medium transition-colors {isActive
							? 'bg-accent text-accent-foreground'
							: 'text-foreground hover:bg-accent hover:text-accent-foreground'}"
						data-sveltekit-preload-data
						data-sveltekit-prefetch
						onclick={closeMobileMenu}
					>
						{item.label}
					</a>
				{/each}
			</div>
		</div>
	{/if}
</header>
