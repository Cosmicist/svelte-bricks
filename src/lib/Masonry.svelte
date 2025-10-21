<script lang="ts" module>
  type IdKey<T> = {
    [K in keyof T]: T[K] extends string | number ? K : never
  }[keyof T]

  type ObjectItem<K extends PropertyKey> = Record<K, string | number> &
    Record<PropertyKey, unknown>
</script>

<script
  lang="ts"
  generics="
    ItemObject extends ObjectItem<Key>,
    Key extends IdKey<Item>,
    Item extends ItemObject | string | number
  "
>
  import type { Snippet } from 'svelte'
  import { flip } from 'svelte/animate'
  import { fade } from 'svelte/transition'

  type NarrowedItem = Item extends ItemObject ? ItemObject : Item

  type ItemProps<T extends Item> = T extends string | number
    ? { items: NarrowedItem[]; idKey?: never }
    : { items: NarrowedItem[]; idKey: Key }

  type Props = ItemProps<Item> & {
    animate?: boolean
    calcCols?: (masonryWidth: number, minColWidth: number, gap: number) => number
    duration?: number
    gap?: number
    getId?: (item: NarrowedItem) => string | number
    masonryHeight?: number
    masonryWidth?: number
    maxColWidth?: number
    minColWidth?: number
    style?: string
    class?: string
    columnStyle?: string
    columnClass?: string
    children?: Snippet<[{ idx: number; item: NarrowedItem }]>
    div?: HTMLDivElement
    onRedraw?: () => void
  }

  function isObjectItem(item: Item | ItemObject): item is ItemObject {
    return typeof item === 'object' && idKey !== undefined && idKey in item
  }

  let {
    animate = true,
    calcCols = (masonryWidth: number, minColWidth: number, gap: number): number => {
      return Math.min(
        items.length,
        Math.floor((masonryWidth + gap) / (minColWidth + gap)) || 1,
      )
    },
    duration = 200,
    gap = 20,
    getId = (item) => {
      if (typeof item === 'string' || typeof item === 'number') return item
      if (isObjectItem(item) && idKey) return item[idKey]

      throw new Error(
        'svelte-bricks: getId() requires idKey prop to be set when using non-primitive items.',
      )
    },
    items,
    idKey,
    masonryHeight = $bindable(0),
    masonryWidth = $bindable(0),
    maxColWidth = 500,
    minColWidth = 330,
    style = '',
    class: className = '',
    columnStyle = '',
    columnClass = '',
    children,
    div = $bindable(undefined), // TODO add unit test for this prop
    onRedraw,
  }: Props = $props()

  $effect.pre(() => {
    if (maxColWidth < minColWidth) {
      console.warn(
        `svelte-bricks: maxColWidth (${maxColWidth}) < minColWidth (${minColWidth}).`,
      )
    }
  })
  let nCols = $derived(calcCols(masonryWidth, minColWidth, gap))
  let itemsToCols = $derived(
    items.reduce<[NarrowedItem, number][][]>(
      (cols, item, idx) => {
        cols[idx % cols.length].push([item, idx])
        return cols
      },
      Array(nCols)
        .fill(null)
        .map(() => []),
    ),
  )

  // Call onRedraw when nCols changes
  $effect(() => {
    if (nCols) {
      onRedraw?.()
    }
  })
</script>

<!-- deno-fmt-ignore -->
<div
  class="masonry {className}"
  bind:clientWidth={masonryWidth}
  bind:clientHeight={masonryHeight}
  bind:this={div}
  style="gap: {gap}px; {style}"
>
  {#each itemsToCols as col, idx}
    <div
      class="col col-{idx} {columnClass}"
      style="gap: {gap}px; max-width: {maxColWidth}px; {columnStyle}"
    >
      {#if animate}
        {#each col as [item, idx] (getId(item))}
          <div
            in:fade={{ delay: 100, duration }}
            out:fade={{ delay: 0, duration }}
            animate:flip={{ duration }}
          >
            {#if children}{@render children({ idx, item })}{:else}
              <span>{item}</span>
            {/if}
          </div>
        {/each}
      {:else}
        {#each col as [item, idx] (getId(item))}
          {#if children}{@render children({ idx, item })}{:else}
            <span>{item}</span>
          {/if}
        {/each}
      {/if}
    </div>
  {/each}
</div>

<style>
  :where(div.masonry) {
    display: flex;
    justify-content: center;
    overflow-wrap: anywhere;
    box-sizing: border-box;
  }
  :where(div.masonry div.col) {
    display: grid;
    height: max-content;
    width: 100%;
  }
</style>
