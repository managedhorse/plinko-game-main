<script lang="ts">
  import { Select } from '$lib/components/ui';
  import { autoBetIntervalMs, rowCountOptions } from '$lib/constants/game';
  import {
    balance,
    betAmount,
    betAmountOfExistingBalls,
    plinkoEngine,
    riskLevel,
    rowCount,
  } from '$lib/stores/game';
  import { isGameSettingsOpen, isLiveStatsOpen } from '$lib/stores/layout';
  import { BetMode, RiskLevel } from '$lib/types';
  import { flyAndScale } from '$lib/utils/transitions';
  import { Popover, Tooltip } from 'bits-ui';
  import ChartLine from 'phosphor-svelte/lib/ChartLine';
  import GearSix from 'phosphor-svelte/lib/GearSix';
  import Infinity from 'phosphor-svelte/lib/Infinity';
  import Question from 'phosphor-svelte/lib/Question';
  import type { FormEventHandler } from 'svelte/elements';
  import { twMerge } from 'tailwind-merge';

  let betMode: BetMode = $state(BetMode.MANUAL);

  let autoBetInput = $state(0);
  let autoBetsLeft: number | null = $state(null);
  let autoBetInterval: ReturnType<typeof setInterval> | null = $state(null);

  let isBetAmountNegative = $derived($betAmount < 0);
  let isBetExceedBalance = $derived($betAmount > $balance);
  let isAutoBetInputNegative = $derived(autoBetInput < 0);

  let isDropBallDisabled = $derived(
    $plinkoEngine === null || isBetAmountNegative || isBetExceedBalance || isAutoBetInputNegative,
  );

  let hasOutstandingBalls = $derived(Object.keys($betAmountOfExistingBalls).length > 0);

  const handleBetAmountFocusOut: FormEventHandler<HTMLInputElement> = (e) => {
    const v = parseFloat(e.currentTarget.value.trim());
    if (isNaN(v)) {
      $betAmount = -1;
      $betAmount = 0;
    } else {
      $betAmount = v;
    }
  };

  const handleAutoBetInputFocusOut: FormEventHandler<HTMLInputElement> = (e) => {
    const v = parseInt(e.currentTarget.value.trim());
    if (isNaN(v)) {
      autoBetInput = -1;
      autoBetInput = 0;
    } else {
      autoBetInput = v;
    }
  };

  function resetAutoBetInterval() {
    if (autoBetInterval !== null) {
      clearInterval(autoBetInterval);
      autoBetInterval = null;
    }
  }

  function autoBetDropBall() {
    if (isBetExceedBalance) {
      resetAutoBetInterval();
      return;
    }
    if (autoBetsLeft === null) {
      $plinkoEngine?.dropBall();
      return;
    }
    if (autoBetsLeft > 0) {
      $plinkoEngine?.dropBall();
      autoBetsLeft -= 1;
    }
    if (autoBetsLeft === 0) {
      resetAutoBetInterval();
    }
  }

  function handleBetClick() {
    if (betMode === BetMode.MANUAL) {
      $plinkoEngine?.dropBall();
    } else if (autoBetInterval === null) {
      autoBetsLeft = autoBetInput === 0 ? null : autoBetInput;
      autoBetInterval = setInterval(autoBetDropBall, autoBetIntervalMs);
    } else {
      resetAutoBetInterval();
    }
  }

  const betModes = [
    { value: BetMode.MANUAL, label: 'Manual' },
    { value: BetMode.AUTO, label: 'Auto' },
  ];
  const riskLevels = [
    { value: RiskLevel.LOW, label: 'Low' },
    { value: RiskLevel.MEDIUM, label: 'Medium' },
    { value: RiskLevel.HIGH, label: 'High' },
  ];
  const rowCounts = rowCountOptions.map((v) => ({ value: v, label: v.toString() }));
</script>

