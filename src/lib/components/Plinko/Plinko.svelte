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
  import { db } from '$lib/firebase';

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

  // Helper function that rounds a number to two decimals
  function roundToTwo(num: number): number {
    return Math.round(num * 100) / 100;
  }

  // Helper function that returns a string exactly two decimals
  function formatBalance(num: number): string {
    return roundToTwo(num).toFixed(2);
  }

  // Function to fetch the plinkoBalance from Firestore (fallback)
  async function fetchPlinkoBalance(userId: string) {
    console.log(`Fetching plinko balance for userId: ${userId}`);
    const userRef = doc(db, 'telegramUsers', userId);
    try {
      const userSnap = await getDoc(userRef);
      if (userSnap.exists()) {
        const userData = userSnap.data();
        // Round and format balance
        const plinkoBalNum = roundToTwo(Number(userData.plinkoBalance || 0));
        console.log(`Fetched plinkoBalance for userId ${userId}: ${plinkoBalNum}`);
        sessionBalance.set(plinkoBalNum);
        oldBalance.set(plinkoBalNum);
        localStorage.setItem('plinkoBalance', formatBalance(plinkoBalNum));
      } else {
        console.error('User not found in Firestore for ID', userId);
      }
    } catch (err) {
      console.error('Error fetching plinko balance:', err);
    }
  }

  // Optionally sync Firestore with local storage
  async function syncFirestoreWithLocalStorage() {
    if (!userId) {
      console.error("User ID not available. Cannot sync balance.");
      return;
    }
    // Convert stored string to a number, then round
    const stored = localStorage.getItem('plinkoBalance');
    const storedBalance = stored ? roundToTwo(Number(stored)) : 0;
    console.log(`Syncing Firestore with localStorage plinkoBalance: ${storedBalance}`);
    try {
      const userRef = doc(db, 'telegramUsers', userId);
      await updateDoc(userRef, { plinkoBalance: storedBalance });
      console.log("Firestore updated with localStorage plinkoBalance:", storedBalance);
    } catch (err) {
      console.error("Error syncing Firestore with localStorage plinkoBalance:", err);
    }
  }

  onMount(() => {
    console.log("Component mounted. Setting up message listener...");

    function handleMessage(event: MessageEvent) {
      console.log('Received message:', event.data, 'from', event.origin);
      const { type, userId: incomingUserId, requestId, amount } = event.data || {};

      if (type === 'USERID' && incomingUserId) {
        console.log('USERID received with:', incomingUserId);
        userId = incomingUserId;

        // Check localStorage first:
        const storedBalanceStr = localStorage.getItem('plinkoBalance');
        if (storedBalanceStr !== null) {
          // Use parseFloat to convert the already formatted string.
          const storedBalance = roundToTwo(parseFloat(storedBalanceStr));
          console.log('Using localStorage plinkoBalance:', storedBalance);
          sessionBalance.set(storedBalance);
          oldBalance.set(storedBalance);
          // Update Firestore with the rounded balance
          (async () => {
            try {
              if (!userId) {
                console.error("User ID is not set.");
                return;
              }
              const userRef = doc(db, 'telegramUsers', userId);
              await updateDoc(userRef, { plinkoBalance: storedBalance });
              console.log('Updated Firestore with localStorage balance:', storedBalance);
            } catch (err) {
              console.error("Error updating Firestore from localStorage balance:", err);
            }
          })();
        } else {
          // No local storage available, fallback to Firestore:
          fetchPlinkoBalance(incomingUserId);
        }
      }

      if (type === 'TRANSFER_BALANCE_REQUEST' && requestId) {
        console.log('Received TRANSFER_BALANCE_REQUEST with requestId:', requestId);
        if (userId) {
          const storedBalanceStr = localStorage.getItem('plinkoBalance');
          const storedBalance = storedBalanceStr 
            ? roundToTwo(parseFloat(storedBalanceStr))
            : 0;
          console.log(`Sending plinkoBalance to parent from localStorage: ${storedBalance}`);
          window.parent.postMessage(
            { type: 'TRANSFER_BALANCE_RESPONSE', requestId, plinkoBalance: storedBalance },
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

      if (type === 'ADD_BALANCE' && typeof amount === 'number') {
        console.log(`Adding ${amount} to local Plinko balance.`);
        // Perform arithmetic in cents if needed:
        const currentBal = get(sessionBalance);
        const newBal = roundToTwo(currentBal + amount);
        sessionBalance.set(newBal);
        oldBalance.set(newBal);
        localStorage.setItem('plinkoBalance', formatBalance(newBal));
      }

      if (type === 'DEDUCT_BALANCE' && typeof amount === 'number') {
        console.log(`Deducting ${amount} from local Plinko balance.`);
        const currentBal = get(sessionBalance);
        const newBal = roundToTwo(currentBal - amount);
        sessionBalance.set(newBal);
        oldBalance.set(newBal);
        localStorage.setItem('plinkoBalance', formatBalance(newBal));
      }
    }

    window.addEventListener('message', handleMessage);

    // Request userId from parent
    console.log("Requesting userId from parent...");
    window.parent.postMessage({ type: 'REQUEST_USERID' }, 'https://miniappre.vercel.app');
    console.log("REQUEST_USERID message sent to parent.");

    // (Optional) Uncomment if you want to sync later.
    // syncFirestoreWithLocalStorage();

    return () => {
      console.log("Cleaning up event listeners...");
      window.removeEventListener('message', handleMessage);
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
