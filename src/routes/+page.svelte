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

<svelte:head>
  <!-- Slackey font import -->
  <link
    href="https://fonts.googleapis.com/css2?family=Slackey&display=swap"
    rel="stylesheet"
  />
</svelte:head>

<svelte:window onbeforeunload={writeBalanceToLocalStorage} />

<div
  class="relative flex min-h-screen w-full flex-col bg-cover bg-center"
  style="background-image: url({grassBg});"
>
  <nav
    class="sticky top-0 z-10 w-full bg-gradient-to-r from-[#ff9a9e] via-[#fad0c4] to-[#ff9a9e] px-6 py-2 drop-shadow-2xl"
  >
    <div class="mx-auto flex h-8 max-w-7xl items-center justify-between">
      <span
        class="text-1xl font-slackey text-gray-900"
        style="font-family: 'Slackey', cursive;"
      >
        Plink Mianus
      </span>
      <Balance />
    </div>
  </nav>

  <div class="flex-1 px-4 md:px-8 lg:px-12">
    <div class="mx-auto mt-6 min-w-[300px] max-w-xl md:mt-10 lg:max-w-7xl">
      <div class="flex flex-col-reverse overflow-hidden rounded-lg bg-white/20 backdrop-blur-sm lg:flex-row">
        <Sidebar />
        <div class="flex-1 p-2">
          <Plinko />
        </div>
      </div>
    </div>
  </div>

  <SettingsWindow />
  <LiveStatsWindow />
</div>

<style>
  :global(.font-slackey) {
    font-family: "Slackey", cursive;
  }
</style>
