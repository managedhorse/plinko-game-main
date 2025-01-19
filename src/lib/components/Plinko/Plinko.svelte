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

  // onMount for message handling, etc.
  onMount(() => {
    function handleMessage(event: MessageEvent) {
      console.log('Child received message from origin:', event.origin);
      // Skip origin check for now, or loosen it

      const { type, userId: incomingUserId, sessionBalance: incomingBal, sessionId: incomingSessionId } = event.data || {};
      if (type === 'INIT_SESSION') {
        console.log('Received INIT_SESSION from parent:', { incomingUserId, incomingBal, incomingSessionId });

        userId = incomingUserId;
        sessionId = incomingSessionId;
        
        // 1) set your new local store
        sessionBalance.set(incomingBal);
        originalBalance = incomingBal;

        // 2) also update the old "balance" store from game.ts
        oldBalance.set(incomingBal);
      }
    }

    window.addEventListener('message', handleMessage);

    // ====== AUTO-END SESSION ON UNLOAD OR VISIBILITY ======
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

    return () => {
      window.removeEventListener('message', handleMessage);
      window.removeEventListener('beforeunload', handleBeforeUnload);
      document.removeEventListener('visibilitychange', handleVisibilityChange);
    };
  });

  /**
   * END_SESSION: Called when the user finishes Plinko or navigates away.
   * netProfit = finalBalance - originalBalance
   * Then we postMessage back to the parent with { type: 'END_SESSION', netProfit }.
   */
  function endSession() {
    if (sessionEnded) return;
    sessionEnded = true;

    const finalBalance = get(sessionBalance);
    const netProfit = finalBalance - originalBalance;

    // Post message back to the parent (React page)
    window.parent.postMessage(
      {
        type: 'END_SESSION',
        netProfit,
        sessionId
      },
      'https://miniappre.vercel.app' // The target origin can remain your main domain
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

  <!-- Button to manually end the session -->
  <div class="absolute left-[5%] bottom-[5%]">
    <button
      class="bg-red-500 text-white px-4 py-2 rounded"
      on:click={endSession}
    >
      End Session
    </button>
  </div>
</div>