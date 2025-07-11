<script lang="ts">
  import Toolbar from './Toolbar.svelte';
  import CircleText from './CircleText.svelte';

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
  
  // Text properties for new circles
  let title = $state('');
  let description = $state('');
  let textPosition = $state<'left' | 'right' | 'top' | 'bottom'>('bottom');

  let backgroundImage = $state<string | null>(null);
  let fileInput: HTMLInputElement;
  let exportWithBackground = $state(false);
  let containerRef: HTMLDivElement;
  
  // Max visible dimensions (80% of viewport)
  let maxVisibleWidth = $state(0);
  let maxVisibleHeight = $state(0);

  // Update max dimensions on mount and window resize
  $effect(() => {
    function updateMaxDimensions() {
      maxVisibleWidth = window.innerWidth * 0.8;
      maxVisibleHeight = window.innerHeight * 1;
    }
    
    updateMaxDimensions();
    window.addEventListener('resize', updateMaxDimensions);
    
    return () => {
      window.removeEventListener('resize', updateMaxDimensions);
    };
  });

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
    title?: string;
    description?: string;
    textPosition?: 'left' | 'right' | 'top' | 'bottom';
    customAttrs?: CustomAttrs;
  };

  type Edge = {
    id: number;
    from: number; // circle id
    to: number;   // circle id
    width: number;
    color: string;
    customAttrs?: CustomAttrs;
    // Curve control point (null means straight line)
    controlPoint: { x: number, y: number } | null;
  };

  let circles = $state<Circle[]>([]);
  let edges = $state<Edge[]>([]);

  // helper counters for ids
  let nextCircleId = $state(1);
  let nextEdgeId = $state(1);

  // Currently active drag state
  let dragCircleId: number | null = $state(null);
  let dragEdgeId: number | null = $state(null); // For dragging edge control points
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
      // Process title and description for i++ pattern
      let newTitle = title;
      let newDescription = description;
      
      if (title.includes('i++')) {
        newTitle = title.replace('i++', String(nextCircleId));
      }
      
      if (description.includes('i++')) {
        newDescription = description.replace('i++', String(nextCircleId));
      }
      
      // Get a unique ID, starting with nextCircleId
      const uniqueId = getUniqueId(nextCircleId);
      
      const root: Circle = {
        id: uniqueId,
        x: width / 2,
        y: height / 2,
        diameter: circleDiameter,
        fill: '#4cc9f0',
        stroke: '#1d3557',
        strokeWidth: edgeWidth,
        title: newTitle,
        description: newDescription,
        textPosition: textPosition
      };
      circles = [...circles, root];
      nextCircleId += 1;
    }
  });

  // Make sure nextCircleId and nextEdgeId are always greater than any existing IDs
  $effect(() => {
    // Only ensure nextEdgeId is greater than existing edge IDs
    if (edges.length > 0) {
      const maxEdgeId = Math.max(...edges.map(e => e.id));
      if (maxEdgeId >= nextEdgeId) {
        nextEdgeId = maxEdgeId + 1;
      }
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

  function pointerDownOnEdge(event: PointerEvent, edgeId: number) {
    event.stopPropagation();
    
    if (mode !== 'draw') {
      // In customize mode we don't drag/create, just allow click handler to select
      return;
    }
    
    const isCtrlOrCmd = event.ctrlKey || event.metaKey;
    
    if (isCtrlOrCmd) {
      // Start control point drag
      dragEdgeId = edgeId;
      const svg = (event.currentTarget as SVGGraphicsElement).ownerSVGElement;
      svg?.setPointerCapture(event.pointerId);
      
      // Find the edge
      const edge = edges.find(e => e.id === edgeId);
      if (edge) {
        // Initialize control point if it doesn't exist
        if (!edge.controlPoint) {
          // Calculate midpoint between the two circles
          const fromCircle = circles.find(c => c.id === edge.from);
          const toCircle = circles.find(c => c.id === edge.to);
          
          if (fromCircle && toCircle) {
            const midX = (fromCircle.x + toCircle.x) / 2;
            const midY = (fromCircle.y + toCircle.y) / 2;
            
            // Offset the control point perpendicular to the line
            const dx = toCircle.x - fromCircle.x;
            const dy = toCircle.y - fromCircle.y;
            const len = Math.sqrt(dx * dx + dy * dy);
            const normalX = -dy / len * 50; // 50px perpendicular offset
            const normalY = dx / len * 50;
            
            // Update the edge
            edges = edges.map(e => 
              e.id === edgeId 
                ? { ...e, controlPoint: { x: midX + normalX, y: midY + normalY } }
                : e
            );
          }
        }
      }
    }
  }

  function pointerMove(event: PointerEvent) {
    if (mode !== 'draw') return;
    if (dragCircleId !== null) {
      circles = circles.map((c) =>
        c.id === dragCircleId ? { ...c, x: event.offsetX, y: event.offsetY } : c
      );
    } else if (dragEdgeId !== null) {
      // Update control point position for the edge
      edges = edges.map((e) =>
        e.id === dragEdgeId ? { ...e, controlPoint: { x: event.offsetX, y: event.offsetY } } : e
      );
    } else if (isCreatingEdge && startCircleId !== null) {
      tempX = event.offsetX;
      tempY = event.offsetY;
    }
  }

  // Helper function to ensure a unique ID
  function getUniqueId(preferredId: number): number {
    // If the ID is not used, return it
    if (!circles.some(c => c.id === preferredId)) {
      return preferredId;
    }
    
    // Otherwise find the next available ID
    let id = preferredId;
    while (circles.some(c => c.id === id)) {
      id++;
    }
    return id;
  }

  function pointerUp(event: PointerEvent) {
    if (mode !== 'draw') return;
    const svg = event.currentTarget as SVGSVGElement;
    svg.releasePointerCapture(event.pointerId);

    if (dragCircleId !== null) {
      dragCircleId = null;
      return;
    }
    
    if (dragEdgeId !== null) {
      dragEdgeId = null;
      return;
    }

    if (isCreatingEdge && startCircleId !== null) {
      // Process title and description for i++ pattern
      let newTitle = title;
      let newDescription = description;
      
      if (title.includes('i++')) {
        newTitle = title.replace('i++', String(nextCircleId));
      }
      
      if (description.includes('i++')) {
        newDescription = description.replace('i++', String(nextCircleId));
      }
      
      // Get a unique ID, starting with nextCircleId
      const uniqueId = getUniqueId(nextCircleId);
      
      // Create new circle at pointer position
      const newCircle: Circle = {
        id: uniqueId,
        x: event.offsetX,
        y: event.offsetY,
        diameter: circleDiameter,
        fill: '#ffbe0b',
        stroke: '#8338ec',
        strokeWidth: edgeWidth,
        title: newTitle,
        description: newDescription,
        textPosition: textPosition
      };
      circles = [...circles, newCircle];
      // Create edge
      const newEdge: Edge = {
        id: nextEdgeId,
        from: startCircleId,
        to: newCircle.id,
        width: edgeWidth,
        color: '#000',
        controlPoint: null
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
    // Process title and description for i++ pattern
    let newTitle = title;
    let newDescription = description;
    
    if (title.includes('i++')) {
      newTitle = title.replace('i++', String(nextCircleId));
    }
    
    if (description.includes('i++')) {
      newDescription = description.replace('i++', String(nextCircleId));
    }
    
    // Get a unique ID, starting with nextCircleId
    const uniqueId = getUniqueId(nextCircleId);
    
    const c: Circle = {
      id: uniqueId,
      x: width / 2,
      y: height / 2,
      diameter: circleDiameter,
      fill: '#4cc9f0',
      stroke: '#1d3557',
      strokeWidth: edgeWidth,
      title: newTitle,
      description: newDescription,
      textPosition: textPosition
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
    overflow: auto;
    max-width: 80vw;
    max-height: 80vh;
  }
  .toolbar {
    position: sticky;
    top: 8px;
    left: 8px;
    background: rgba(255, 255, 255, 0.9);
    border: 1px solid #ddd;
    padding: 6px 8px;
    border-radius: 6px;
    display: flex;
    gap: 4px;
    z-index: 10;
    margin-bottom: 8px;
  }
  .sidepanel {
    position: fixed;
    top: 0;
    right: 0;
    width: 260px;
    height: 100%;
    background: #f9f9f9;
    border-left: 1px solid #ccc;
    padding: 8px;
    overflow: auto;
    z-index: 20;
  }
</style>

<Toolbar 
  width={width}
  height={height}
  circleDiameter={circleDiameter}
  edgeWidth={edgeWidth}
  mode={mode}
  exportWithBackground={exportWithBackground}
  title={title}
  description={description}
  textPosition={textPosition}
  iteratorValue={nextCircleId}
  onUpdate={(property, value) => {
    if (property === 'width') width = value as number;
    else if (property === 'height') height = value as number;
    else if (property === 'circleDiameter') circleDiameter = value as number;
    else if (property === 'edgeWidth') edgeWidth = value as number;
    else if (property === 'exportWithBackground') exportWithBackground = value as boolean;
    else if (property === 'title') title = value as string;
    else if (property === 'description') description = value as string;
    else if (property === 'textPosition') textPosition = value as 'left' | 'right' | 'top' | 'bottom';
    else if (property === 'iteratorValue') nextCircleId = value as number;
  }}
  onAddRoot={addRootCircle}
  onExport={exportSVG}
  onToggleMode={toggleMode}
  onUploadBg={triggerFileUpload}
/>
<div class="container" style="width:{Math.min(width, maxVisibleWidth)}px;height:{Math.min(height, maxVisibleHeight)}px" bind:this={containerRef}>
  <input type="file" accept="image/*" style="display:none" bind:this={fileInput} onchange={handleFileSelect} />

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
      {#if e.controlPoint}
        <!-- Curved line with control point -->
        <path
          d="M {circles.find((c) => c.id === e.from)?.x} {circles.find((c) => c.id === e.from)?.y} 
             Q {e.controlPoint.x} {e.controlPoint.y} 
               {circles.find((c) => c.id === e.to)?.x} {circles.find((c) => c.id === e.to)?.y}"
          fill="none"
          stroke={e.color}
          stroke-width={e.width}
          onclick={() => selectEdge(e.id)}
          onpointerdown={(event) => pointerDownOnEdge(event, e.id)}
          {...(e.customAttrs ?? {})}
        />
        <!-- Show control point when in customize mode -->
        {#if mode === 'customize' && selectedEdgeId === e.id}
          <circle
            cx={e.controlPoint.x}
            cy={e.controlPoint.y}
            r="5"
            fill="#ff0"
            stroke="#000"
            stroke-width="1"
          />
        {/if}
      {:else}
        <!-- Straight line -->
        <line
          x1={circles.find((c) => c.id === e.from)?.x}
          y1={circles.find((c) => c.id === e.from)?.y}
          x2={circles.find((c) => c.id === e.to)?.x}
          y2={circles.find((c) => c.id === e.to)?.y}
          stroke={e.color}
          stroke-width={e.width}
          onclick={() => selectEdge(e.id)}
          onpointerdown={(event) => pointerDownOnEdge(event, e.id)}
          {...(e.customAttrs ?? {})}
        />
      {/if}
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
        
        <CircleText 
          x={c.x}
          y={c.y}
          diameter={c.diameter}
          title={c.title}
          description={c.description}
          textPosition={c.textPosition}
        />
      {/key}
    {/each}
  </svg>

  <!-- Side panel -->
  {#if mode==='customize' && (selectedCircle() || selectedEdge())}
    <div
      class="sidepanel"
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
        <h4>Text Properties</h4>
        <label>Title <input type="text" value={selectedCircle()!.title || ''} onkeyup={(e)=>updateCircleField(selectedCircle()!.id,'title',(e.target as HTMLInputElement).value)} style="width:100%"/></label><br />
        <label>Description <input type="text" value={selectedCircle()!.description || ''} onkeyup={(e)=>updateCircleField(selectedCircle()!.id,'description',(e.target as HTMLInputElement).value)} style="width:100%"/></label><br />
        <label>Position 
          <select value={selectedCircle()!.textPosition || 'bottom'} onchange={(e)=>updateCircleField(selectedCircle()!.id,'textPosition',(e.target as HTMLSelectElement).value)}>
            <option value="left">Left</option>
            <option value="right">Right</option>
            <option value="top">Top</option>
            <option value="bottom">Bottom</option>
          </select>
        </label>
        
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