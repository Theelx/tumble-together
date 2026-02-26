<script>
  import MarbleTray from './MarbleTray.svelte';
  import NumbersModal from './NumbersModal.svelte';
  import { tick, createEventDispatcher } from 'svelte';
  import { holding, currentChallenge, customMix } from '../store.js';
  import Board, { board } from '../board.js';
  import { marbles } from '../marbles.js';
  import { socket } from '../socket.js';
  import { basePath } from '../store.js';

  export let lastGrab;
  export let boardElement;

  const dispatch = createEventDispatcher();

  $: if ($socket) $socket.on('run', (side) => {
      triggerLever(side);
      console.log('run');
    });

  function grab(e, x, y) {
    if (((e.type === 'mousedown' && e.button === 0) ||
         (e.type === 'touchstart' && e.touches.length === 1)) &&
        !$holding && !$board.marble && $board[y][x]) {
      e.preventDefault();
      e.stopPropagation();
      if (e.type === 'touchstart') dispatch('touch', e);
      if (!$board[y][x].locked) {
        console.log(`Grabbed ${$board[y][x].name}`);
        $holding = $board[y][x];
        $board[y][x] = false;
        lastGrab = {x, y, timeout: setTimeout(() => lastGrab = {x: false, y: false, timeout: false}, 300)};
        socket.sendBoard();
      }else if ($board[y][x].flipsOnMarble) {
        $board.flip(x, y);
        $board = $board;
        console.log(`Flipped ${$board[y][x].name}`);
        socket.sendBoard();
      }
    }
  }

  let marbleElement;
  let reset = false;
  let triggerLock = false;
  let alternateMode = false;
  let alternateNextSide = 'left';
  let autoTriggerActive = false;
  let showSolution = false;
  let ghostBoard = null;
  let lastChallengeId = null;
  let hintDisabled = false;
  let hintedPositions = new Set();
  export let escapedMarbles = [];
  const AUTO_TRIGGER_DELAY_MS = 500;

  $: challengeRestricts = Boolean(
    $currentChallenge &&
    ($currentChallenge.trigger === 'left' || $currentChallenge.trigger === 'right') &&
    $currentChallenge.restrictTrigger !== false
  );

  $: if ($currentChallenge?.id !== lastChallengeId) {
    showSolution = false;
    hintedPositions = new Set();
    lastChallengeId = $currentChallenge?.id ?? null;
  }

  $: if ($currentChallenge?.solutionCode) {
    try {
      ghostBoard = Board.create($currentChallenge.solutionCode);
    } catch (err) {
      console.log(err);
      ghostBoard = null;
    }
  } else {
    ghostBoard = null;
  }

  function toggleSolution() {
    showSolution = !showSolution;
  }

  function ghostPartAt(x, y) {
    if (!ghostBoard) return false;
    return ghostBoard[y]?.[x] || false;
  }

  function isSamePart(a, b) {
    if (!a && !b) return true;
    if (!a || !b) return false;
    return a.constructor === b.constructor &&
      a.facing === b.facing &&
      !!a.locked === !!b.locked;
  }

  function clonePart(part) {
    if (!part) return false;
    return new part.constructor(part.facing, part.locked);
  }

  $: hintDisabled = ghostBoard ? boardsMatchSolution() : true;

  function boardsMatchSolution() {
    if (!ghostBoard) return false;
    for (let y = 0; y < $board.length; y++) {
      for (let x = 0; x < $board[y].length; x++) {
        if (!$board.isValid(x, y)) continue;
        if (!isSamePart($board[y][x], ghostBoard[y][x])) return false;
      }
    }
    return true;
  }

  function revealNextHint() {
    if (!ghostBoard || hintDisabled) return;
    for (let y = 0; y < $board.length; y++) {
      for (let x = 0; x < $board[y].length; x++) {
        if (!$board.isValid(x, y)) continue;
        const solutionPart = ghostBoard[y][x];
        const key = `${x},${y}`;
        if (solutionPart && !isSamePart($board[y][x], solutionPart) && !hintedPositions.has(key)) {
          hintedPositions = new Set([...hintedPositions, key]);
          return;
        }
      }
    }
  }

  function resolveAllowedSide(desiredSide) {
    if (challengeRestricts) {
      return $currentChallenge.trigger;
    }
    return desiredSide;
  }

  function pickAutoSide(preferredSide) {
    if (challengeRestricts) {
      return $marbles[$currentChallenge.trigger]?.length ? $currentChallenge.trigger : null;
    }
    return $marbles[preferredSide]?.length ? preferredSide : null;
  }

  function delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  function stopAutoTrigger() {
    alternateMode = false;
    autoTriggerActive = false;
  }

  function startAutoTrigger(startSide) {
    alternateMode = true;
    alternateNextSide = startSide;
    if (autoTriggerActive) return;
    autoTriggerActive = true;
    runAutoTrigger();
  }

  async function runAutoTrigger() {
    while (alternateMode) {
      const allowedSide = pickAutoSide(alternateNextSide);
      if (!allowedSide) {
        stopAutoTrigger();
        break;
      }
      if ($board.marble) {
        await delay(AUTO_TRIGGER_DELAY_MS);
        continue;
      }
      const result = await triggerLever(allowedSide);
      if (!alternateMode) break;
      if (result && result.side) {
        alternateNextSide = result.side;
      } else {
        alternateNextSide = allowedSide;
      }
      await delay(AUTO_TRIGGER_DELAY_MS);
    }
  }

  function triggerLever(side='left') {
    if (!$board.marble && $marbles[side].length) {
      const marble = $marbles[side].shift();
      $marbles = $marbles;
      return $board.startRun(marble, side, async () => {
        // Starts a marble run, waits for the marble to animate through current part before advancing it through the board.
        $board = $board;
        reset = true; // Applies reset class to clear transition based animations.
        await tick();
        setTimeout(() => reset = false, 1);

        return new Promise(resolve => {
          marbleElement.addEventListener('animationend', resolve, {once: true});
          marbleElement.addEventListener('oTransitionEnd', resolve, {once: true});
          marbleElement.addEventListener('webkitTransitionEnd', resolve, {once: true});
        });
      }).then(result => {
        if (result) {
          // Adds the marble to the results and atempts to launch another one from the coresponding side.
          $marbles.results.push(result.marble);
          $board = $board;
          if (!alternateMode) {
            triggerLever(result.side);
          }
          $marbles = $marbles;
        }
        return result;
      }).catch(() => {
        if (marble) {
          escapedMarbles = [...escapedMarbles, marble];
        }
        // If anything goes wrong, reset the board.
        // Need to add better feedback for dropped marbles.
        if ($marbles.results.length === 0) {
          marbles.reset();
          triggerLock = false;
        } else {
          $board.marble = false;
          triggerLock = true;
        }
        return false;
      });
    }
    return Promise.resolve(false);
  }

  function hasMarble(x, y) {
    return ($board.marble && $board.position &&
      ((x === $board.position.x && y === $board.position.y)));
  }

  function isMoving(x, y) {
    // Determines if the part at x,y should be animating.
    return ((hasMarble(x, y) && ($board.at(x, y).name != 'gear')) ||
      ($board.position && !hasMarble(x, y) &&
        $board.flipableNeighbors($board.position.x, $board.position.y).has($board.at(x, y))))
  }

  function marbleDirection(x, y) {
    return hasMarble(x, y) &&
      (Math.floor(($board.direction +1) / 2) === $board.at(x, y).facing);
  }

  function triggerLeft() {
    stopAutoTrigger();
    if ($socket) $socket.emit('run', 'left');
    triggerLever('left');
  }

  function triggerRight() {
    stopAutoTrigger();
    if ($socket) $socket.emit('run', 'right');
    triggerLever('right');
  }

  function triggerAlternating(startSide) {
    stopAutoTrigger();
    startAutoTrigger(startSide);
  }

  export function resetMarbles() {
    marbles.reset($marbles.numbers.left, $marbles.numbers.right);
    triggerLock = false;
    stopAutoTrigger();
    escapedMarbles = [];
    socket.sendBoard();
  }

  function editMarbleNumbers(side) {
    $marbles.edit[side] = !$currentChallenge && !$customMix;
    $marbles = $marbles;
  }

  function updateMarbleNumbers(side, number) {
    const oldNumber = $marbles.numbers[side];
    $marbles.numbers[side] = number;
    $marbles = $marbles;
    if (oldNumber != $marbles.numbers[side]) resetMarbles();
  }

