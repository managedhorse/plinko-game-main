// plinko.svelte:
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
  import { db } from '$lib/firebase';  // Ensure this path is correct based on your project structure

  const { WIDTH, HEIGHT } = PlinkoEngine;

  // Local store for the user's Plinko balance
  const sessionBalance = writable<number>(0);

  let userId: string | null = null;

  // Initialize the Plinko engine
  const initPlinko: Action<HTMLCanvasElement> = (node) => {
    console.log("Initializing Plinko Engine...");
    $plinkoEngine = new PlinkoEngine(node);
    $plinkoEngine.start();

    return {
      destroy: () => {
        console.log("Destroying Plinko Engine...");
        $plinkoEngine?.stop();
      },
    };
  };

  // Function to fetch the plinkoBalance from Firestore
  async function fetchPlinkoBalance(userId: string) {
    console.log(`Fetching plinko balance for userId: ${userId}`);
    const userRef = doc(db, 'telegramUsers', userId);
    try {
      const userSnap = await getDoc(userRef);
      if (userSnap.exists()) {
        const userData = userSnap.data();
        const plinkoBal = userData.plinkoBalance || 0;
        console.log(`Fetched plinkoBalance for userId ${userId}: ${plinkoBal}`);
        sessionBalance.set(plinkoBal);
        oldBalance.set(plinkoBal);
        localStorage.setItem('plinkoBalance', plinkoBal.toString());
      } else {
        console.error('User not found in Firestore for ID', userId);
      }
    } catch (err) {
      console.error('Error fetching plinko balance:', err);
    }
  }

  onMount(() => {
    console.log("Component mounted. Setting up message listener and interval...");
    
    function handleMessage(event: MessageEvent) {
  console.log('Received message:', event.data, 'from', event.origin);
  const { type, userId: incomingUserId, requestId, amount } = event.data || {};
  
  if (type === 'USERID' && incomingUserId) {
    console.log('USERID received with:', incomingUserId);
    userId = incomingUserId;
    fetchPlinkoBalance(incomingUserId);
  }
  
  if (type === 'TRANSFER_BALANCE_REQUEST' && requestId) {
    console.log('Received TRANSFER_BALANCE_REQUEST with requestId:', requestId);
    if (userId) {
      const plinkoBal = get(sessionBalance);
      console.log(`Sending plinkoBalance to parent: ${plinkoBal}`);
      window.parent.postMessage(
        { type: 'TRANSFER_BALANCE_RESPONSE', requestId, plinkoBalance: plinkoBal },
        'https://miniappre.vercel.app' // Replace with parent's origin
      );
    } else {
      console.error("Cannot transfer balance: userId is not set.");
      window.parent.postMessage(
        { type: 'TRANSFER_BALANCE_ERROR', requestId, message: 'User ID not set.' },
        'https://miniappre.vercel.app'
      );
    }
  }
  // New handler for DEDUCT_BALANCE
  if (type === 'DEDUCT_BALANCE' && typeof amount === 'number') {
    console.log(`Deducting ${amount} from local Plinko balance.`);
    const currentBal = get(sessionBalance);
    const newBal = currentBal - amount;
    sessionBalance.set(newBal);
    oldBalance.set(newBal);
    localStorage.setItem('plinkoBalance', newBal.toString());
  }
}

    window.addEventListener('message', handleMessage);
    
    // Request userId from parent
    console.log("Requesting userId from parent...");
    window.parent.postMessage({ type: 'REQUEST_USERID' }, 'https://miniappre.vercel.app'); // Replace with your parent's origin
    console.log("REQUEST_USERID message sent to parent.");

    // Periodically update plinkoBalance in Firestore every 1 minute
    const interval = setInterval(async () => {
      if (!userId) {
        console.log("UserId is not set yet. Skipping balance update.");
        return;
      }
      const currentBal = Number(localStorage.getItem('plinkoBalance')) || 0;
console.log(`Updating Firestore for userId ${userId} with plinkoBalance: ${currentBal}`);
      const userRef = doc(db, 'telegramUsers', userId);
      try {
        await updateDoc(userRef, { plinkoBalance: currentBal });
        console.log('Updated plinkoBalance in Firestore:', currentBal);
        localStorage.setItem('plinkoBalance', currentBal.toString());
      } catch (err) {
        console.error('Error updating plinkoBalance:', err);
      }
    }, 60000); // Update every 1 minute

    return () => {
      console.log("Cleaning up event listeners and intervals...");
      window.removeEventListener('message', handleMessage);
      clearInterval(interval);
    };
  });
</script>

<style>
  /* Add any necessary styles here */
</style>

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