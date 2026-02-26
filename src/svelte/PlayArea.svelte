<script>
  import GameBoard from './GameBoard.svelte';
  import MarbleTray from './MarbleTray.svelte';
  import PartsTray from  './PartsTray.svelte';
  import Hand from './Hand.svelte';

  import { holding, currentChallenge, basePath } from '../store.js';
  import { board } from '../board.js';
  import { marbles } from '../marbles.js';
  import { parts } from '../parts.js';
  import { socket } from '../socket.js';

  let mousePosition = {};
  let lastGrab = {x: false, y: false, timeout: false};
  let boardElement;
  let gameBoard;
  let escapedMarbles = [];
  let winActive = false;
  let lastWin = false;
  let winTimer;
  const confettiCount = 24;

  const resultsDisplayReversed = true;
  const goalDisplayReversed = false;

  function normalizeSequence(sequence, reversed) {
    return reversed ? [...sequence].reverse() : sequence;
  }

  function isWin() {
    if (!$currentChallenge?.output) return false;
    if ($marbles.results.length !== $currentChallenge.output.length) return false;
    const resultSeq = normalizeSequence($marbles.results.map(m => m.color), resultsDisplayReversed);
    const goalSeq = normalizeSequence([...$currentChallenge.output], goalDisplayReversed);
    for (let i = 0; i < goalSeq.length; i++) {
      const expected = goalSeq[i] === 'b' ? 'blue' : 'red';
      if (resultSeq[i] !== expected) return false;
    }
    return true;
  }

  $: {
    const nowWin = isWin();
    if ($currentChallenge?.output) {
      console.log('win-check', {
        nowWin,
        results: $marbles.results.map(m => m.color),
        goal: $currentChallenge.output,
        resultsDisplayReversed,
        goalDisplayReversed
      });
    }
    if (nowWin && !lastWin) {
      winActive = true;
      if (winTimer) clearTimeout(winTimer);
      winTimer = setTimeout(() => { winActive = false; }, 4000);
    }
    lastWin = nowWin;
  }


  function getBoardPosition(pageX, pageY) {
    // Returns x,y coordinates based on the position on the board element that the user has selected.
    let boardRect = boardElement.getBoundingClientRect();
    let x = Math.floor((pageX - (boardRect.left + window.scrollX)) / (boardRect.width / $board[0].length));
    let y = Math.floor((pageY - (boardRect.top + window.scrollY)) / (boardRect.height / $board.length));
    if (pageX > (boardRect.left + window.scrollX) && pageX < (boardRect.right + window.scrollX) &&
        pageY > (boardRect.top + window.scrollY) && pageY < (boardRect.bottom + window.scrollY) &&
        $board.isValid(x, y)) return [x, y];
    else return false;
  }

  function setMousePosition(e) {
    mousePosition.left = e.pageX;
    mousePosition.top = e.pageY;
  }

  function touchMove(e, force=false) {
    if (e.touches.length === 1 && ($holding || force)) {
      setMousePosition(e.touches[0]);
      e.preventDefault();
      e.stopPropagation();
    }
  }

  function drop(e, force=false) {
    // Places the currently held part on the board, otherwise returns it to the parts tray.
    if ($holding && ((e.type === 'mouseup' && e.button === 0) ||
        (e.type === 'touchend' && e.changedTouches.length === 1 && e.touches.length === 0) || force)) {
      let boardPosition;
      if (e.type === 'touchend') boardPosition = getBoardPosition(e.changedTouches[0].pageX, e.changedTouches[0].pageY);
      else boardPosition = getBoardPosition(e.pageX, e.pageY);
      if (!$board.marble && boardPosition && (!$holding.requiresSlot || $board.hasSlot(...boardPosition)) &&
          !$board[boardPosition[1]][boardPosition[0]]) {
        $board[boardPosition[1]][boardPosition[0]] = $holding;
        if (lastGrab.x === boardPosition[0] && lastGrab.y === boardPosition[1]) {
          $board.flip(...boardPosition);
          console.log(`Flipped ${$holding.name}`);
        }
        console.log(`Placed ${$holding.name}`);
        let flipableNeighbors = Array.from($board.flipableNeighbors(...boardPosition));
        if (flipableNeighbors.length > 1) flipableNeighbors.forEach(part => part.facing = flipableNeighbors[1].facing);
        
        // flip every gear in a non-slot position, so they animate correctly
        for (let row = 0; row < 11; ++row) {
          for (let col = 0; col < 11; ++col) {
            if (!$board.hasSlot(col, row) && $board[col][row]) {
              $board[col][row].facing = $board[col][row].facing + 1 % 2;
            }
          }
        }
      }else{
        $parts.find(part => part.name === $holding.name).count++;
        $parts = $parts;
        console.log(`Dropped ${$holding.name}`);
      }
      if (lastGrab.timeout) window.clearTimeout(lastGrab.timeout)
      $holding = false;
      $board = $board;
      socket.sendBoard();
      e.preventDefault();
      e.stopPropagation();
    }
  }

</script>

