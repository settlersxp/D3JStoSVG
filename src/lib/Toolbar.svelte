<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

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
    exportWithBackground = false
  } = $props<{
    width?: number;
    height?: number;
    circleDiameter?: number;
    edgeWidth?: number;
    mode?: 'draw' | 'customize';
    exportWithBackground?: boolean;
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
  <label>W <input type="number" min="100" value={width} oninput={(e) => dispatch('update', { property: 'width', value: +(e.target as HTMLInputElement).value })} style="width:70px" /></label>
  <label>H <input type="number" min="100" value={height} oninput={(e) => dispatch('update', { property: 'height', value: +(e.target as HTMLInputElement).value })} style="width:70px" /></label>
  <label>Ø <input type="number" min="1" value={circleDiameter} oninput={(e) => dispatch('update', { property: 'circleDiameter', value: +(e.target as HTMLInputElement).value })} style="width:50px" title="Circle Diameter" /></label>
  <label>Line <input type="number" min="1" value={edgeWidth} oninput={(e) => dispatch('update', { property: 'edgeWidth', value: +(e.target as HTMLInputElement).value })} style="width:50px" title="Edge Width" /></label>
  <button onclick={() => dispatch('addRoot')} disabled={mode !== 'draw'}>+ Root</button>
  <button onclick={() => dispatch('export')}>Export SVG</button>
  <label title="Include background in export"><input type="checkbox" checked={exportWithBackground} onchange={(e) => dispatch('update', { property: 'exportWithBackground', value: (e.target as HTMLInputElement).checked })} /> BG</label>
  <button onclick={() => dispatch('toggleMode')}>{mode === 'draw' ? 'Customize' : 'Draw'}</button>
  <button onclick={() => dispatch('uploadBg')}>Upload BG</button>
</div> 