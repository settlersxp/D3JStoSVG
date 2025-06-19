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
    
    onUpdate?: (property: string, value: number | boolean) => void;
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
</div> 