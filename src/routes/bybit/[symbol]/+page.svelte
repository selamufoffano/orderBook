<script lang="ts">
  import { page } from '$app/state';

  type OrderLevel = [string, string]; 

  const wsUrl = 'wss://stream.bybit.com/v5/public/linear'; 
  const depth = 200; 
  
  let groupDecimals = $state(6); 

  let rawAsks = $state<OrderLevel[]>([]);
  let rawBids = $state<OrderLevel[]>([]);
  let connectionStatus = $state('Connessione...');

  let symbol = $derived((page.params.symbol || 'BTCUSDT').toUpperCase());

  const mergeData = (currentBook: OrderLevel[], updates: string[][], isAsk: boolean): OrderLevel[] => {
    let bookMap = new Map<string, string>(currentBook.map(item => [item[0], item[1]]));
    
    updates.forEach(([price, qty]) => {
      if (parseFloat(qty) === 0) bookMap.delete(price);
      else bookMap.set(price, qty);
    });

    let mergedArray = Array.from(bookMap.entries()) as OrderLevel[];
    if (isAsk) return mergedArray.sort((a, b) => parseFloat(a[0]) - parseFloat(b[0])).slice(0, depth);
    else return mergedArray.sort((a, b) => parseFloat(b[0]) - parseFloat(a[0])).slice(0, depth);
  };

  const aggregateData = (book: OrderLevel[], isAsk: boolean): OrderLevel[] => {
    const grouped = new Map<string, number>();

    for (const [price, qty] of book) {
      const groupPrice = parseFloat(price).toFixed(groupDecimals);
      const currentQty = grouped.get(groupPrice) || 0;
      grouped.set(groupPrice, currentQty + parseFloat(qty));
    }

    const result = Array.from(grouped.entries()).map(([p, q]) => [p, q.toString()] as OrderLevel);

    if (isAsk) return result.sort((a, b) => parseFloat(a[0]) - parseFloat(b[0]));
    else return result.sort((a, b) => parseFloat(b[0]) - parseFloat(a[0]));
  };

  let aggregatedAsks = $derived(aggregateData(rawAsks, true));
  let aggregatedBids = $derived(aggregateData(rawBids, false));

  $effect(() => {
    rawAsks = [];
    rawBids = [];
    const ws = new WebSocket(wsUrl);

    ws.onopen = () => {
      connectionStatus = `Live: ${symbol} (Perp)`;
      ws.send(JSON.stringify({ op: 'subscribe', args: [`orderbook.${depth}.${symbol}`] }));
    };

    ws.onmessage = (event) => {
      const response = JSON.parse(event.data);
      if (!response.topic || !response.topic.includes(symbol)) return;
      const { data, type } = response;
      if (type === 'snapshot') {
        rawAsks = data.a || [];
        rawBids = data.b || [];
      } else if (type === 'delta') {
        if (data.a) rawAsks = mergeData(rawAsks, data.a, true);
        if (data.b) rawBids = mergeData(rawBids, data.b, false);
      }
    };

    ws.onclose = () => { connectionStatus = 'Riconnessione...'; };
    return () => { ws.close(); };
  });
</script>

<div class="h-screen w-full bg-neutral-950 text-white flex justify-center overflow-hidden font-sans">
  
  <div class="w-full max-w-md md:max-w-5xl flex flex-col overflow-hidden border-x border-neutral-800 bg-neutral-900/20 shadow-2xl transition-all duration-300">
    
    <div class="shrink-0 flex justify-between items-center px-4 py-2 bg-neutral-900 border-b border-neutral-800 shadow-md">
      <div class="flex items-center gap-3">
        <h1 class="text-md font-black tracking-tighter text-emerald-400">{symbol}</h1>
        <span class="text-[10px] bg-neutral-800 px-2 py-0.5 rounded text-neutral-400 uppercase tracking-widest font-bold">Perp</span>
      </div>
      
      <div class="flex items-center gap-4 w-8 justify-evenly">
        <div class="flex items-center gap-5">
          <select 
            bind:value={groupDecimals}
            class="bg-neutral-800 text-xs font-mono border-none rounded px-5 py-1 outline-none focus:ring-1 focus:ring-emerald-500 cursor-pointer"
          >
            {#each Array(9) as _, i}
              <option value={i}>{i}</option>
            {/each}
          </select>
        </div>

        <div class="text-[10px] font-mono text-neutral-500 hidden sm:block uppercase tracking-wider">
          {connectionStatus}
        </div>
      </div>
    </div>

    <div class="flex-1 flex flex-col md:flex-row overflow-hidden">
      <div class="h-1/2 md:h-auto md:flex-1 flex flex-col border-b md:border-b-0 md:border-r border-neutral-800">
        <div class="bg-neutral-900/50 px-4 py-2 text-[10px] font-bold text-neutral-500 uppercase flex justify-between z-20 shadow-sm">
          <span>Prezzo (USDT)</span>
          <span>Quantità</span>
        </div>
        
        <div class="flex-1 overflow-y-auto scrollbar-thin scrollbar-thumb-neutral-800 relative flex flex-col-reverse">
          
          {#each aggregatedAsks as [price, qty] (price)}
            <div class="shrink-0 w-full relative flex justify-between px-4 py-1 border-b border-neutral-900/50 font-mono text-sm group">
              {#key qty}
                <div class="absolute inset-0 animate-flash-ask pointer-events-none"></div>
              {/key}
              <span class="text-red-500 relative z-10 group-hover:font-bold">{price}</span>
              <span class="text-neutral-400 relative z-10">{parseFloat(qty).toLocaleString('it-IT')}</span>
            </div>
          {/each}
          
        </div>
      </div>

      <div class="h-1/2 md:h-auto md:flex-1 flex flex-col">
        <div class="md:hidden shrink-0 bg-neutral-800 text-center py-1 text-[10px] font-bold text-neutral-400 uppercase tracking-widest border-b border-neutral-900 z-20">
          Spread
        </div>
        <div class="hidden md:flex bg-neutral-900/50 px-4 py-2 text-[10px] font-bold text-neutral-500 uppercase justify-between z-20 shadow-sm">
          <span>Prezzo (USDT)</span>
          <span>Quantità</span>
        </div>
        <div class="flex-1 overflow-y-auto scrollbar-thin scrollbar-thumb-neutral-800 relative">
          <div class="flex flex-col">
            {#each aggregatedBids as [price, qty] (price)}
              <div class="relative flex justify-between px-4 py-1 border-b border-neutral-900/50 font-mono text-sm group">
                {#key qty}
                  <div class="absolute inset-0 animate-flash-bid pointer-events-none"></div>
                {/key}
                <span class="text-emerald-500 relative z-10 group-hover:font-bold">{price}</span>
                <span class="text-neutral-400 relative z-10">{parseFloat(qty).toLocaleString('it-IT')}</span>
              </div>
            {/each}
          </div>
        </div>
      </div>
    </div>
  </div>
</div>