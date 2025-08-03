<script lang="ts">
  import grassBg from '$lib/assets/grassbg.webp';
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
  
  // Local store for the user's Plinko balance (in dollars)
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
  
  // Helper functions to work in cents
  function toCents(num: number): number {
    return Math.round(num * 100);
  }
  
  function fromCents(cents: number): number {
    return cents / 100;
  }
  
  // Formats the number (in dollars) as a string with two decimals.
  function formatBalance(num: number): string {
    return fromCents(toCents(num)).toFixed(2);
  }
  
  // Function to fetch the plinkoBalance from Firestore (fallback)
  async function fetchPlinkoBalance(userId: string) {
    console.log(`Fetching plinko balance for userId: ${userId}`);
    const userRef = doc(db, 'telegramUsers', userId);
    try {
      const userSnap = await getDoc(userRef);
      if (userSnap.exists()) {
        const userData = userSnap.data();
        // Convert the fetched value to a number and treat it as dollars
        const plinkoBal = fromCents(toCents(Number(userData.plinkoBalance || 0)));
        console.log(`Fetched plinkoBalance for userId ${userId}: ${plinkoBal}`);
        sessionBalance.set(plinkoBal);
        oldBalance.set(plinkoBal);
        localStorage.setItem('plinkoBalance', formatBalance(plinkoBal));
      } else {
        console.error('User not found in Firestore for ID', userId);
      }
    } catch (err) {
      console.error('Error fetching plinko balance:', err);
    }
  }
  
  async function syncFirestoreWithLocalStorage() {
    if (!userId) {
      console.error("User ID not available. Cannot sync balance.");
      return;
    }
    const stored = localStorage.getItem('plinkoBalance');
    const storedBalance = stored ? fromCents(toCents(Number(stored))) : 0;
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
          const storedBalance = fromCents(toCents(Number(storedBalanceStr)));
          console.log('Using localStorage plinkoBalance:', storedBalance);
          sessionBalance.set(storedBalance);
          oldBalance.set(storedBalance);
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
          fetchPlinkoBalance(incomingUserId);
        }
      }
  
      if (type === 'TRANSFER_BALANCE_REQUEST' && requestId) {
        console.log('Received TRANSFER_BALANCE_REQUEST with requestId:', requestId);
        if (userId) {
          const stored = localStorage.getItem('plinkoBalance');
          const storedBalance = stored ? fromCents(toCents(Number(stored))) : 0;
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
        const currentCents = toCents(get(sessionBalance));
        const amountCents = toCents(amount);
        const newBal = fromCents(currentCents + amountCents);
        sessionBalance.set(newBal);
        oldBalance.set(newBal);
        localStorage.setItem('plinkoBalance', formatBalance(newBal));
      }
  
      if (type === 'DEDUCT_BALANCE' && typeof amount === 'number') {
        console.log(`Deducting ${amount} from local Plinko balance.`);
        const storedBalanceStr = localStorage.getItem('plinkoBalance');
        const currentBalance = storedBalanceStr ? parseFloat(storedBalanceStr) : get(sessionBalance);
        const currentCents = toCents(currentBalance);
        const amountCents = toCents(amount);
        const newCents = currentCents - amountCents;
        const newBalance = fromCents(newCents);
        sessionBalance.set(newBalance);
        oldBalance.set(newBalance);
        localStorage.setItem('plinkoBalance', formatBalance(newBalance));
      }
    }
  
    window.addEventListener('message', handleMessage);
    window.parent.postMessage({ type: 'REQUEST_USERID' }, 'https://miniappre.vercel.app');
    return () => window.removeEventListener('message', handleMessage);
  });
</script>
  
<style>
  .grass-bg {
    background: url("{grassBg}") center/cover no-repeat;
  }
</style>
  
<div class="relative grass-bg">
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
