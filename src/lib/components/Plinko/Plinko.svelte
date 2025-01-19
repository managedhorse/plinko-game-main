<script lang="ts">
  import { onMount } from 'svelte';
  import { writable, get } from 'svelte/store';
  import { plinkoEngine } from '$lib/stores/game';
  import CircleNotch from 'phosphor-svelte/lib/CircleNotch';
  import type { Action } from 'svelte/action';
  import BinsRow from './BinsRow.svelte';
  import LastWins from './LastWins.svelte';
  import PlinkoEngine from './PlinkoEngine';

  const { WIDTH, HEIGHT } = PlinkoEngine;

  // The local session balance used for Plinko bets (separate from Firestore/parent)
  const sessionBalance = writable<number>(0);

  // Track if we've already ended the session, so we don’t end it multiple times
  let sessionEnded = false;

  // We store basic session info
  let sessionId: string | null = null;
  let userId: string | null = null;
  let originalBalance = 0; // so netProfit = (final - original)

  // Initialize the Plinko engine
  const initPlinko: Action<HTMLCanvasElement> = (node) => {
    $plinkoEngine = new PlinkoEngine(node);
    $plinkoEngine.start();

    return {
      destroy: () => {
        $plinkoEngine?.stop();
      },
    };
  };

  // Called by the parent via postMessage({ type: 'INIT_SESSION', ... })
  // to set up the local session data
  onMount(() => {
    function handleMessage(event: MessageEvent) {
      // For security, accept messages only if the origin is your parent domain
      if (!event.origin.includes('miniappre.vercel.app')) {
        console.warn('Ignored message from untrusted origin:', event.origin);
        return;
      }

      const { type, userId: incomingUserId, sessionBalance: incomingBal, sessionId: incomingSessionId } = event.data || {};

      if (type === 'INIT_SESSION') {
        console.log('Received INIT_SESSION from parent:', { incomingUserId, incomingBal, incomingSessionId });

        userId = incomingUserId;
        sessionId = incomingSessionId; 
        sessionBalance.set(incomingBal);
        originalBalance = incomingBal;
      }
    }

    window.addEventListener('message', handleMessage);

    // ====== AUTO-END SESSION ON UNLOAD OR VISIBILITY ======

    function handleBeforeUnload() {
      endSession(); 
    }
    function handleVisibilityChange() {
      // If the document is hidden (user switches app/tab),
      // you can attempt to end the session immediately
      if (document.hidden) {
        endSession();
      }
    }

    window.addEventListener('beforeunload', handleBeforeUnload);
    document.addEventListener('visibilitychange', handleVisibilityChange);

    // Cleanup when component is destroyed
    return () => {
      window.removeEventListener('message', handleMessage);
      window.removeEventListener('beforeunload', handleBeforeUnload);
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    };
  });

  /**
   * END_SESSION: Called when the user finishes Plinko or navigates away.
   *
   * netProfit = finalBalance - originalBalance
   * We then postMessage back to the parent with { type: 'END_SESSION', netProfit }.
   */
  function endSession() {
    // Avoid ending multiple times
    if (sessionEnded) return;
    sessionEnded = true;

    const finalBalance = get(sessionBalance);
    const netProfit = finalBalance - originalBalance;

    // Post message to the parent (React page)
    window.parent.postMessage(
      {
        type: 'END_SESSION',
        netProfit,
        sessionId // if applicable
      },
      'https://miniappre.vercel.app'
    );

    console.log('Sent END_SESSION to parent:', { netProfit, sessionId });
  }
</script>

<div class="relative bg-gray-900">
  <div class="mx-auto flex h-full flex-col px-4 pb-4" style:max-width={`${WIDTH}px`}>
    <div class="relative w-full" style:aspect-ratio={`${WIDTH} / ${HEIGHT}`}>
      {#if $plinkoEngine === null}
        <div class="absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2">
          <CircleNotch class="size-20 animate-spin text-slate-600" weight="bold" />
        </div>
      {/if}

      <!-- The Plinko canvas -->
      <canvas
        use:initPlinko
        width={WIDTH}
        height={HEIGHT}
        class="absolute inset-0 h-full w-full"
      ></canvas>
    </div>

    <BinsRow />
  </div>

  <!-- Optional side display for last wins, etc. -->
  <div class="absolute right-[5%] top-1/2 -translate-y-1/2">
    <LastWins />
  </div>

  <!-- Example button to manually end the session -->
  <div class="absolute left-[5%] bottom-[5%]">
    <button
      class="bg-red-500 text-white px-4 py-2 rounded"
      on:click={endSession}
    >
      End Session
    </button>
  </div>
</div>
