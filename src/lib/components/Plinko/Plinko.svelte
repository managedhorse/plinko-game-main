<script lang="ts">
  import { onMount } from 'svelte';
  import { writable, get } from 'svelte/store';
  import { plinkoEngine, balance as oldBalance } from '$lib/stores/game'; 
  import CircleNotch from 'phosphor-svelte/lib/CircleNotch';
  import type { Action } from 'svelte/action';
  import BinsRow from './BinsRow.svelte';
  import LastWins from './LastWins.svelte';
  import PlinkoEngine from './PlinkoEngine';
  import { doc, getDoc, updateDoc } from 'firebase/firestore';
  import { db } from '$lib/firebase';  // Adjust the path as necessary

  const { WIDTH, HEIGHT } = PlinkoEngine;

  // Local store for the user's Plinko balance
  const sessionBalance = writable<number>(0);

  let userId: string | null = null;

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

  // Function to fetch the plinkoBalance from Firestore
  async function fetchPlinkoBalance(userId: string) {
    const userRef = doc(db, 'telegramUsers', userId.toString());
    try {
      const userSnap = await getDoc(userRef);
      if (userSnap.exists()) {
        const userData = userSnap.data();
        const plinkoBal = userData.plinkoBalance || 0;
        sessionBalance.set(plinkoBal);
        oldBalance.set(plinkoBal);
      } else {
        console.error('User not found in Firestore for ID', userId);
      }
    } catch (err) {
      console.error('Error fetching plinko balance:', err);
    }
  }

  onMount(() => {
    window.addEventListener('message', handleMessage);

    function handleMessage(event: MessageEvent) {
  console.log('Received message:', event.data, 'from', event.origin); 
  const { type, userId: incomingUserId } = event.data || {};
  if (type === 'INIT_SESSION' && incomingUserId) {
    console.log('INIT_SESSION received with userId:', incomingUserId);
    userId = incomingUserId;
    fetchPlinkoBalance(incomingUserId);
  }
}

  window.addEventListener('message', handleMessage);


    const interval = setInterval(async () => {
      if (!userId) return;
      const currentBal = get(sessionBalance);
      const userRef = doc(db, 'telegramUsers', userId.toString());
      try {
        await updateDoc(userRef, { plinkoBalance: currentBal });
        console.log('Updated plinkoBalance in Firestore:', currentBal);
      } catch (err) {
        console.error('Error updating plinkoBalance:', err);
      }
    }, 60000); // Update every 1 minute

    return () => {
      window.removeEventListener('message', handleMessage);
      clearInterval(interval);
    };
  });
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
</div>