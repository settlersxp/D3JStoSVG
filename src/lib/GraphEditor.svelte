<script lang="ts">
  import { tick } from 'svelte';

  /* ---------------------
   * Parameters – exposed as component props
   * --------------------- */
  // Receive props in runes mode
  const {
    width: widthProp = 1024,
    height: heightProp = 760,
    circleDiameter: cdProp = 60,
    edgeWidth: ewProp = 5
  } = $props<{
    width?: number;
    height?: number;
    circleDiameter?: number;
    edgeWidth?: number;
  }>();

  // Canvas dimensions (mutable)
  let width = $state(widthProp);
  let height = $state(heightProp);

  // Global defaults (can be overridden for every circle / edge)
  let circleDiameter = $state(cdProp);
  let edgeWidth = $state(ewProp);

  let backgroundImage = $state<string | null>(null);
  let fileInput: HTMLInputElement;
  let exportWithBackground = $state(false);

  /* ---------------------
   * Internal state
   * --------------------- */
  type CustomAttrs = Record<string, string>;

  type Circle = {
    id: number;
    x: number;
    y: number;
    diameter: number;
    fill: string;
    stroke: string;
    strokeWidth: number;
    customAttrs?: CustomAttrs;
  };

  type Edge = {
    id: number;
    from: number; // circle id
    to: number;   // circle id
    width: number;
    color: string;
    customAttrs?: CustomAttrs;
  };

  let circles = $state<Circle[]>([]);
  let edges = $state<Edge[]>([]);

  // helper counters for ids
  let nextCircleId = $state(1);
  let nextEdgeId = $state(1);

  // Currently active drag state
  let dragCircleId: number | null = $state(null);
  let isCreatingEdge = $state(false);
  let startCircleId: number | null = $state(null);
  let tempX = $state(0);
  let tempY = $state(0);

  // selection for editing
  let selectedCircleId = $state<number | null>(null);
  let selectedEdgeId = $state<number | null>(null);

  // state for new custom attribute input fields
  let newCircleAttrKey = $state('');
  let newCircleAttrVal = $state('');
  let newEdgeAttrKey = $state('');
  let newEdgeAttrVal = $state('');

  // Editor mode: 'draw' | 'customize'
  let mode = $state<'draw' | 'customize'>('draw');

  function toggleMode() {
    mode = mode === 'draw' ? 'customize' : 'draw';
    if (mode === 'draw') {
      // close side panel when leaving customize
      selectedCircleId = null;
      selectedEdgeId = null;
    }
  }

  /* ---------------------
   * Initialise with one root circle
   * --------------------- */
  $effect(() => {
    if (circles.length === 0) {
      const root: Circle = {
        id: nextCircleId,
        x: width / 2,
        y: height / 2,
        diameter: circleDiameter,
        fill: '#4cc9f0',
        stroke: '#1d3557',
        strokeWidth: edgeWidth
      };
      circles = [...circles, root];
      nextCircleId += 1;
    }
  });

  /* ---------------------
   * Event handlers
   * --------------------- */

  function pointerDownOnCircle(event: PointerEvent, circleId: number) {
    event.stopPropagation();
    if (mode !== 'draw') {
      // In customize mode we don't drag/create, just allow click handler to select
      return;
    }
    const isAlt = event.altKey || event.metaKey;

    if (isAlt) {
      // Start edge-creation drag
      isCreatingEdge = true;
      startCircleId = circleId;
      const svg = (event.currentTarget as SVGGraphicsElement).ownerSVGElement;
      svg?.setPointerCapture(event.pointerId);
      tempX = event.offsetX;
      tempY = event.offsetY;
    } else {
      // Start move drag
      dragCircleId = circleId;
      const svg = (event.currentTarget as SVGGraphicsElement).ownerSVGElement;
      svg?.setPointerCapture(event.pointerId);
    }
  }

  function pointerMove(event: PointerEvent) {
    if (mode !== 'draw') return;
    if (dragCircleId !== null) {
      circles = circles.map((c) =>
        c.id === dragCircleId ? { ...c, x: event.offsetX, y: event.offsetY } : c
      );
    } else if (isCreatingEdge && startCircleId !== null) {
      tempX = event.offsetX;
      tempY = event.offsetY;
    }
  }

  function pointerUp(event: PointerEvent) {
    if (mode !== 'draw') return;
    const svg = event.currentTarget as SVGSVGElement;
    svg.releasePointerCapture(event.pointerId);

    if (dragCircleId !== null) {
      dragCircleId = null;
      return;
    }

    if (isCreatingEdge && startCircleId !== null) {
      // Create new circle at pointer position
      const newCircle: Circle = {
        id: nextCircleId,
        x: event.offsetX,
        y: event.offsetY,
        diameter: circleDiameter,
        fill: '#ffbe0b',
        stroke: '#8338ec',
        strokeWidth: edgeWidth
      };
      circles = [...circles, newCircle];
      // Create edge
      const newEdge: Edge = {
        id: nextEdgeId,
        from: startCircleId,
        to: newCircle.id,
        width: edgeWidth,
        color: '#000'
      };
      edges = [...edges, newEdge];

      nextCircleId += 1;
      nextEdgeId += 1;
    }

    // reset
    isCreatingEdge = false;
    startCircleId = null;
  }

  /* ---------------------
   * UI helpers
   * --------------------- */
  function addRootCircle() {
    const c: Circle = {
      id: nextCircleId,
      x: width / 2,
      y: height / 2,
      diameter: circleDiameter,
      fill: '#4cc9f0',
      stroke: '#1d3557',
      strokeWidth: edgeWidth
    };
    circles = [...circles, c];
    nextCircleId += 1;
  }

  async function exportSVG() {
    await tick();
    // clone current svg string
    const svgElement = document.getElementById('graph-svg') as unknown as SVGSVGElement | null;
    if (!svgElement) return;

    // If we're exporting without background, temporarily hide it
    const bgImage = svgElement.querySelector('image');
    if (bgImage && !exportWithBackground) {
      bgImage.style.display = 'none';
    }

    const serializer = new XMLSerializer();
    const source = serializer.serializeToString(svgElement);
    
    // Restore background visibility if needed
    if (bgImage && !exportWithBackground) {
      bgImage.style.display = '';
    }
    
    const blob = new Blob([source], { type: 'image/svg+xml;charset=utf-8' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'graph.svg';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  }

  function selectCircle(id: number) {
    if (mode === 'customize') {
      selectedCircleId = id;
      selectedEdgeId = null;
    }
  }

  function selectEdge(id: number) {
    if (mode === 'customize') {
      selectedEdgeId = id;
      selectedCircleId = null;
    }
  }

  // Helpers to update attributes
  function updateCircleField(id: number, field: keyof Circle, value: any) {
    circles = circles.map((c) => (c.id === id ? { ...c, [field]: value } : c));
  }

  function updateCircleCustom(id: number, key: string, value: string) {
    circles = circles.map((c) =>
      c.id === id
        ? { ...c, customAttrs: { ...(c.customAttrs ?? {}), [key]: value } }
        : c
    );
  }

  function removeCircleCustom(id: number, key: string) {
    circles = circles.map((c) => {
      if (c.id !== id) return c;
      const ca = { ...(c.customAttrs ?? {}) };
      delete ca[key];
      return { ...c, customAttrs: ca };
    });
  }

  function updateEdgeField(id: number, field: keyof Edge, value: any) {
    edges = edges.map((e) => (e.id === id ? { ...e, [field]: value } : e));
  }

  function updateEdgeCustom(id: number, key: string, value: string) {
    edges = edges.map((e) =>
      e.id === id ? { ...e, customAttrs: { ...(e.customAttrs ?? {}), [key]: value } } : e
    );
  }

  function removeEdgeCustom(id: number, key: string) {
    edges = edges.map((e) => {
      if (e.id !== id) return e;
      const ca = { ...(e.customAttrs ?? {}) };
      delete ca[key];
      return { ...e, customAttrs: ca };
    });
  }

  // derived selected elements
  const selectedCircle = $derived(() =>
    selectedCircleId !== null ? circles.find((c) => c.id === selectedCircleId) : null
  );
  const selectedEdge = $derived(() =>
    selectedEdgeId !== null ? edges.find((e) => e.id === selectedEdgeId) : null
  );

  function triggerFileUpload() {
    fileInput.click();
  }

  function handleFileSelect(event: Event) {
    const target = event.target as HTMLInputElement;
    const file = target.files?.[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = (e) => {
        backgroundImage = e.target?.result as string;
      };
      reader.readAsDataURL(file);
    }
  }

  function addCircleAttribute() {
    if (newCircleAttrKey && selectedCircle()) {
      updateCircleCustom(selectedCircle()!.id, newCircleAttrKey, newCircleAttrVal);
      newCircleAttrKey = '';
      newCircleAttrVal = '';
    }
  }

  function addEdgeAttribute() {
    if (newEdgeAttrKey && selectedEdge()) {
      updateEdgeCustom(selectedEdge()!.id, newEdgeAttrKey, newEdgeAttrVal);
      newEdgeAttrKey = '';
      newEdgeAttrVal = '';
    }
  }

  function deleteCircle(id: number) {
    // remove the circle and any connected edges
    circles = circles.filter((c) => c.id !== id);
    edges = edges.filter((e) => e.from !== id && e.to !== id);
    selectedCircleId = null;
  }

  function deleteEdge(id: number) {
    // remove just the edge
    edges = edges.filter((e) => e.id !== id);
    selectedEdgeId = null;
  }
</script>

<style>
  .container {
    position: relative;
    background: white;
    border: 1px solid #ccc;
    overflow: hidden;
  }
  .toolbar {
    position: absolute;
    top: 8px;
    left: 8px;
    background: rgba(255, 255, 255, 0.9);
    border: 1px solid #ddd;
    padding: 6px 8px;
    border-radius: 6px;
    display: flex;
    gap: 4px;
    z-index: 10;
  }
</style>

<div class="container" style="width:{width}px;height:{height}px">
  <input type="file" accept="image/*" style="display:none" bind:this={fileInput} onchange={handleFileSelect} />
  <div class="toolbar">
    <label>W <input type="number" min="100" bind:value={width} style="width:70px" /></label>
    <label>H <input type="number" min="100" bind:value={height} style="width:70px" /></label>
    <label>Ø <input type="number" min="1" bind:value={circleDiameter} style="width:50px" title="Circle Diameter" /></label>
    <label>Line <input type="number" min="1" bind:value={edgeWidth} style="width:50px" title="Edge Width" /></label>
    <button onclick={addRootCircle} disabled={mode!=='draw'}>+ Root</button>
    <button onclick={exportSVG}>Export SVG</button>
    <label title="Include background in export"><input type="checkbox" bind:checked={exportWithBackground} /> BG</label>
    <button onclick={toggleMode}>{mode === 'draw' ? 'Customize' : 'Draw'}</button>
    <button onclick={triggerFileUpload}>Upload BG</button>
  </div>

  <svg
    id="graph-svg"
    width={width}
    height={height}
    viewBox={`0 0 ${width} ${height}`}
    pointer-events="all"
    onpointermove={pointerMove}
    onpointerup={pointerUp}
  >
    <!-- background -->
    <rect width={width} height={height} fill="white" />
    {#if backgroundImage}
      <image href={backgroundImage} x="0" y="0" width={width} height={height} />
    {/if}

    <!-- edges -->
    {#each edges as e (e.id)}
      <line
        x1={circles.find((c) => c.id === e.from)?.x}
        y1={circles.find((c) => c.id === e.from)?.y}
        x2={circles.find((c) => c.id === e.to)?.x}
        y2={circles.find((c) => c.id === e.to)?.y}
        stroke={e.color}
        stroke-width={e.width}
        onclick={() => selectEdge(e.id)}
        {...(e.customAttrs ?? {})}
      />
    {/each}

    <!-- temp edge while creating -->
    {#if isCreatingEdge && startCircleId !== null}
      <line
        x1={circles.find((c) => c.id === startCircleId)?.x}
        y1={circles.find((c) => c.id === startCircleId)?.y}
        x2={tempX}
        y2={tempY}
        stroke="#999"
        stroke-dasharray="4 4"
        stroke-width={edgeWidth}
      />
    {/if}

    <!-- circles -->
    {#each circles as c (c.id)}
      {#key c.id}
        <circle
          cx={c.x}
          cy={c.y}
          r={c.diameter / 2}
          fill={c.fill}
          stroke={c.stroke}
          stroke-width={c.strokeWidth}
          onpointerdown={(e) => pointerDownOnCircle(e, c.id)}
          onclick={() => selectCircle(c.id)}
          {...(c.customAttrs ?? {})}
        />
      {/key}
    {/each}
  </svg>

  <!-- Side panel -->
  {#if mode==='customize' && (selectedCircle() || selectedEdge())}
    <div
      class="sidepanel"
      style="position:absolute; top:0; right:0; width:260px; height:100%; background:#f9f9f9; border-left:1px solid #ccc; padding:8px; overflow:auto;"
    >
      <button onclick={() => {selectedCircleId=null; selectedEdgeId=null;}} style="float:right">✕</button>
      {#if selectedCircle()}
        <h3>Circle {selectedCircle()!.id}</h3>
        <button onclick={() => deleteCircle(selectedCircle()!.id)} style="width:100%; margin: 4px 0 12px; padding: 4px; background: #ffdddd; border-color: #ff9999; color: #d8000c; cursor: pointer;">Delete Circle</button>
        <label>Fill <input type="color" value={selectedCircle()!.fill} onkeyup={(e)=>updateCircleField(selectedCircle()!.id,'fill',(e.target as HTMLInputElement).value)} oninput={(e)=>updateCircleField(selectedCircle()!.id,'fill',(e.target as HTMLInputElement).value)} onchange={(e)=>updateCircleField(selectedCircle()!.id,'fill',(e.target as HTMLInputElement).value)}/></label><br />
        <label>Stroke <input type="color" value={selectedCircle()!.stroke} onkeyup={(e)=>updateCircleField(selectedCircle()!.id,'stroke',(e.target as HTMLInputElement).value)} oninput={(e)=>updateCircleField(selectedCircle()!.id,'stroke',(e.target as HTMLInputElement).value)} onchange={(e)=>updateCircleField(selectedCircle()!.id,'stroke',(e.target as HTMLInputElement).value)}/></label><br />
        <label>Diameter <input type="number" min="1" value={selectedCircle()!.diameter} onkeyup={(e)=>updateCircleField(selectedCircle()!.id,'diameter',+(e.target as HTMLInputElement).value)}/></label><br />
        <label>Border <input type="number" min="1" value={selectedCircle()!.strokeWidth} onkeyup={(e)=>updateCircleField(selectedCircle()!.id,'strokeWidth',+(e.target as HTMLInputElement).value)}/></label>
        <hr />
        <h4>Custom Attributes</h4>
        {#each Object.entries(selectedCircle()!.customAttrs ?? {}) as [k,v] (k)}
          <div style="display:flex; gap:4px; margin-bottom:4px;">
            <input type="text" value={k} readonly style="flex:1 1 40%" />
            <input type="text" value={v} style="flex:1 1 50%;" onkeyup={(e)=>updateCircleCustom(selectedCircle()!.id,k,(e.target as HTMLInputElement).value)} />
            <button onclick={()=>removeCircleCustom(selectedCircle()!.id,k)}>🗑</button>
          </div>
        {/each}
        <div style="margin-top:6px;">
          <input placeholder="attr" value={newCircleAttrKey} onkeyup={(e)=>{newCircleAttrKey=(e.target as HTMLInputElement).value; if(e.key === 'Enter') addCircleAttribute()}} style="width:45%" />
          <input placeholder="value" value={newCircleAttrVal} onkeyup={(e)=>{newCircleAttrVal=(e.target as HTMLInputElement).value; if(e.key === 'Enter') addCircleAttribute()}} style="width:45%" />
        </div>
      {:else if selectedEdge()}
        <h3>Edge {selectedEdge()!.id}</h3>
        <button onclick={() => deleteEdge(selectedEdge()!.id)} style="width:100%; margin: 4px 0 12px; padding: 4px; background: #ffdddd; border-color: #ff9999; color: #d8000c; cursor: pointer;">Delete Edge</button>
        <label>Color <input type="color" value={selectedEdge()!.color} onkeyup={(e)=>updateEdgeField(selectedEdge()!.id,'color',(e.target as HTMLInputElement).value)} oninput={(e)=>updateEdgeField(selectedEdge()!.id,'color',(e.target as HTMLInputElement).value)} onchange={(e)=>updateEdgeField(selectedEdge()!.id,'color',(e.target as HTMLInputElement).value)}/></label><br />
        <label>Width <input type="number" min="1" value={selectedEdge()!.width} onkeyup={(e)=>updateEdgeField(selectedEdge()!.id,'width',+(e.target as HTMLInputElement).value)}/></label>
        <hr />
        <h4>Custom Attributes</h4>
        {#each Object.entries(selectedEdge()!.customAttrs ?? {}) as [k,v] (k)}
          <div style="display:flex; gap:4px; margin-bottom:4px;">
            <input type="text" value={k} readonly style="flex:1 1 40%" />
            <input type="text" value={v} style="flex:1 1 50%;" onkeyup={(e)=>updateEdgeCustom(selectedEdge()!.id,k,(e.target as HTMLInputElement).value)} />
            <button onclick={()=>removeEdgeCustom(selectedEdge()!.id,k)}>🗑</button>
          </div>
        {/each}
        <div style="margin-top:6px;">
          <input placeholder="attr" value={newEdgeAttrKey} onkeyup={(e)=>{newEdgeAttrKey=(e.target as HTMLInputElement).value; if(e.key === 'Enter') addEdgeAttribute()}} style="width:45%" />
          <input placeholder="value" value={newEdgeAttrVal} onkeyup={(e)=>{newEdgeAttrVal=(e.target as HTMLInputElement).value; if(e.key === 'Enter') addEdgeAttribute()}} style="width:45%" />
        </div>
      {/if}
    </div>
  {/if}

  {#if false}<style></style>{/if}
</div> 