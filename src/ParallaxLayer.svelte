<script>
  import { getContext } from 'svelte';
  import { spring } from 'svelte/motion';
  import { contextKey, clamp } from './utils';

  let { rate = 0.5, offset = 0, span = 1, onProgress = undefined, children, ...rest } = $props();

  // get context from Parallax
  const { config, addLayer, removeLayer } = getContext(contextKey);

  // spring store to hold changing scroll coordinate
  const coord = spring(undefined, config);
  // and one to hold intersecting progress
  const progress = spring(0, { ...config, precision: 0.001 });
  // layer height
  let height = $state(0);

  const layer = {
    setPosition: (scrollTop, sectionHeight, disabled) => {
      if (disabled) {
        coord.set(offset * sectionHeight, { hard: true });
        return;
      }
      // amount to scroll before layer is at target position
      const targetScroll = Math.floor(offset) * sectionHeight;
      // distance to target position
      const distance = offset * sectionHeight + targetScroll * rate;
      coord.set(-(scrollTop * rate) + distance);
      progress.set(getProgress(scrollTop, rate, distance, sectionHeight));
    },
    setHeight: (sectionHeight) => {
      height = span * sectionHeight;
    },
  };

  const getProgress = (scrollTop, rate, distance, sectionHeight) => {
    const apparentRate = rate + 1;
    const halfWay = distance / apparentRate;
    const direction = rate >= 0 ? 1 : -1;
    const scrollDistance = (sectionHeight / apparentRate) * direction;
    const start = halfWay - scrollDistance;
    const end = halfWay + scrollDistance * span;
    const progress = (scrollTop - start) / (end - start);
    return clamp(progress, 0, 1);
  };

  $effect(() => {
    // register layer with parent
    addLayer(layer);

    return () => {
      // clean up
      removeLayer(layer);
    };
  });

  // translate layer according to coordinate
  const translate = $derived(`translate3d(0px, ${$coord}px, 0px);`);

  $effect(() => {
    if (onProgress) onProgress($progress ?? 0);
  });
</script>

<div
  {...rest}
  class="parallax-layer {rest.class ? rest.class : ''}"
  style="
    {rest.style ? rest.style : ''};
    height: {height}px;
    -ms-transform: {translate};
    -webkit-transform: {translate};
    transform: {translate};
  "
>
  {@render children?.({ progress: $progress })}
</div>

<style>
  .parallax-layer {
    width: 100%;
    position: absolute;
    box-sizing: border-box;
  }
</style>
