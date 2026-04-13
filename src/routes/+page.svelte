<script lang="ts">
  import { onMount, onDestroy } from 'svelte';

  type OrderLevel = [string, string]; 

  const symbol = 'CLUSDT'; 
  const wsUrl = 'wss://stream.bybit.com/v5/public/linear'; 
  const depth = 200; 
  
  let ws: WebSocket | null = null;

  let asks = $state<OrderLevel[]>([]);
  let bids = $state<OrderLevel[]>([]);
  let connectionStatus = $state('Connessione...');

  const mergeData = (currentBook: OrderLevel[], updates: string[][], isAsk: boolean): OrderLevel[] => {
    let bookMap = new Map<string, string>(currentBook.map(item => [item[0], item[1]]));
    
    updates.forEach(([price, qty]) => {
      if (parseFloat(qty) === 0) bookMap.delete(price);
      else bookMap.set(price, qty);
    });

    let mergedArray = Array.from(bookMap.entries()) as OrderLevel[];
    
    if (isAsk) {
      return mergedArray.sort((a, b) => parseFloat(a[0]) - parseFloat(b[0])).slice(0, depth);
    } else {
      return mergedArray.sort((a, b) => parseFloat(b[0]) - parseFloat(a[0])).slice(0, depth);
    }
  };

  onMount(() => {
    ws = new WebSocket(wsUrl);

    ws.onopen = () => {
      connectionStatus = `Live: ${symbol} (Perp)`;
      ws?.send(JSON.stringify({ 
        op: 'subscribe', 
        args: [`orderbook.${depth}.${symbol}`] 
      }));
    };

    ws.onmessage = (event) => {
      const response = JSON.parse(event.data);
      if (!response.topic || !response.topic.includes(symbol)) return;

      const { data, type } = response;

      if (type === 'snapshot') {
        asks = data.a || [];
        bids = data.b || [];
      } else if (type === 'delta') {
        if (data.a) asks = mergeData(asks, data.a, true);
        if (data.b) bids = mergeData(bids, data.b, false);
      }
    };

    ws.onclose = () => { connectionStatus = 'Riconnessione... 🔄'; };
  });

  onDestroy(() => ws?.close());
</script>


<div class="h-screen w-full bg-neutral-950 text-white flex justify-center overflow-hidden font-sans">
  
  <div class="w-full max-w-5xl flex flex-col overflow-hidden border-x border-neutral-800 bg-neutral-900/20 shadow-2xl">
    
    <div class="flex justify-between items-center px-6 py-3 bg-neutral-900 border-b border-neutral-800 shadow-md">
      <div class="flex items-center gap-3">
        <h1 class="text-xl font-black tracking-tighter text-emerald-400">{symbol}</h1>
        <span class="text-[10px] bg-neutral-800 px-2 py-0.5 rounded text-neutral-400 uppercase tracking-widest font-bold">Perpetual</span>
      </div>
      <div class="text-xs font-mono text-neutral-500">
        {connectionStatus} | Livelli: {depth}
      </div>
    </div>

    <div class="flex flex-col md:flex-row flex-1 overflow-hidden">
      
      <div class="flex-1 flex flex-col border-b md:border-b-0 md:border-r border-neutral-800">
        <div class="bg-neutral-900/50 px-4 py-2 text-[10px] font-bold text-neutral-500 uppercase flex justify-between z-20">
          <span>Quantità</span> <span>Prezzo (USDT)</span> </div>
        <div class="flex-1 overflow-y-auto scrollbar-thin scrollbar-thumb-neutral-800 relative">
          <div class="flex flex-col-reverse">
            {#each asks as [price, qty] (price)}
              <div class="relative flex justify-between px-4 py-1 border-b border-neutral-900/50 font-mono text-sm group">
                {#key qty}
                  <div class="absolute inset-0 animate-flash-ask pointer-events-none"></div>
                {/key}
                <span class="text-neutral-400 relative z-10">{parseFloat(qty).toLocaleString()}</span>
                <span class="text-red-500 relative z-10 group-hover:font-bold">{parseFloat(price).toFixed(5)}</span>
              </div>
            {/each}
          </div>
        </div>
      </div>

      <div class="flex-1 flex flex-col">
        <div class="bg-neutral-900/50 px-4 py-2 text-[10px] font-bold text-neutral-500 uppercase flex justify-between z-20">
          <span>Prezzo (USDT)</span>
          <span>Quantità</span>
        </div>
        <div class="flex-1 overflow-y-auto scrollbar-thin scrollbar-thumb-neutral-800 relative">
          <div class="flex flex-col">
            {#each bids as [price, qty] (price)}
              <div class="relative flex justify-between px-4 py-1 border-b border-neutral-900/50 font-mono text-sm group">
                {#key qty}
                  <div class="absolute inset-0 animate-flash-bid pointer-events-none"></div>
                {/key}
                <span class="text-emerald-500 relative z-10 group-hover:font-bold">{parseFloat(price).toFixed(5)}</span>
                <span class="text-neutral-400 relative z-10">{parseFloat(qty).toLocaleString()}</span>
              </div>
            {/each}
          </div>
        </div>
      </div>

    </div>
    
  </div>
</div>
