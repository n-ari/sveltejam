<script>
  import { readable } from "svelte/store";

  // params
  const FPS = 30;
  const WIDTH = 600;
  const HEIGHT = 400;

  let gameEnd = false;
  let stage = 0;

  // time-limited clock
  const clock = readable(0, set => {
    // 30fps
    let t = 0;
    const interval = setInterval(() => {
      if (gameEnd) {
        clearInterval(interval);
        return;
      }
      t += 1;
      set(t);
    }, 1000 / FPS);
    return () => clearInterval(interval);
  });
  // time
  $: timeString = ($clock / FPS).toFixed(2);

  const BLACK = 0,
    WHITE = 1;

  // player
  let player = { x: 20, y: HEIGHT - 40, r: 10, color: BLACK };
  let vx = +1,
    vy = 0,
    ay = 0.1;

  // goal
  let goal = { x: 40, y: 40, r: 15 };
  clock.subscribe(t => {
    if (collisionBlock(player, goal)) {
      gameEnd = true;
    }
  });

  // functions
  function collisionBlock(block1, block2, EPS = 1e-9) {
    const { x: x1, y: y1, r: r1 } = block1;
    const { x: x2, y: y2, r: r2 } = block2;
    return (
      x1 - r1 < x2 + r2 + EPS &&
      x2 - r2 < x1 + r1 + EPS &&
      y1 - r1 < y2 + r2 + EPS &&
      y2 - r2 < y1 + r1 + EPS
    );
  }

  // blocks
  let blocks = [];
  function bhelper(x, y, r, c) {
    blocks = [...blocks, { x, y, r, color: c }];
  }
  if (stage === 0) {
    let color = BLACK;
    for (let x = 5; x < WIDTH; x += 10) {
      bhelper(x, HEIGHT - 15, 5, color);
      color = 1 - color;
    }
  }
  function blockSpawn() {
    const THRESHOLD = 4;
    let low = 0,
      high = 1e9;
    for (let _ = 0; _ < 40; _++) {
      let size = (low + high) / 2.0;
      let newBlock = {
        x: player.x,
        y: player.y + player.r + vy + size / 2,
        r: size / 2
      };
      let ok = true;
      for (let block of blocks) {
        if (collisionBlock(newBlock, block, -1e-9)) {
          ok = false;
          break;
        }
      }
      if (ok) {
        low = size;
      } else {
        high = size;
      }
    }
    if (low < THRESHOLD) return;
    if (low > 1e5) return;
    let size = low;
    let newBlock = {
      x: player.x,
      y: player.y + player.r + vy + size / 2,
      r: size / 2,
      color: 1 - player.color
    };
    let collided = blocks.filter(block => collisionBlock(newBlock, block, 1));
    let bottomCollided = collided.filter(
      block => newBlock.y + newBlock.r < block.y - block.r + 3
    );
    if (bottomCollided.length > 0) {
      blocks = [...blocks, newBlock];
    }
  }

  // button
  let button = false;
  function handleKeydown(event) {
    if (event.keyCode === 32) {
      button = true;
    }
  }

  // jump and spawn block
  clock.subscribe(t => {
    if (!button) return;
    button = false;
    // get bottom block
    let collided = blocks.filter(block => collisionBlock(player, block));
    let bottomCollided = collided.filter(
      block => player.y + player.r < block.y - block.r + 3
    );
    if (bottomCollided.length > 0) {
      // jump
      ay = 0.1;
      vy = -3;
      player = { ...player, y: player.y + vy, color: 1 - player.color };
    } else {
      // spawn block
      blockSpawn();
    }
  });

  clock.subscribe(t => {
    for (let c = 0; c < 4; c++) {
      // get bottom block
      let oppositeBlocks = blocks.filter(block => block.color !== player.color);
      let collided = oppositeBlocks.filter(block =>
        collisionBlock(player, block)
      );
      let bottomCollided = collided.filter(
        block => player.y + player.r < block.y - block.r + 3
      );
      let notBottomCollided = collided.filter(
        block => player.y + player.r >= block.y - block.r + 3
      );
      let rightCollided = notBottomCollided.filter(
        block => player.x + player.r < block.x - block.r + 3
      );
      let leftCollided = notBottomCollided.filter(
        block => player.x - player.r > block.x + block.r - 3
      );
      let topCollided = notBottomCollided.filter(
        block => player.y - player.r > block.y + block.r - 3
      );
      let falling = bottomCollided.length === 0;
      if (falling) {
        vy += ay;
        vy = Math.min(vy, 2);
      } else if (topCollided.length > 0) {
        vy = 0;
        player = {
          ...player,
          y: Math.max(
            player.y,
            topCollided.reduce((a, b) => Math.max(a, b.y + b.r), 1e9) + player.r
          )
        };
      } else {
        vy = 0;
        player = {
          ...player,
          y: Math.min(
            player.y,
            bottomCollided.reduce((a, b) => Math.min(a, b.y - b.r), 1e9) -
              player.r
          )
        };
      }
      if (vx > 0) {
        if (player.x + player.r >= WIDTH || rightCollided.length > 0) {
          vx = -vx;
        }
      } else {
        if (player.x - player.r <= 0 || leftCollided.length > 0) {
          vx = -vx;
        }
      }
      player = { ...player, x: player.x + vx, y: player.y + vy };
    }
  });
</script>

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
  h2 {
    color: #ff3e00;
    font-size: 2em;
    font-weight: bold;
  }

  svg {
    background-color: #98d9fa;
  }

  rect.black {
    fill: black;
  }

  rect.white {
    fill: white;
  }
</style>

<svelte:head>
	<title>Ikaruga Block Filling / traP3jam 2020-03-27</title>
</svelte:head>
<svelte:window on:keydown={handleKeydown} />

<main>
  <h1>Ikaruga Block Filling</h1>
  <h2>traP3jam 2020-03-27</h2>
  <svg width="{WIDTH}px" height="{HEIGHT}px">
    <!-- score -->
    <!-- <text x="10" y="20">score: {score} points</text> -->

    <!-- timer -->
    <text x={WIDTH - 120} y="20">time: {timeString} sec</text>

    <!-- player -->
    <rect
      x={player.x - player.r}
      y={player.y - player.r}
      width={player.r * 2}
      height={player.r * 2}
      class="player"
      class:black={player.color === BLACK}
      class:white={player.color === WHITE} />

    <!-- blocks -->
    {#each blocks as block}
      <rect
        x={block.x - block.r}
        y={block.y - block.r}
        width={block.r * 2}
        height={block.r * 2}
        class="block"
        class:black={block.color === BLACK}
        class:white={block.color === WHITE} />
    {/each}

    <!-- goal -->
    <polygon
      fill="yellow"
      stroke="yellow"
      stroke-width="10"
      points="40,30 37,37 30,40 37,43 40,50 43,43 50,40 43,37" />
  </svg>
</main>
