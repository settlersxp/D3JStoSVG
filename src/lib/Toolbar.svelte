<script lang="ts">
  /* ---------------------
   * Parameters – exposed as component props
   * --------------------- */
  // Receive props in runes mode
  const {
    width = 1024,
    height = 760,
    circleDiameter = 60,
    edgeWidth = 5,
    mode = 'draw',
    exportWithBackground = false,
    title = '',
    description = '',
    textPosition = 'bottom',
    iteratorValue = 1,
    
    // Callback props instead of event dispatcher
    onUpdate = () => {},
    onAddRoot = () => {},
    onExport = () => {},
    onToggleMode = () => {},
    onUploadBg = () => {}
  } = $props<{
    width?: number;
    height?: number;
    circleDiameter?: number;
    edgeWidth?: number;
    mode?: 'draw' | 'customize';
    exportWithBackground?: boolean;
    title?: string;
    description?: string;
    textPosition?: 'left' | 'right' | 'top' | 'bottom';
    iteratorValue?: number;
    
    onUpdate?: (property: string, value: number | boolean | string) => void;
    onAddRoot?: () => void;
    onExport?: () => void;
    onToggleMode?: () => void;
    onUploadBg?: () => void;
  }>();
</script>

<style>
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
    flex-wrap: wrap;
  }
</style>

<div class="toolbar">
  <label>W <input type="number" min="100" value={width} oninput={(e) => onUpdate('width', +(e.target as HTMLInputElement).value)} style="width:70px" /></label>
  <label>H <input type="number" min="100" value={height} oninput={(e) => onUpdate('height', +(e.target as HTMLInputElement).value)} style="width:70px" /></label>
  <label>Ø <input type="number" min="1" value={circleDiameter} oninput={(e) => onUpdate('circleDiameter', +(e.target as HTMLInputElement).value)} style="width:50px" title="Circle Diameter" /></label>
  <label>Line <input type="number" min="1" value={edgeWidth} oninput={(e) => onUpdate('edgeWidth', +(e.target as HTMLInputElement).value)} style="width:50px" title="Edge Width" /></label>
  <button onclick={onAddRoot} disabled={mode !== 'draw'}>+ Root</button>
  <button onclick={onExport}>Export SVG</button>
  <label title="Include background in export"><input type="checkbox" checked={exportWithBackground} onchange={(e) => onUpdate('exportWithBackground', (e.target as HTMLInputElement).checked)} /> BG</label>
  <button onclick={onToggleMode}>{mode === 'draw' ? 'Customize' : 'Draw'}</button>
  <button onclick={onUploadBg}>Upload BG</button>
  <label title="Current iterator value for i++ replacement">ID <input type="number" value={iteratorValue} oninput={(e) => onUpdate('iteratorValue', +(e.target as HTMLInputElement).value)} style="width:50px" /></label>
  <div style="width:100%; margin-top:4px; display:flex; gap:4px; align-items:center;">
    <label>Title <input type="text" value={title} oninput={(e) => onUpdate('title', (e.target as HTMLInputElement).value)} style="width:100px" /></label>
    <label>Desc <input type="text" value={description} oninput={(e) => onUpdate('description', (e.target as HTMLInputElement).value)} style="width:100px" /></label>
    <label>Pos 
      <select value={textPosition} onchange={(e) => onUpdate('textPosition', (e.target as HTMLSelectElement).value)} style="width:80px">
        <option value="left">Left</option>
        <option value="right">Right</option>
        <option value="top">Top</option>
        <option value="bottom">Bottom</option>
      </select>
    </label>
  </div>
</div> 