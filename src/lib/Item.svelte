<script lang="ts" generics="T">
    import { onDestroy, onMount, type Snippet, tick} from "svelte"
    interface Props {
        horizontal?: boolean,
        uniqueKey: any,
        type?: string,
        onResize?: (id: any, size: number, type: string) => void,
        children?: Snippet
    }

    let {
        horizontal = false,
        uniqueKey,
        type = "item",
        onResize = () => {},
        children
    }: Props = $props()

    let resizeObserver: ResizeObserver | null
    let itemDiv: HTMLDivElement
    let previousSize: number

    const shapeKey = horizontal ? "offsetWidth" : "offsetHeight"

    onMount(() => {
        resizeObserver = new ResizeObserver(dispatchSizeChange)
        resizeObserver.observe(itemDiv)
        tick().then(dispatchSizeChange);
    })

    onDestroy(() => {
        if (resizeObserver) {
            resizeObserver.disconnect()
            resizeObserver = null
        }
    })

    function dispatchSizeChange() {
        const size = itemDiv ? itemDiv[shapeKey] : 0
        if (size === previousSize) return
        previousSize = size
        onResize(uniqueKey, size, type)
    }
</script>

<div bind:this={itemDiv} class="virtual-scroll-item">
    {@render children?.()}
</div>
