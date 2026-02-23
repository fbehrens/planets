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

  const zodiacRadius = viewSize / 2 - 4;
  const zodiacInner = zodiacRadius - 50;

  // Simplified constellation patterns: stars [x,y] in 0..1 range, edges [i,j]
  const constellations = [
    // Aries
    { stars: [[0,0.5],[0.4,0.3],[0.7,0.35],[1,0.5]], edges: [[0,1],[1,2],[2,3]] },
    // Taurus - V shape
    { stars: [[0,0],[0.3,0.4],[0.5,0.7],[0.7,0.4],[1,0],[0.5,0.5]], edges: [[0,1],[1,5],[5,2],[5,3],[3,4]] },
    // Gemini - two parallel lines
    { stars: [[0.2,0],[0.2,0.4],[0.2,0.8],[0.7,0],[0.7,0.4],[0.7,0.8],[0.2,0.4],[0.7,0.4]], edges: [[0,1],[1,2],[3,4],[4,5],[1,4]] },
    // Cancer - inverted Y
    { stars: [[0,0.3],[0.35,0.5],[0.65,0.5],[1,0.3],[0.35,0.8],[0.65,0.2]], edges: [[0,1],[1,2],[2,3],[1,4],[2,5]] },
    // Leo - sickle
    { stars: [[0,0.8],[0.2,0.4],[0.4,0.2],[0.6,0.3],[0.5,0.6],[0.8,0.7],[1,0.7]], edges: [[0,1],[1,2],[2,3],[3,4],[4,1],[4,5],[5,6]] },
    // Virgo - Y with arms
    { stars: [[0,0.3],[0.3,0.4],[0.5,0.5],[0.7,0.3],[1,0.2],[0.5,0.8],[0.8,0.7]], edges: [[0,1],[1,2],[2,3],[3,4],[2,5],[5,6]] },
    // Libra - balance
    { stars: [[0.2,0.2],[0.5,0],[0.8,0.2],[0.5,0.5],[0.2,0.8],[0.8,0.8]], edges: [[0,1],[1,2],[0,3],[2,3],[3,4],[3,5]] },
    // Scorpio - J curve
    { stars: [[0,0.4],[0.2,0.3],[0.4,0.35],[0.6,0.5],[0.7,0.7],[0.85,0.85],[1,0.7]], edges: [[0,1],[1,2],[2,3],[3,4],[4,5],[5,6]] },
    // Sagittarius - teapot
    { stars: [[0.3,0],[0.5,0.2],[0.7,0],[0.8,0.4],[0.5,0.6],[0.2,0.4],[0.5,0.9]], edges: [[0,1],[1,2],[2,3],[3,4],[4,5],[5,0],[4,6]] },
    // Capricorn - triangle
    { stars: [[0,0.5],[0.3,0.2],[0.7,0.1],[1,0.3],[0.8,0.7],[0.4,0.8]], edges: [[0,1],[1,2],[2,3],[3,4],[4,5],[5,0]] },
    // Aquarius - zigzag
    { stars: [[0,0.3],[0.25,0.5],[0.5,0.3],[0.75,0.5],[1,0.3],[0.25,0.8],[0.75,0.8]], edges: [[0,1],[1,2],[2,3],[3,4],[1,5],[3,6]] },
    // Pisces - two fish connected
    { stars: [[0,0.5],[0.2,0.2],[0.3,0.5],[0.5,0.5],[0.7,0.5],[0.8,0.2],[1,0.5],[0.8,0.8]], edges: [[0,1],[1,2],[2,3],[3,4],[4,5],[5,6],[4,7]] },
  ];

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

  // Which zodiac sign is each planet in, as seen from Earth?
  let planetsInSign = $derived(() => {
    const earth = positions[2];
    const map = Array.from({ length: 12 }, () => []);
    for (const p of positions) {
      if (p.name === 'Earth') continue;
      const angle = Math.atan2(p.y - earth.y, p.x - earth.x);
      // Convert to degrees 0-360, offset so 0° = top (-90°)
      let deg = ((angle * 180 / Math.PI) + 90 + 360) % 360;
      const signIdx = Math.floor(deg / 30);
      map[signIdx].push(p);
    }
    return map;
  });

  let hoveredSign = $state(null);
  let tooltipX = $state(0);
  let tooltipY = $state(0);

  function handleSignEnter(e, i) {
    hoveredSign = i;
    updateTooltipPos(e);
  }

  function handleSignMove(e) {
    updateTooltipPos(e);
  }

  function handleSignLeave() {
    hoveredSign = null;
  }

  function updateTooltipPos(e) {
    const rect = e.currentTarget.closest('svg').getBoundingClientRect();
    tooltipX = e.clientX - rect.left;
    tooltipY = e.clientY - rect.top;
  }
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
      <circle cx={cx} cy={cy} r={zodiacInner} fill="none" stroke="#2a2a3a" stroke-width="0.5" />
      {#each zodiacSigns as sign, i}
        {@const startAngle = (i * 30 - 90) * Math.PI / 180}
        {@const endAngle = ((i + 1) * 30 - 90) * Math.PI / 180}
        {@const midAngle = ((i + 0.5) * 30 - 90) * Math.PI / 180}
        <!-- sector line -->
        <line
          x1={cx + zodiacInner * Math.cos(startAngle)}
          y1={cy + zodiacInner * Math.sin(startAngle)}
          x2={cx + zodiacRadius * Math.cos(startAngle)}
          y2={cy + zodiacRadius * Math.sin(startAngle)}
          stroke="#2a2a3a"
          stroke-width="1"
        />
        <!-- Unicode symbol (outer) -->
        <text
          x={cx + (zodiacRadius - 12) * Math.cos(midAngle)}
          y={cy + (zodiacRadius - 12) * Math.sin(midAngle)}
          text-anchor="middle"
          dominant-baseline="central"
          fill={hoveredSign === i ? '#ccf' : '#778'}
          font-size="14"
        >
          {sign.symbol}
        </text>
        <!-- Constellation pattern (inner band) -->
        {@const cst = constellations[i]}
        {@const cMidR = (zodiacInner + zodiacRadius) / 2 - 4}
        {@const cSize = 22}
        {@const cCx = cx + cMidR * Math.cos(midAngle) - cSize / 2}
        {@const cCy = cy + cMidR * Math.sin(midAngle) - cSize / 2}
        {@const rotDeg = midAngle * 180 / Math.PI}
        <g opacity={hoveredSign === i ? 1 : 0.5}>
          {#each cst.edges as [a, b]}
            <line
              x1={cCx + cst.stars[a][0] * cSize}
              y1={cCy + cst.stars[a][1] * cSize}
              x2={cCx + cst.stars[b][0] * cSize}
              y2={cCy + cst.stars[b][1] * cSize}
              stroke={hoveredSign === i ? '#aac' : '#556'}
              stroke-width="0.8"
            />
          {/each}
          {#each cst.stars as [sx, sy]}
            <circle
              cx={cCx + sx * cSize}
              cy={cCy + sy * cSize}
              r="1.5"
              fill={hoveredSign === i ? '#ddf' : '#889'}
            />
          {/each}
        </g>
        <!-- invisible hover sector -->
        {@const r1 = zodiacInner}
        {@const r2 = zodiacRadius}
        <path
          d="M {cx + r1 * Math.cos(startAngle)} {cy + r1 * Math.sin(startAngle)}
             L {cx + r2 * Math.cos(startAngle)} {cy + r2 * Math.sin(startAngle)}
             A {r2} {r2} 0 0 1 {cx + r2 * Math.cos(endAngle)} {cy + r2 * Math.sin(endAngle)}
             L {cx + r1 * Math.cos(endAngle)} {cy + r1 * Math.sin(endAngle)}
             A {r1} {r1} 0 0 0 {cx + r1 * Math.cos(startAngle)} {cy + r1 * Math.sin(startAngle)}
             Z"
          fill="transparent"
          class="zodiac-sector"
          onmouseenter={(e) => handleSignEnter(e, i)}
          onmousemove={handleSignMove}
          onmouseleave={handleSignLeave}
        />
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

    {#if hoveredSign !== null}
      <div class="tooltip" style="left: {tooltipX}px; top: {tooltipY}px;">
        <div class="tooltip-header">
          <span class="tooltip-symbol">{zodiacSigns[hoveredSign].symbol}</span>
          {zodiacSigns[hoveredSign].name}
          <svg class="tooltip-constellation" width="40" height="40" viewBox="0 0 1 1">
            {#each constellations[hoveredSign].edges as [a, b]}
              <line x1={constellations[hoveredSign].stars[a][0]} y1={constellations[hoveredSign].stars[a][1]} x2={constellations[hoveredSign].stars[b][0]} y2={constellations[hoveredSign].stars[b][1]} stroke="#aac" stroke-width="0.04" />
            {/each}
            {#each constellations[hoveredSign].stars as [sx, sy]}
              <circle cx={sx} cy={sy} r="0.06" fill="#ddf" />
            {/each}
          </svg>
        </div>
        {#if planetsInSign()[hoveredSign].length > 0}
          <div class="tooltip-planets">
            {#each planetsInSign()[hoveredSign] as p}
              <span class="tooltip-planet">
                <svg width="14" height="14" viewBox="0 0 14 14">
                  <circle cx="7" cy="7" r="5" fill={p.color} />
                </svg>
                {p.name}
              </span>
            {/each}
          </div>
        {:else}
          <div class="tooltip-empty">No planets</div>
        {/if}
      </div>
    {/if}
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
    position: relative;
  }

  .zodiac-sector {
    cursor: pointer;
  }

  .tooltip {
    position: absolute;
    pointer-events: none;
    background: #1a1a2eee;
    border: 1px solid #444;
    border-radius: 6px;
    padding: 0.5rem 0.7rem;
    font-size: 0.8rem;
    white-space: nowrap;
    transform: translate(10px, -100%);
    z-index: 10;
  }

  .tooltip-header {
    font-weight: bold;
    margin-bottom: 0.3rem;
    display: flex;
    align-items: center;
    gap: 0.3rem;
  }

  .tooltip-symbol {
    font-size: 1.2rem;
  }

  .tooltip-constellation {
    margin-left: 0.3rem;
    background: #0a0a1a;
    border-radius: 3px;
    padding: 2px;
  }

  .tooltip-planets {
    display: flex;
    flex-direction: column;
    gap: 0.2rem;
  }

  .tooltip-planet {
    display: flex;
    align-items: center;
    gap: 0.3rem;
  }

  .tooltip-empty {
    color: #666;
    font-style: italic;
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
