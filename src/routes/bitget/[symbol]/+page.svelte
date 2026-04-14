<script lang="ts">
	import { page } from '$app/state';

	const wsUrl = 'wss://ws.bitget.com/v2/ws/public';
	let symbol = $derived((page.params.symbol || 'BTCUSDT').toUpperCase());

	let connectionStatus = $state('Connessione...');

	$effect(() => {
		const ws = new WebSocket(wsUrl);

		ws.onopen = () => {
			connectionStatus = `Live: ${symbol} (Perp)`;
			ws.send(JSON.stringify({ op: 'subscribe', args: [`orderbook.${1}.${symbol}`] }));
		};

		ws.onmessage = (event) => {
			const response = JSON.parse(event.data);
			if (!response.topic || !response.topic.includes(symbol)) return;
		};

		ws.onclose = () => {
			connectionStatus = 'Riconnessione...';
		};

		return () => {
			ws.close();
		};
	});
</script>

<section>
	<h1>Bitget Order Book</h1>
</section>
