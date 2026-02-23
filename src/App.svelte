<script>
  import { onMount } from 'svelte';

  const planets = [
    { name: 'Mercury', au: 0.387, period: 87.97,   size: 3,  color: '#b5b5b5', ecc: 0.2056 },
    { name: 'Venus',   au: 0.723, period: 224.7,    size: 5,  color: '#e8cda0', ecc: 0.0068 },
    { name: 'Earth',   au: 1.0,   period: 365.25,   size: 5,  color: '#4da6ff', ecc: 0.0167 },
    { name: 'Mars',    au: 1.524, period: 687.0,     size: 4,  color: '#c1440e', ecc: 0.0934 },
    { name: 'Jupiter', au: 5.203, period: 4332.59,   size: 14, color: '#c88b3a', ecc: 0.0489 },
    { name: 'Saturn',  au: 9.537, period: 10759.22,  size: 12, color: '#e0c068', ecc: 0.0565 },
    { name: 'Uranus',  au: 19.19, period: 30688.5,   size: 8,  color: '#73c2c6', ecc: 0.0463 },
    { name: 'Neptune', au: 30.07, period: 60182.0,   size: 7,  color: '#3f54ba', ecc: 0.0095 },
  ];

  // Log scale so outer planets don't crush inner ones
  function auToRadius(au) {
    return 40 + 90 * Math.log2(1 + au);
  }

  const viewSize = 800;
  const cx = viewSize / 2;
  const cy = viewSize / 2;

  // Days from J2000 to today
  let time = $state(Math.floor((Date.now() - new Date(2000, 0, 1).getTime()) / 86400000));
  let speed = $state(0.1);
  let animFrame;
  let paused = $state(true);
  let selectedPlanet = $state(null);
  let showSightLines = $state(false);

  // J2000 epoch: Jan 1 2000
  const j2000 = new Date(2000, 0, 1);

  // J2000 mean longitudes (degrees)
  const j2000MeanLong = [252.25, 181.98, 100.46, 355.45, 34.40, 49.94, 313.23, 304.88];

  const zodiacSigns = [
    { name: 'Aries',       symbol: '\u2648' },
    { name: 'Taurus',      symbol: '\u2649' },
    { name: 'Gemini',      symbol: '\u264A' },
    { name: 'Cancer',      symbol: '\u264B' },
    { name: 'Leo',         symbol: '\u264C' },
    { name: 'Virgo',       symbol: '\u264D' },
    { name: 'Libra',       symbol: '\u264E' },
    { name: 'Scorpio',     symbol: '\u264F' },
    { name: 'Sagittarius', symbol: '\u2650' },
    { name: 'Capricorn',   symbol: '\u2651' },
    { name: 'Aquarius',    symbol: '\u2652' },
    { name: 'Pisces',      symbol: '\u2653' },
  ];

  const zodiacRadius = viewSize / 2 - 10;

  function currentDate() {
    const d = new Date(j2000);
    d.setDate(d.getDate() + Math.floor(time));
    return d;
  }

  function formatDate(d) {
    return d.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' });
  }

  function planetPosition(planet, idx) {
    const r = auToRadius(planet.au);
    const startAngle = j2000MeanLong[idx] * Math.PI / 180;
    const angle = startAngle + (2 * Math.PI * time) / planet.period;
    const rx = r * (1 + planet.ecc);
    const ry = r * (1 - planet.ecc);
    return {
      x: cx + rx * Math.cos(angle),
      y: cy + ry * Math.sin(angle),
      orbitRx: rx,
      orbitRy: ry,
    };
  }

  function animate() {
    if (!paused) {
      time += speed;
    }
    animFrame = requestAnimationFrame(animate);
  }

  onMount(() => {
    animFrame = requestAnimationFrame(animate);
    return () => cancelAnimationFrame(animFrame);
  });

  let positions = $derived(planets.map((p, i) => ({ ...p, ...planetPosition(p, i), idx: i })));
  let date = $derived(currentDate());
</script>