<div class="flex flex-col gap-2 bg-slate-700 p-3 lg:max-w-80">

  <!-- Top row: Manual/Auto & Drop Ball -->
  <div class="flex gap-2">
    <div class="w-1/2 flex gap-1 rounded-full bg-slate-900 p-1">
      {#each betModes as { value, label }}
        <button
          class={twMerge(
            'w-full rounded-full py-2 text-sm font-medium text-white transition disabled:opacity-50',
            betMode === value && 'bg-slate-600'
          )}
          disabled={autoBetInterval !== null}
          onclick={() => (betMode = value)}
        >{label}</button>
      {/each}
    </div>
    <button
      class={twMerge(
        'w-1/2 rounded-md py-3 font-semibold transition-colors',
        autoBetInterval !== null
          ? 'bg-yellow-500 hover:bg-yellow-400 active:bg-yellow-600 text-slate-900'
          : 'bg-green-500 hover:bg-green-400 active:bg-green-600 text-slate-900',
        isDropBallDisabled && 'bg-neutral-600 text-neutral-400'
      )}
      onclick={handleBetClick}
      disabled={isDropBallDisabled}
    >
      {#if betMode === BetMode.MANUAL}
        Drop Ball
      {:else if autoBetInterval === null}
        Start Autobet
      {:else}
        Stop Autobet
      {/if}
    </button>
  </div>

  <!-- Bet Amount -->
  <div class="relative">
    <label for="betAmount" class="text-sm font-medium text-slate-300">Bet Amount</label>
    <div class="flex">
      <div class="relative flex-1">
        <input
          id="betAmount"
          type="number"
          min="0"
          step="0.01"
          value={$betAmount}
          onfocusout={handleBetAmountFocusOut}
          disabled={autoBetInterval !== null}
          class={twMerge(
            'w-full rounded-l-md border-2 border-slate-600 bg-slate-900 py-2 pl-7 pr-2 text-sm text-white',
            (isBetAmountNegative || isBetExceedBalance) && 'border-red-500'
          )}
        />
        <div class="absolute left-3 top-2 text-slate-500">$</div>
      </div>
      <button
        class="bg-slate-600 px-4 font-bold text-white"
        disabled={autoBetInterval !== null}
        onclick={() => ($betAmount = parseFloat(($betAmount / 2).toFixed(2)))}
      >1/2</button>
      <button
        class="bg-slate-600 px-4 text-white"
        disabled={autoBetInterval !== null}
        onclick={() => ($betAmount = parseFloat(($betAmount * 2).toFixed(2)))}
      >2×</button>
    </div>
    {#if isBetAmountNegative}
      <p class="absolute text-xs text-red-400">Must be ≥ 0.</p>
    {:else if isBetExceedBalance}
      <p class="absolute text-xs text-red-400">Can't exceed balance!</p>
    {/if}
  </div>

  <!-- Risk & Rows -->
  <div class="flex gap-2">
    <div class="flex-1">
      <label for="riskLevel" class="text-sm font-medium text-slate-300">Risk</label>
      <Select
        id="riskLevel"
        bind:value={$riskLevel}
        items={riskLevels}
        disabled={hasOutstandingBalls || autoBetInterval !== null}
      />
    </div>
    <div class="flex-1">
      <label for="rowCount" class="text-sm font-medium text-slate-300">Rows</label>
      <Select
        id="rowCount"
        bind:value={$rowCount}
        items={rowCounts}
        disabled={hasOutstandingBalls || autoBetInterval !== null}
      />
    </div>
  </div>

  <!-- Bottom row: Live Stats + (when AUTO) Number of Bets -->
  <div class="mt-auto">
    <div class="flex items-center gap-4">
      <Tooltip.Root openDelay={0} closeOnPointerDown={false}>
        <Tooltip.Trigger asChild let:builder>
          <button
            use:builder.action
            {...builder}
            onclick={() => ($isLiveStatsOpen = !$isLiveStatsOpen)}
            class={twMerge(
              'rounded-full p-2 text-slate-300 transition hover:bg-slate-600 active:bg-slate-500',
              $isLiveStatsOpen && 'text-slate-100'
            )}
          >
            <ChartLine class="size-6" weight="bold" />
          </button>
        </Tooltip.Trigger>
        <Tooltip.Content
          transition={flyAndScale}
          sideOffset={4}
          class="z-30 max-w-lg rounded-md bg-white p-3 text-sm font-medium text-gray-950 drop-shadow-xl"
        >
          <Tooltip.Arrow />
          <p>{$isLiveStatsOpen ? 'Close' : 'Open'} Live Stats</p>
        </Tooltip.Content>
      </Tooltip.Root>

      {#if betMode === BetMode.AUTO}
        <div class="relative w-1/2">
          <input
            id="autoBetInputBottom"
            type="number"
            min="0"
            value={autoBetInterval === null ? autoBetInput : autoBetsLeft ?? 0}
            onfocusout={handleAutoBetInputFocusOut}
            disabled={autoBetInterval !== null}
            class={twMerge(
              'w-full rounded-md border-2 border-slate-600 bg-slate-900 py-2 pl-3 pr-8 text-sm text-white',
              isAutoBetInputNegative && 'border-red-500'
            )}
          />
          {#if autoBetInput === 0}
            <Infinity class="absolute right-3 top-3 size-4 text-slate-400" weight="bold" />
          {/if}
        </div>
      {/if}
    </div>
  </div>
</div>
