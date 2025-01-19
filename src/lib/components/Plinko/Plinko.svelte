<script lang="ts">
  import { onMount } from 'svelte';
  import { writable, get } from 'svelte/store';
  import { plinkoEngine } from '$lib/stores/game';
  import CircleNotch from 'phosphor-svelte/lib/CircleNotch';
  import type { Action } from 'svelte/action';
  import BinsRow from './BinsRow.svelte';
  import LastWins from './LastWins.svelte';
  import PlinkoEngine from './PlinkoEngine';

  // Plinko engine config
  const { WIDTH, HEIGHT } = PlinkoEngine;

  /**
   * The local session balance used for Plinko bets.
   * This is separate from your Firestore or parent "real" balance.
   */
  const sessionBalance = writable<number>(0);

  // Track basic session info
  let sessionId: string | null = null;
  let userId: string | null = null;
  let originalBalance = 0; // so we can compute netProfit = final - original

  // Store reference to the Plinko engine instance
  const initPlinko: Action<HTMLCanvasElement> = (node) => {
    $plinkoEngine = new PlinkoEngine(node);
    $plinkoEngine.start();

    return {
      destroy: () => {
        $plinkoEngine?.stop();
      },
    };
  };

  /**
   * Listen for 'INIT_SESSION' message from parent.
   * The parent (mini app) should send something like:
   * 
   * iframe.contentWindow.postMessage(
   *   {
   *     type: 'INIT_SESSION',
   *     userId: 'abc123',
   *     sessionBalance: 100,
   *     sessionId: 'someSessionDocId' // optional
   *   },
   *   'https://miniappre.vercel.app'
   * );
   */
  onMount(() => {
    function handleMessage(event: MessageEvent) {
      // Check origin for security
      if (event.origin !== 'https://miniappre.vercel.app') {
        console.warn('Ignored message from untrusted origin:', event.origin);
        return;
      }

      const { type, userId: incomingUserId, sessionBalance: incomingBal, sessionId: incomingSessionId } = event.data || {};

      if (type === 'INIT_SESSION') {
        console.log('Received INIT_SESSION from parent:', { incomingUserId, incomingBal, incomingSessionId });

        userId = incomingUserId;
        sessionId = incomingSessionId; // If you need to track a session doc in Firestore
        sessionBalance.set(incomingBal);
        originalBalance = incomingBal;
      }
    }

    window.addEventListener('message', handleMessage);

    return () => {
      window.removeEventListener('message', handleMessage);
    };
  });

  /**
   * END_SESSION: Called when the user finishes Plinko or navigates away.
   * 
   * - finalBalance = get(sessionBalance)
   * - netProfit = finalBalance - originalBalance
   * - Then we postMessage back to the parent (mini app).
   */
  function endSession() {
    const finalBalance = get(sessionBalance);
    const netProfit = finalBalance - originalBalance;

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

      <!-- Canvas that runs the PlinkoEngine -->
      <canvas
        use:initPlinko
        width={WIDTH}
        height={HEIGHT}
        class="absolute inset-0 h-full w-full"
      ></canvas>
    </div>

    <!-- Display your bins, session stats, etc. -->
    <BinsRow />
  </div>

  <!-- Optional last-wins display -->
  <div class="absolute right-[5%] top-1/2 -translate-y-1/2">
    <LastWins />
  </div>

  <!-- Example button to end the session -->
  <div class="absolute left-[5%] bottom-[5%]">
    <button
      class="bg-red-500 text-white px-4 py-2 rounded"
      on:click={endSession}
    >
      End Session
    </button>
  </div>
</div>