<main>
  <h1>Solar System</h1>

  <div class="controls">
    <button onclick={() => paused = !paused}>{paused ? 'Play' : 'Pause'}</button>
    <label>
      Speed
      <input type="range" min="-50" max="50" step="0.1" bind:value={speed} />
      <span>{speed > 0 ? '+' : ''}{speed.toFixed(1)}d/f</span>
    </label>
    <span class="date"><span class="day-num">{formatDate(date)}</span></span>
    <label class="checkbox-label">
      <input type="checkbox" bind:checked={showSightLines} />
      Sight lines
    </label>
  </div>

  <div class="solar-system">
    <svg viewBox="0 0 {viewSize} {viewSize}" xmlns="http://www.w3.org/2000/svg">
      <!-- Zodiac ring -->
      <circle cx={cx} cy={cy} r={zodiacRadius} fill="none" stroke="#2a2a3a" stroke-width="1" />
      {#each zodiacSigns as sign, i}
        {@const startAngle = (i * 30 - 90) * Math.PI / 180}
        {@const endAngle = ((i + 1) * 30 - 90) * Math.PI / 180}
        {@const midAngle = ((i + 0.5) * 30 - 90) * Math.PI / 180}
        <!-- sector line -->
        <line
          x1={cx + (zodiacRadius - 24) * Math.cos(startAngle)}
          y1={cy + (zodiacRadius - 24) * Math.sin(startAngle)}
          x2={cx + zodiacRadius * Math.cos(startAngle)}
          y2={cy + zodiacRadius * Math.sin(startAngle)}
          stroke="#2a2a3a"
          stroke-width="1"
        />
        <!-- symbol -->
        <text
          x={cx + (zodiacRadius - 12) * Math.cos(midAngle)}
          y={cy + (zodiacRadius - 12) * Math.sin(midAngle)}
          text-anchor="middle"
          dominant-baseline="central"
          fill="#778"
          font-size="20"
        >
          {sign.symbol}
        </text>
      {/each}

      {#each positions as p}
        <ellipse
          cx={cx} cy={cy}
          rx={p.orbitRx} ry={p.orbitRy}
          fill="none"
          stroke="#333"
          stroke-width="0.5"
          stroke-dasharray="4 4"
        />
      {/each}

      <!-- Sun -->
      <circle cx={cx} cy={cy} r="16" fill="#ffcc33" />
      <circle cx={cx} cy={cy} r="20" fill="none" stroke="#ffcc3344" stroke-width="4" />

      <!-- Sight lines from Earth -->
      {#if showSightLines}
        {@const earth = positions[2]}
        {#each positions as p}
          {#if p.name !== 'Earth'}
            {@const dx = p.x - earth.x}
            {@const dy = p.y - earth.y}
            {@const angle = Math.atan2(dy, dx)}
            {@const ex = cx + zodiacRadius * Math.cos(angle)}
            {@const ey = cy + zodiacRadius * Math.sin(angle)}
            <line
              x1={earth.x} y1={earth.y}
              x2={ex} y2={ey}
              stroke={p.color}
              stroke-width="0.7"
              stroke-opacity="0.6"
            />
            <circle cx={ex} cy={ey} r="3" fill={p.color} opacity="0.8" />
          {/if}
        {/each}
      {/if}

      {#each positions as p}
        <circle
          cx={p.x} cy={p.y} r={p.size}
          fill={p.color}
          stroke={selectedPlanet === p.name ? '#fff' : 'none'}
          stroke-width="2"
          class="planet"
          onmouseenter={() => selectedPlanet = p.name}
          onmouseleave={() => selectedPlanet = null}
          role="img"
          aria-label={p.name}
        />
        <text
          x={p.x} y={p.y - p.size - 6}
          text-anchor="middle"
          fill={selectedPlanet === p.name ? '#fff' : '#aaa'}
          font-size={selectedPlanet === p.name ? '12' : '10'}
          opacity={selectedPlanet === p.name ? '1' : '0.7'}
        >
          {p.name}
        </text>
      {/each}
    </svg>
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    background: #0a0a1a;
    color: #eee;
    font-family: system-ui, sans-serif;
    display: flex;
    justify-content: center;
  }

  main {
    text-align: center;
    padding: 1rem;
    max-width: 850px;
  }

  h1 {
    font-size: 1.5rem;
    margin: 0.5rem 0;
    color: #ffcc33;
  }

  .controls {
    display: flex;
    align-items: center;
    gap: 1rem;
    justify-content: center;
    flex-wrap: wrap;
    margin-bottom: 0.5rem;
    font-size: 0.85rem;
  }

  .controls button {
    background: #222;
    color: #eee;
    border: 1px solid #444;
    padding: 0.3rem 0.8rem;
    border-radius: 4px;
    cursor: pointer;
    min-width: 5rem;
  }

  .controls button:hover {
    background: #333;
  }

  .controls label {
    display: flex;
    align-items: center;
    gap: 0.4rem;
  }

  .controls input[type="range"] {
    width: 120px;
  }

  .checkbox-label {
    display: flex;
    align-items: center;
    gap: 0.3rem;
    cursor: pointer;
  }

  .date {
    color: #888;
  }

  .day-num {
    font-variant-numeric: tabular-nums;
    display: inline-block;
    min-width: 14ch;
    text-align: left;
  }

  .solar-system {
    width: 100%;
    max-width: 800px;
    margin: 0 auto;
  }

  svg {
    width: 100%;
    height: auto;
    background: radial-gradient(circle at 50% 50%, #111122 0%, #0a0a1a 70%);
    border-radius: 8px;
  }

  .planet {
    cursor: pointer;
    transition: r 0.15s;
  }

  .planet:hover {
    filter: brightness(1.3);
  }
</style>