<div id="play-area" class:grabbed={$holding}
    on:mousemove={setMousePosition}
    on:mouseup={drop}
    on:mouseleave="{e => drop(e, true)}"
    on:touchmove={touchMove}
    on:touchend={drop}>
  {#if winActive}
    <div class="win-overlay" aria-live="polite">
      <div class="win-banner">You Win</div>
      <div class="confetti" style="--count:{confettiCount}">
        {#each Array.from({ length: confettiCount }) as _, i (i)}
          <span class="confetti-piece" style="--i:{i}"></span>
        {/each}
      </div>
    </div>
  {/if}
  <GameBoard bind:this={gameBoard} bind:escapedMarbles bind:boardElement bind:lastGrab
    on:touch="{e => touchMove(e.detail)}"/>
  <div id="side-panel">
    <div class="tray-row">
      <div class="tray-container" id="results-tray">
        <div class="tray-label">Results</div>
        <div class="tray-body">
          <MarbleTray result={true} direction="right" marbles={$marbles.results} slots={16}/>
        </div>
      </div>
      <div class="tray-container" id="goal-tray">
        <div class="tray-label">Goal</div>
        <div class="tray-body">
          {#if $currentChallenge?.output}
            <div class="goal-row">
              {#each $currentChallenge.output as marble, i (i)}
                <img class="goal-marble" src="{$basePath}images/marble{marble === 'b' ? 'blue' : 'red'}.svg" alt="Goal Marble">
              {/each}
            </div>
          {:else}
            <div class="goal-row empty"></div>
          {/if}
        </div>
      </div>
    </div>
    <div class="tray-container" id="escaped-tray">
      <div class="tray-label">Escaped</div>
      <div class="tray-body">
        <MarbleTray result={true} direction="right" marbles={escapedMarbles} slots={16}/>
      </div>
    </div>
    <PartsTray on:touch="{e => touchMove(e.detail, true)}"/>
  </div>
</div>
<Hand {mousePosition}/>

<style>
  #play-area {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: flex-start;
    width: 92%;
    margin: auto;
    position: relative;
  }
  #side-panel {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    margin-left: 0.5rem;
    align-items: stretch;
  }
  .tray-row {
    display: flex;
    gap: 0.5rem;
    align-items: stretch;
  }
  .tray-container {
    display: flex;
    flex-direction: column;
    gap: 0.3rem;
    min-height: 2vh;
    padding: 0.25rem 0.5rem;
    border: 1px dashed rgba(0, 0, 0, 0.2);
    border-radius: 6px;
    background: rgba(255, 255, 255, 0.75);
  }
  .tray-label {
    font-size: 0.9rem;
    font-weight: 600;
  }
  .tray-body {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.5rem;
    min-height: 1.6vh;
    width: 28vh;
  }
  .goal-row {
    display: flex;
    flex-direction: row;
    justify-content: flex-start;
    gap: 0.1rem;
    width: 100%;
  }
  .goal-row.empty {
    min-height: 1.6vh;
  }
  .goal-marble {
    width: 1.44vh;
    height: 1.44vh;
  }
  .win-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    pointer-events: none;
    z-index: 10;
  }
  .win-banner {
    font-family: 'Orbitron', sans-serif;
    font-size: 2rem;
    padding: 0.4rem 0.9rem;
    border-radius: 0.5rem;
    color: #113355;
    background: rgba(240, 248, 255, 0.9);
    border: 2px solid rgba(30, 90, 160, 0.35);
    box-shadow: 0 6px 18px rgba(0, 0, 0, 0.12);
    animation: win-pop 0.3s ease-out;
  }
  .confetti {
    position: absolute;
    inset: 0;
    overflow: hidden;
  }
  .confetti-piece {
    position: absolute;
    top: -10%;
    left: calc((var(--i) + 1) * (100% / (var(--count) + 1)));
    width: 0.5rem;
    height: 0.8rem;
    opacity: 0.9;
    border-radius: 0.1rem;
    background: #5aa2ff;
    animation: confetti-fall 3.2s ease-out forwards;
    animation-delay: calc(var(--i) * 0.035s);
    transform: rotate(calc(var(--i) * 23deg));
  }
  .confetti-piece:nth-child(3n) { background: #ff7b7b; }
  .confetti-piece:nth-child(3n+1) { background: #ffd166; }
  .confetti-piece:nth-child(3n+2) { background: #7bdff6; }
  @keyframes win-pop {
    0% { transform: scale(0.9); opacity: 0; }
    100% { transform: scale(1); opacity: 1; }
  }
  @keyframes confetti-fall {
    0% { transform: translateY(0) rotate(0deg); opacity: 0.9; }
    45% { transform: translateY(120vh) rotate(360deg); opacity: 0; }
    50% { transform: translateY(0) rotate(0deg); opacity: 0.9; }
    95% { transform: translateY(120vh) rotate(360deg); opacity: 0; }
    100% { transform: translateY(120vh) rotate(360deg); opacity: 0; }
  }
  @media (max-aspect-ratio: 7/9) {
    .goal-marble {
      width: 1.92vw;
      height: 1.92vw;
    }
  }
  :global(#side-panel #marble-tray) {
    width: 100%;
  }
  @media (max-aspect-ratio: 7/9) {
    #play-area {
      flex-direction: column;
      align-items: center;
    }
    #side-panel {
      margin-left: 0;
      width: 100%;
      align-items: center;
    }
    .tray-row {
      flex-direction: column;
      align-items: center;
      width: 100%;
    }
    .tray-container {
      width: min(90vw, 70vh);
    }
    .tray-body {
      width: min(90vw, 70vh);
    }
  }
  .grabbed {
    cursor: grabbing;
  }
</style>
