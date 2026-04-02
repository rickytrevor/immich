<script lang="ts">
  import type { AssetCursor } from '$lib/components/asset-viewer/asset-viewer.svelte';
  import Portal from '$lib/elements/Portal.svelte';
  import { assetViewerManager } from '$lib/managers/asset-viewer-manager.svelte';
  import { authManager } from '$lib/managers/auth-manager.svelte';
  import { getAssetMediaUrl, handlePromiseError } from '$lib/utils';
  import { navigate } from '$lib/utils/navigation';
  import { AssetMediaSize, getAssetInfo } from '@immich/sdk';
  import { CloseButton, Icon } from '@immich/ui';
  import { mdiImageMultiple } from '@mdi/js';
  import { t } from 'svelte-i18n';

  const COLS = 3;
  const GAP = 4; // gap-1 = 0.25rem = 4px
  const BUFFER_ROWS = 5;

  interface Props {
    assetIds: string[];
    onClose: () => void;
  }

  let { assetIds, onClose }: Props = $props();

  let currentIndex = $state(-1);
  let cursor = $state<AssetCursor | undefined>(undefined);
  let scrollContainer = $state<HTMLElement>();
  let scrollTop = $state(0);
  let containerHeight = $state(0);

  let totalRows = $derived(Math.ceil(assetIds.length / COLS));

  // Cell size is computed from container width
  let cellSize = $derived.by(() => {
    if (!scrollContainer) {
      return 100;
    }
    const width = scrollContainer.clientWidth - 8; // p-1 = 4px each side
    return (width - GAP * (COLS - 1)) / COLS;
  });

  let rowHeight = $derived(cellSize + GAP);
  let totalHeight = $derived(totalRows * rowHeight - GAP);

  let startRow = $derived(Math.max(0, Math.floor(scrollTop / rowHeight) - BUFFER_ROWS));
  let endRow = $derived(Math.min(totalRows, Math.ceil((scrollTop + containerHeight) / rowHeight) + BUFFER_ROWS));

  let visibleItems = $derived.by(() => {
    const items: { id: string; index: number; row: number; col: number }[] = [];
    for (let row = startRow; row < endRow; row++) {
      for (let col = 0; col < COLS; col++) {
        const index = row * COLS + col;
        if (index < assetIds.length) {
          items.push({ id: assetIds[index], index, row, col });
        }
      }
    }
    return items;
  });

  function handleScroll() {
    if (scrollContainer) {
      scrollTop = scrollContainer.scrollTop;
      containerHeight = scrollContainer.clientHeight;
    }
  }

  function initContainer(node: HTMLElement) {
    containerHeight = node.clientHeight;
  }

  async function handleAssetClick(id: string, index: number) {
    currentIndex = index;
    const asset = await getAssetInfo({ ...authManager.params, id });
    assetViewerManager.setAsset(asset);
    await loadCursor(asset, index);
  }

  async function loadCursor(current: NonNullable<typeof assetViewerManager.asset>, index: number) {
    const [previousAsset, nextAsset] = await Promise.all([
      index > 0 ? getAssetInfo({ ...authManager.params, id: assetIds[index - 1] }) : undefined,
      index < assetIds.length - 1 ? getAssetInfo({ ...authManager.params, id: assetIds[index + 1] }) : undefined,
    ]);
    cursor = { current, previousAsset, nextAsset };
  }
</script>

<aside class="h-full w-full overflow-hidden bg-immich-bg dark:bg-immich-dark-bg flex flex-col contain-content">
  <div class="flex items-center justify-between border-b border-gray-200 dark:border-immich-dark-gray pb-1 pe-1">
    <div class="flex items-center gap-2 ps-2">
      <Icon icon={mdiImageMultiple} size="20" />
      <p class="text-sm font-medium text-immich-fg dark:text-immich-dark-fg">
        {$t('assets_count', { values: { count: assetIds.length } })}
      </p>
    </div>
    <CloseButton onclick={onClose} />
  </div>

  {#if !assetViewerManager.isViewing}
    <div
      class="flex-1 overflow-y-auto p-1"
      bind:this={scrollContainer}
      onscroll={handleScroll}
      use:initContainer
    >
      <div style="height: {totalHeight}px; position: relative;">
        {#each visibleItems as item (item.id)}
          <button
            type="button"
            class="absolute overflow-hidden rounded-sm hover:brightness-75 transition-[filter] cursor-pointer"
            style="
              top: {item.row * rowHeight}px;
              left: {item.col * (cellSize + GAP)}px;
              width: {cellSize}px;
              height: {cellSize}px;
            "
            onclick={() => handleAssetClick(item.id, item.index)}
          >
            <img
              src={getAssetMediaUrl({ id: item.id, size: AssetMediaSize.Thumbnail })}
              alt=""
              class="h-full w-full object-cover"
              loading="lazy"
            />
          </button>
        {/each}
      </div>
    </div>
  {/if}
</aside>

<Portal target="body">
  {#if assetViewerManager.isViewing && cursor}
    {#await import('$lib/components/asset-viewer/asset-viewer.svelte') then { default: AssetViewer }}
      <AssetViewer
        {cursor}
        onClose={() => {
          assetViewerManager.showAssetViewer(false);
          cursor = undefined;
          handlePromiseError(navigate({ targetRoute: 'current', assetId: null }));
        }}
        isShared={false}
      />
    {/await}
  {/if}
</Portal>
