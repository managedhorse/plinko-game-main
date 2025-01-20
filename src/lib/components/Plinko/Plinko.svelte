<script lang="ts">
  import { onMount } from 'svelte';
  import { writable, get } from 'svelte/store';
  import { plinkoEngine, balance as oldBalance } from '$lib/stores/game';
  import CircleNotch from 'phosphor-svelte/lib/CircleNotch';
  import type { Action } from 'svelte/action';
  import BinsRow from './BinsRow.svelte';
  import LastWins from './LastWins.svelte';
  import PlinkoEngine from './PlinkoEngine';

  const { WIDTH, HEIGHT } = PlinkoEngine;

  // Local store for the Plinko session’s visible balance:
  const sessionBalance = writable<number>(0);

  // Basic flags and variables to manage session lifecycle:
  let sessionEnded = false;
  let sessionId: string | null = null;
  let userId: string | null = null;
  let originalBalance = 0; // so final netProfit = finalBalance - originalBalance

  // Show spinner until the parent sends INIT_SESSION
  let isLoadingSession = true;

  // Initialize the Plinko engine when our canvas mounts:
  const initPlinko: Action<HTMLCanvasElement> = (node) => {
    $plinkoEngine = new PlinkoEngine(node);
    $plinkoEngine.start();
    return {
      destroy: () => {
        $plinkoEngine?.stop();
      },
    };
  };

  // Call this after each spin to store partial results in localStorage
  function registerSpinProfit(spinProfit: number) {
    // "plinko_unapplied" is the key we store partial results under
    const storedJson = localStorage.getItem('plinko_unapplied') || '[]';
    const data = JSON.parse(storedJson);

    // Push a small object with sessionId, spinProfit, timestamp:
    data.push({
      sessionId,
      spinProfit,
      timestamp: Date.now()
    });

    localStorage.setItem('plinko_unapplied', JSON.stringify(data));

    // Also update this session’s in-memory balance:
    sessionBalance.update(b => b + spinProfit);
  }

  // Example function: after each ball drop completes, call onSpinComplete
  // to record the netProfit for that spin.
  // You can call this from your Plinko engine callback or a "Drop Ball" handler, etc.
  function onSpinComplete(spinProfit: number) {
    registerSpinProfit(spinProfit);
  }

  // Called once on component mount:
  onMount(() => {
    function handleMessage(event: MessageEvent) {
      // The parent will post { type: 'INIT_SESSION', userId, sessionBalance, sessionId }
      const { type, userId: incomingUserId, sessionBalance: incomingBal, sessionId: incomingSessionId } = event.data || {};

      if (type === 'INIT_SESSION') {
        userId = incomingUserId;
        sessionId = incomingSessionId;
        originalBalance = incomingBal;

        sessionBalance.set(incomingBal);
        // Also update the older "balance" store from game.ts if needed:
        oldBalance.set(incomingBal);

        // We can now show the main UI:
        isLoadingSession = false;
        console.log('INIT_SESSION arrived:', { userId, sessionId, originalBalance });
      }
    }

    window.addEventListener('message', handleMessage);

    // If the user forcibly closes or hides the tab, we run endSession:
    function handleBeforeUnload() {
      endSession();
    }
    function handleVisibilityChange() {
      if (document.hidden) {
        endSession();
      }
    }

    window.addEventListener('beforeunload', handleBeforeUnload);
    document.addEventListener('visibilitychange', handleVisibilityChange);

    // Cleanup
    return () => {
      window.removeEventListener('message', handleMessage);
      window.removeEventListener('beforeunload', handleBeforeUnload);
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    };
  });

  // Called once the user leaves or the session is forcibly closed:
  function endSession() {
    if (sessionEnded) return;
    sessionEnded = true;

    const finalBalance = get(sessionBalance);
    const netProfit = finalBalance - originalBalance;

    // Post final netProfit back to parent:
    window.parent.postMessage(
      {
        type: 'END_SESSION',
        netProfit,
        sessionId
      },
      'https://miniappre.vercel.app'
    );

    console.log('END_SESSION posted with netProfit:', netProfit);
  }
</script>

<!-- If still loading, show spinner; else show the Plinko UI. -->
{#if isLoadingSession}
  <div class="flex h-screen w-screen items-center justify-center bg-gray-900 text-white">
    <CircleNotch class="size-12 animate-spin" weight="bold" />
    <span class="ml-3 text-lg">Loading session...</span>
  </div>
{:else}
  <div class="relative bg-gray-900">
    <div class="mx-auto flex h-full flex-col px-4 pb-4" style:max-width={`${WIDTH}px`}>
      <div class="relative w-full" style:aspect-ratio={`${WIDTH} / ${HEIGHT}`}>
        {#if $plinkoEngine === null}
          <div class="absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2">
            <CircleNotch class="size-20 animate-spin text-slate-600" weight="bold" />
          </div>
        {/if}

        <canvas
          use:initPlinko
          width={WIDTH}
          height={HEIGHT}
          class="absolute inset-0 h-full w-full"
        ></canvas>
      </div>

      <!-- Example Bins, can call onSpinComplete(...) for partial results. -->
      <BinsRow />
    </div>

    <!-- Optional last wins display. -->
    <div class="absolute right-[5%] top-1/2 -translate-y-1/2">
      <LastWins />
    </div>
  </div>
{/if}
