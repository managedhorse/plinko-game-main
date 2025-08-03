<script lang="ts">
  import grassBg from '$lib/assets/grassbg.webp';
  import logo from '$lib/assets/logo.svg';
  import Balance from '$lib/components/Balance.svelte';
  import LiveStatsWindow from '$lib/components/LiveStatsWindow/LiveStatsWindow.svelte';
  import Plinko from '$lib/components/Plinko';
  import SettingsWindow from '$lib/components/SettingsWindow';
  import Sidebar from '$lib/components/Sidebar';
  import { setBalanceFromLocalStorage, writeBalanceToLocalStorage } from '$lib/utils/game';
  import GitHubLogo from 'phosphor-svelte/lib/GithubLogo';

  $effect(() => {
    setBalanceFromLocalStorage();
  });
</script>

<svelte:window onbeforeunload={writeBalanceToLocalStorage} />

<div class="relative flex min-h-dvh w-full flex-col">
  <nav class="sticky top-0 z-10 w-full bg-gray-700 px-5 drop-shadow-lg">
    <div class="mx-auto flex h-14 max-w-7xl items-center justify-between">
      <span class="text-2xl font-bold text-white">Plink Mianus</span>
      <div class="mx-auto">
        <Balance />
      </div>
    </div>
  </nav>

  <div class="flex-1 px-5">
    <div class="mx-auto mt-5 min-w-[300px] max-w-xl drop-shadow-xl md:mt-10 lg:max-w-7xl">
      <div class="flex flex-col-reverse overflow-hidden rounded-lg lg:w-full lg:flex-row">
        <Sidebar />
        <div class="flex-1">
          <Plinko />
        </div>
      </div>
    </div>
  </div>

  <SettingsWindow />
  <LiveStatsWindow />
</div>

<style>
  :global(body) {
    background: url("{grassBg}") center/cover no-repeat;
    /* ensure the page is at least full viewport height */
    min-height: 100vh;
  }
</style>