</script>
<div id="board-container">
  <div id="top-trays" class:no-challenge={!$currentChallenge}>
    <div class="container">
      <div class="marble-numbers" on:click="{() => editMarbleNumbers('left')}">
        <img src="{$basePath}images/marbleblue.svg" alt="{$marbles.numbers.left} Blue Marbles">
        x {$marbles.numbers.left}
      </div>
      <MarbleTray marbles={$marbles.left}/>
    </div>
    <div class="container">
      <div class="marble-numbers" on:click="{() => editMarbleNumbers('right')}">
        <img src="{$basePath}images/marblered.svg" alt="{$marbles.numbers.right} Red Marbles">
        x {$marbles.numbers.right}
      </div>
      <MarbleTray direction="right" marbles={$marbles.right}/>
    </div>
  </div>
  <div id="start-ramps">
    <div class="marble-start">
      {#if ($board.marble && $board.position && (($board.direction === 1 && $board.position.y === -1)))}
        <img bind:this={marbleElement} class="marble" src="{$basePath}images/marble{$board.marble.color}.svg" alt="">
      {/if}
    </div>
    <div class="marble-start flipped">
      {#if ($board.marble && $board.position && (($board.direction === -1 && $board.position.y === -1)))}
        <img bind:this={marbleElement} class="marble" src="{$basePath}images/marble{$board.marble.color}.svg" alt="">
      {/if}
    </div>
  </div>
  <div id="board" bind:this={boardElement}>
    {#each $board as row, y (y)}
    <div class="row">
      {#each row as part, x (x)}
        {#if $board.isValid(x, y)}
        <div class:occupied="{part && !part.locked}" class:empty={!part}
            class:slot="{$board.hasSlot(x, y)}" class="position"
            on:mousedown="{e => grab(e, x, y)}"
            on:touchstart="{e => grab(e, x, y)}">
          {#if ghostBoard}
            {#if !part && ghostPartAt(x, y) && (showSolution || hintedPositions.has(`${x},${y}`))}
              <div class="ghost-overlay" aria-hidden="true">
                <div class:flipped={ghostPartAt(x, y).facing} class={ghostPartAt(x, y).name}>
                  <img class="part ghost-part" src="{$basePath}images/{ghostPartAt(x, y).name}.svg" alt="">
                </div>
              </div>
            {/if}
          {/if}
          {#if part}
          <div class:flipped={part.facing} class={part.name} class:right="{marbleDirection(x, y)}" >
            <div class="wrapper" class:down="{isMoving(x, y)}"
                class:reset>
              <img class="part" class:locked={part.locked} src="{$basePath}images/{part.name}.svg" alt={part.name}>
              {#if hasMarble(x, y)}
                <img bind:this={marbleElement} class="marble" src="{$basePath}images/marble{$board.marble.color}.svg" alt="">
              {/if}
            </div>
          </div>
          {:else if hasMarble(x, y)}
            <img bind:this={marbleElement} class="marble" class:right="{!Math.floor(($board.direction +1) / 2)}"
              src="{$basePath}images/marble{$board.marble.color}.svg" alt="">
          {/if}
        </div>
        {:else}
          <div class="position blank"></div>
        {/if}
      {/each}
    </div>
    {/each}
  </div>
  <div id="finish-line" aria-hidden="true"></div>
  <div id="levers">
    <div class="lever-column">
      <button class="lever-blue" on:click={triggerLeft}
        disabled="{$board.marble || triggerLock || (challengeRestricts && $currentChallenge.trigger === 'right')}">Trigger Left</button>
      <button class="lever-blue" on:click={() => triggerAlternating('left')}
        disabled="{$board.marble || triggerLock || (challengeRestricts && $currentChallenge.trigger === 'right')}">Trigger Alternating - Blue</button>
    </div>
    <div class="lever-column">
      <button class="lever-red" on:click={triggerRight}
        disabled="{$board.marble || triggerLock || (challengeRestricts && $currentChallenge.trigger === 'left')}">Trigger Right</button>
      <button class="lever-red" on:click={() => triggerAlternating('right')}
        disabled="{$board.marble || triggerLock || (challengeRestricts && $currentChallenge.trigger === 'left')}">Trigger Alternating - Red</button>
    </div>
  </div>
  <div id="board-actions">
    {#if $currentChallenge?.solutionCode}
      <button class="solution-button" on:click={toggleSolution}>
        {showSolution ? 'Hide Solution' : 'Show Solution'}
      </button>
      <button class="hint-button" on:click={revealNextHint} disabled={hintDisabled}>Hint</button>
    {/if}
    <button class="reset-button" on:click={resetMarbles}>Reset</button>
  </div>
</div>

<NumbersModal bind:visible={$marbles.edit.left} title="Number of Blue Marbles"
    number={$marbles.numbers.left} infinity={false} max=20
    on:update="{(e) => updateMarbleNumbers('left', e.detail)}">
  <img class="modal-image" src="{$basePath}images/marbleblue.svg" alt="Blue Marble">
</NumbersModal>

<NumbersModal bind:visible={$marbles.edit.right} title="Number of Red Marbles"
    number={$marbles.numbers.right} infinity={false} max=20
    on:update="{(e) => updateMarbleNumbers('right', e.detail)}">
  <img class="modal-image" src="{$basePath}images/marblered.svg" alt="Red Marble">
</NumbersModal>


<style>
  #board-container {
    border: 3px solid #c8c8c8;
    border-radius: 10px;
  }
  #top-trays {
    display: flex;
    justify-content: space-around;
  }
  #top-trays .container {
    width: 100%;
  }
  .marble-numbers {
    text-align: center;
    padding-bottom: 0.5em;
  }
  .no-challenge .marble-numbers {
    cursor: pointer;
  }
  .marble-numbers img {
    height: 1.5em;
    width: 1.5em;
    margin-bottom: -0.4em;
  }

  .row {
    width: 66vh;
    height: 6vh;
    overflow: visible;
  }
  .position {
    background-image: url(./images/bg-peg.svg);
    background-image: url(../images/bg-peg.svg);
    background-size: cover;
    display: inline-block;
    width: 6vh;
    height: 6vh;
    overflow: visible;
    position: relative;
  }
  @media (max-aspect-ratio: 7/9) {
    .row {
      width: 88vw;
      height: 8vw;
    }
    .position {
      width: 8vw;
      height: 8vw;
    }

  }

  .position.slot {
    background-image: url(./images/bg-slot.svg);
    background-image: url(../images/bg-slot.svg);
  }
  .position.blank {
    background-image: none;
    z-index: 0;
  }
  .position.occupied {
    cursor: grab;
  }

  .occupied div {
    pointer-events: none;
  }

  .wrapper {
    position: relative;
  }
  .part {
    width: 120%;
    height: 120%;
    margin: -10%;
  }
  .slot .gear .part {
    transform: rotate(22.5deg);
  }
  .slot .gear.flipped .part {
    transform: rotate(-22.5deg);
  }
  .flipped {
    transform: scaleX(-1);
  }
  .locked {
    filter: grayscale(75%);
  }
  .interceptor .locked {
    filter: brightness(150%);
  }
  
  .marble {
    position: absolute;
    top: 10%;
    left: 10%;
    transform: translate(-50%, -50%);
    animation-duration: 0.3s;
    width: 1.44vh;
    height: 1.44vh;
    z-index: 1;
  }

  .bit .part, .gearbit .part {
    position: relative;
    z-index: 2;
  }

  @media (max-aspect-ratio: 7/9) {
    .marble {
      width: 1.92vw;
      height: 1.92vw;
    }
  }

  /*Component Animations*/

  .ramp .wrapper {
    transition: transform 0.2s ease-in;
  }
  .down {
    transition: transform 0.2s ease-in;
    transform: rotate(-90deg);
    transition-delay: 0.05s;
  }

  .down.reset {
    transition-property: none;
    transform: none;
  }

  .crossover .down, .interceptor .down {
    transform: none;
  }
  .ramp.right .down, .bit.right .down {
    transition-delay: 0.1s;
  }

  @keyframes ramp {
    0% {top: -10%; left: 0;}
    30% {top: 20%; left: 20%;}
    90% {top: 20%; left: 20%;}
    100% {top: 0%; left: 5%;}
  }

  @keyframes rampright {
    0% {top: -10%; left: 100%;}
    50% {top: 20%; left: 20%;}
    93% {top: 20%; left: 20%;}
    100% {top: 0%; left: 5%;}
  }

  .ramp .marble, .bit .marble {
    animation-name: ramp;
    top: -10%;
    left: -10%;
  }

  .ramp.right .marble {
    animation-name: rampright;
    animation-duration: 0.4s
  }

  .ramp.right, .bit.right {
    transition-delay: 0.15s
  }

  @keyframes bit {
    0% {top: -10%; left: 0;}
    30% {top: 38%; left: 20%;}
    85% {top: 38%; left: 20%;}
    100% {top: 0%; left: 5%;}
  }

  @keyframes bitright {
    0% {top: -10%; left: 100%;}
    20% {top: 0; left: 60%;}
    50% {top: 38%; left: 20%;}
    90% {top: 38%; left: 20%;}
    100% {top: 0%; left: 5%;}
  }


  .bit .marble, .gearbit .marble {
    animation-name: bit;
    top: -10%;
    left: -10%;
  }

  .bit.right .marble, .gearbit.right .marble {
    animation-name: bitright;
    animation-duration: 0.4s
  }

  @keyframes crossover {
    0% {top: -10%; left: 0;}
    10% {top: 10%; left: 10%;}
    20% {top: 25%; left: 20%;}
    30% {top: 40%; left: 15%;}
    40% {top: 60%; left: 25%;}
    50% {top: 75%; left: 40%;}
    60% {top: 85%; left: 50%;}
    100% {left: 100%;}
  }

  @keyframes crossoverright {
    0% {top: -10%; left: 100%;}
    10% {top: 10%; left: 90%;}
    20% {top: 25%; left: 80%;}
    30% {top: 40%; left: 85%;}
    40% {top: 60%; left: 75%;}
    50% {top: 75%; left: 60%;}
    60% {top: 85%; left: 50%;}
    100% {left: 0;}
  }

  .crossover .marble {
    animation-name: crossover;
    top: 85%;
    left: 100%;
  }

  .crossover.right .marble {
    animation-name: crossoverright;
    top: 85%;
    left: 0;
  }

  @keyframes interceptor {
    0% {top: -10%; left: 0;}
    10% {top: 10%; left: 10%;}
    20% {top: 25%; left: 20%;}
    30% {left: 30%;}
    100% {left: 50%;}
  }

  @keyframes interceptorright {
    0% {top: -10%; left: 100%;}
    10% {top: 10%; left: 90%;}
    20% {top: 25%; left: 80%;}
    30% {left: 70%;}
    100% {left: 50%;}
  }

  .interceptor .marble {
    animation-name: interceptor;
    top: 25%;
    left: 50%;
  }
  .interceptor.right .marble{
    animation-name: interceptorright;
  }

  .position.empty {
    position: relative;
  }

  @keyframes empty {
    0% {top: -10%; left: 0;}
    10% {top: 0%; left: 10%;}
    25% {top: 25%; left: 25%;}
    35% {top: 50%; left: 35%;}
    50% {top: 100%; left: 50%; opacity: 1;}
    75% {top: 175%; left: 75%; opacity: 0.5;}
    100% {top: 300%; left: 100%; opacity: 0;}
  }

  @keyframes emptyright {
    0% {top: -10%; left: 100%;}
    10% {top: 0%; left: 90%;}
    25% {top: 25%; left: 75%;}
    35% {top: 50%; left: 65%;}
    50% {top: 100%; left: 50%; opacity: 1;}
    75% {top: 175%; left: 25%; opacity: 0.5;}
    100% {top: 300%; left: 00%; opacity: 0;}
  }
  
  .position.empty .marble, .gear .marble {
    animation-name: empty;
    animation-duration: 0.3s;
  }

  .position.empty .marble.right, .gear.right .marble {
    animation-name: emptyright;
    animation-duration: 0.3s;
  }
  

  #levers {
    display: flex;
    justify-content: space-around;
    gap: 1rem;
  }

  #board-actions {
    display: flex;
    justify-content: flex-start;
    padding: 0.25rem 0.5rem 0.5rem;
  }

  .solution-button {
    margin-right: 0.5rem;
  }

  .hint-button {
    margin-right: 0.5rem;
  }

  .reset-button {
    margin-left: auto;
  }

  .ghost-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    pointer-events: none;
    z-index: 0;
  }

  .ghost-part {
    opacity: 0.5;
  }

  #finish-line {
    margin: 0.2vh auto 0.6vh;
    width: 66vh;
    border-top: 2px dashed #c8c8c8;
    opacity: 0.7;
    pointer-events: none;
  }

  .lever-column {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    align-items: center;
  }

  .lever-blue {
    background-color: rgba(100, 160, 255, 0.25);
  }

  .lever-red {
    background-color: rgba(255, 120, 120, 0.25);
  }

  #start-ramps {
    display: flex;
    justify-content: space-between;
  }

  .marble-start {
    position: relative;
    width: 18vh;
    height: 6vh;
    overflow: visible;
    background-image: url(./images/bg-start.svg);
    background-image: url(../images/bg-start.svg);
    background-repeat: none;
    background-size: cover;
  }

  @media (max-aspect-ratio: 7/9) {
    .marble-start {
      width: 24vw;
      height: 8vw;
    }

  }

  @keyframes marblestart {
    0% {top: 0%; left: 0%;}
    100% {top: 90%; left: 106%;}
  }

  .marble-start .marble {
    animation-name: marblestart;
    animation-duration: 0.4s;
    animation-timing-function: ease-in;
    top: 90%;
    left: 106%;
  }

  .modal-image {
    width: 40px;
    height: 40px;
    margin: 1rem;
  }

  button {
    margin-bottom: 0;
  }

  @media (max-width: 576px), (max-height: 576px) {
    .marble-numbers {
      padding-bottom: 0.1em;
    }
    .marble-numbers img {
      height: 1em;
      width: 1em;
      margin-bottom: -0.2em;
    }

    button {
      font-size: 0.8em;
      padding: 0.1em;
    }
  }

  @media (max-aspect-ratio: 7/9) {
    #finish-line {
      width: 88vw;
      margin: 0.2vw auto 0.6vw;
    }
  }
</style>
