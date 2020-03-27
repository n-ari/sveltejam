<script>
	import { readable } from 'svelte/store';
	import { spring } from 'svelte/motion';

	// params
	const FPS = 60;
	const WIDTH = 600;
	const HEIGHT = 400;
	const TIME_LIMIT = 15.0;
	const ITEMS_LIMIT = 30;

	// time-limited clock
	const clock = readable(0, set => {
		// 30fps
		let t = 0;
		const interval = setInterval(() => {
			t += 1;
			set(t);
			if(t/FPS >= TIME_LIMIT){
				clearInterval(interval);
			}
		}, 1000/FPS);
		return () => clearInterval(interval);
	});
	// rest time
	$: restTimeString = (TIME_LIMIT - $clock/FPS).toFixed(2);

	// mouse
	let mousepos = spring({ x:WIDTH/2, y:HEIGHT/2 }, {
		stiffness: 0.1,
		damping: 0.3,
	});
	function mousemove(e) {
		// const rect = e.target.getBoundingClientRect();
		const rect = document.getElementsByTagName("svg")[0].getBoundingClientRect();
		mousepos.set({
			x: e.clientX - rect.left,
			y: e.clientY - rect.top,
		});
	}

	// score
	let score = 0;

	// player
	let player = {x: 0, y: 0, r: 10};
	mousepos.subscribe(pos => {
		player.x = pos.x;
		player.y = pos.y;
	});

	// items
	let items = [];
	function itemSpawn() {
		const x = Math.random() * WIDTH;
		const y = Math.random() * HEIGHT;
		const r = 3;
		items = [...items, {x, y, r}];
	}
	clock.subscribe(t => {
		if(t % Math.floor(FPS*0.2) === 0 && items.length < 30){
			// per 3 seconds
			itemSpawn();
		}
	});

	// collision
	clock.subscribe(() => {
		const beforeCount = items.length;
		const filtered = items.filter(item => {
			const dx = item.x - player.x;
			const dy = item.y - player.y;
			const dr = item.r + player.r;
			return dx*dx + dy*dy > dr*dr;
		});
		items = filtered;
		const afterCount = items.length;
		score += beforeCount - afterCount;
	});
</script>

<main>
	<h1>Svelte jam</h1>
	<svg
		width="{WIDTH}px"
		height="{HEIGHT}px"
		on:mousemove="{mousemove}"
	>
		<!-- score -->
		<text x="10" y="20">score: {score} points</text>

		<!-- timer -->
		<text x="{WIDTH-120}" y="20">time: {restTimeString} sec</text>

		<!-- player -->
		<circle
			cx="{player.x}"
			cy="{player.y}"
			r="{player.r}"
			class="player"
		/>

		<!-- items -->
		{#each items as item}
			<circle
				cx="{item.x}"
				cy="{item.y}"
				r="3"
				class="item"
			/>
		{/each}
	</svg>
</main>

<style>
	main {
		text-align: center;
		padding: 1em;
		margin: 0 auto;
	}

	h1 {
		color: #ff3e00;
		font-size: 4em;
		font-weight: 100;
	}

	svg {
		background-color: #efefef;
	}

	circle.player {
		fill: red;
	}
	
	circle.item {
		fill: blue;
	}
</style>
