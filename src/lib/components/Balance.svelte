<script lang="ts">
  import { balance } from '$lib/stores/game';
  import { flyAndScale } from '$lib/utils/transitions';
  // import { Popover } from 'bits-ui'; // No longer needed

  // We'll still format the balance for display.
  let balanceFormatted = $derived(
    $balance.toLocaleString('en-US', {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
    })
  );
</script>

<div class="flex overflow-hidden rounded-md">
  <!-- Just the balance display. Removed the entire popover trigger/content. -->
  <div
    class="flex gap-2 bg-slate-900 px-3 py-2 text-sm font-semibold tabular-nums text-white sm:text-base"
  >
    <span class="select-none text-gray-500">$</span>
    <span class="min-w-16 text-right">
      {balanceFormatted}
    </span>
  </div>
</div>