<script lang="ts">
  // Props for the component
  const {
    x = 0,
    y = 0,
    diameter = 60,
    title = '',
    description = '',
    textPosition = 'bottom'
  } = $props<{
    x: number;
    y: number;
    diameter: number;
    title?: string;
    description?: string;
    textPosition?: 'left' | 'right' | 'top' | 'bottom';
  }>();
</script>

{#if title || description}
  {#if textPosition === 'left'}
    <!-- Title with white border - left position -->
    {#if title}
      <text
        x={x - diameter / 2 - 10}
        y={y - 5}
        text-anchor="end"
        dominant-baseline="middle"
        font-weight="bold"
        stroke="white"
        stroke-width="3"
        paint-order="stroke"
      >{title}</text>
    {/if}
    
    <!-- Description with white border - left position -->
    {#if description}
      <text
        x={x - diameter / 2 - 10}
        y={y + 15}
        text-anchor="end"
        dominant-baseline="middle"
        stroke="white"
        stroke-width="3"
        paint-order="stroke"
      >{description}</text>
    {/if}
  {:else if textPosition === 'right'}
    <!-- Title with white border - right position -->
    {#if title}
      <text
        x={x + diameter / 2 + 10}
        y={y - 5}
        text-anchor="start"
        dominant-baseline="middle"
        font-weight="bold"
        stroke="white"
        stroke-width="3"
        paint-order="stroke"
      >{title}</text>
    {/if}
    
    <!-- Description with white border - right position -->
    {#if description}
      <text
        x={x + diameter / 2 + 10}
        y={y + 15}
        text-anchor="start"
        dominant-baseline="middle"
        stroke="white"
        stroke-width="3"
        paint-order="stroke"
      >{description}</text>
    {/if}
  {:else if textPosition === 'top'}
    <!-- Title with white border - top position -->
    {#if title}
      <text
        x={x}
        y={y - diameter / 2 - 25}
        text-anchor="middle"
        dominant-baseline="middle"
        font-weight="bold"
        stroke="white"
        stroke-width="3"
        paint-order="stroke"
      >{title}</text>
    {/if}
    
    <!-- Description with white border - top position -->
    {#if description}
      <text
        x={x}
        y={y - diameter / 2 - 10}
        text-anchor="middle"
        dominant-baseline="middle"
        stroke="white"
        stroke-width="3"
        paint-order="stroke"
      >{description}</text>
    {/if}
  {:else}
    <!-- Title with white border - bottom position (default) -->
    {#if title}
      <text
        x={x}
        y={y + diameter / 2 + 15}
        text-anchor="middle"
        dominant-baseline="middle"
        font-weight="bold"
        stroke="white"
        stroke-width="3"
        paint-order="stroke"
      >{title}</text>
    {/if}
    
    <!-- Description with white border - bottom position -->
    {#if description}
      <text
        x={x}
        y={y + diameter / 2 + 30}
        text-anchor="middle"
        dominant-baseline="middle"
        stroke="white"
        stroke-width="3"
        paint-order="stroke"
      >{description}</text>
    {/if}
  {/if}
{/if} 