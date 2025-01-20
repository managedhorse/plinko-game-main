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

  // The local session balance used for Plinko bets
  const sessionBalance = writable<number>(0);

  let sessionEnded = false;
  let sessionId: string | null = null;
  let userId: string | null = null;
  let originalBalance = 0;
  
  // 1) A local boolean that tracks if we've received INIT_SESSION yet
  let isLoadingSession = true;

  const initPlinko: Action<HTMLCanvasElement> = (node) => {
    $plinkoEngine = new PlinkoEngine(node);
    $plinkoEngine.start();

    return {
      destroy: () => {
        $plinkoEngine?.stop();
      },
    };
  };

  onMount(() => {
    function handleMessage(event: MessageEvent) {
      console.log('Child received message from origin:', event.origin);
      const { type, userId: incomingUserId, sessionBalance: incomingBal, sessionId: incomingSessionId } = event.data || {};

      if (type === 'INIT_SESSION') {
        console.log('Received INIT_SESSION from parent:', { incomingUserId, incomingBal, incomingSessionId });

        userId = incomingUserId;
        sessionId = incomingSessionId;
        
        // Update local store and the old balance store
        sessionBalance.set(incomingBal);
        originalBalance = incomingBal;
        oldBalance.set(incomingBal);

        // 2) Now we have the user's real balance. Stop showing the spinner:
        isLoadingSession = false;
      }
    }

    window.addEventListener('message', handleMessage);

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

  function endSession() {
    if (sessionEnded) return;
    sessionEnded = true;

    const finalBalance = get(sessionBalance);
    const netProfit = finalBalance - originalBalance;

    window.parent.postMessage(
      {
        type: 'END_SESSION',
        netProfit,
        sessionId
      },
      'https://miniappre.vercel.app'
    );

    console.log('Sent END_SESSION to parent:', { netProfit, sessionId });
  }
</script>

<!-- 3) If isLoadingSession is true, show spinner; otherwise show the Plinko UI -->
{#if isLoadingSession}
  <!-- A very simple loading UI -->
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
      <BinsRow />
    </div>

    <div class="absolute right-[5%] top-1/2 -translate-y-1/2">
      <LastWins />
    </div>
  </div>
{/if}
