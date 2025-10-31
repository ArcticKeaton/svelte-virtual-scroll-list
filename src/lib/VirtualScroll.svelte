<script lang="ts" generics="T">
    import { isBrowser, Virtual } from "./virtual.js"
    import Item from "./Item.svelte"
    import { onDestroy, onMount, type Snippet, tick, untrack } from "svelte"

    interface Props {
        key: keyof T | typeof keyFn,
        data: T[],
        overflow?: number,
        estimateSize?: number | ((item: T) => number),
        isHorizontal?: boolean,
        start?: number,
        offset?: number,
        pageMode?: boolean,
        topThreshold?: number,
        bottomThreshold?: number,
        onScroll?: (event: Event, range: {start: number, end: number, padFront: number, padBehind: number}) => void,
        onTop?: () => void,
        onBottom?: () => void,
        header?: Snippet,
        footer?: Snippet,
        children?: Snippet<[{ data: T, index: number, localIndex: number }]>
    }

    let {
    /**
     * Unique key for getting data from `data`
     * @type {string}
     */
        key,
    /**
     * Source for list
     * @type {Array<any>}
     */
        data,
    /**
     * Count of items rendered outside of view (for each direction)
     * @type {number}
     */
        overflow = 5,
    /**
     * Estimate size of each item, needs for smooth scrollbar
     * @type {number}
     */
        estimateSize = 50,
    /**
     * Scroll direction
     * @type {boolean}
     */
        isHorizontal = false,
    /**
     * scroll position start index
     */
        start = 0,
    /**
     * scroll position offset
     */
        offset = 0,
        /**
         * Let virtual list using global document to scroll through the list
         * @type {boolean}
         */
        pageMode = false,
        /**
         * The threshold to emit `top` event in px, attention to multiple calls.
         * @type {number}
         */
        topThreshold = 0,
        /**
         * The threshold to emit `bottom` event in px, attention to multiple calls.
         * @type {number}
         */
        bottomThreshold = 0,
        onScroll = () => {},
        onTop = () => {},
        onBottom = () => {},
        header,
        footer,
        children
    }: Props = $props()

    let keyFn: (item: T, index: number) => any = key instanceof Function ? key : (item: T) => item[key]

    let displayItems: T[] = $state([])
    let paddingStyle: string = $state("")
    let directionKey = isHorizontal ? "scrollLeft" as const : "scrollTop" as const
    let virtual = new Virtual({
        slotHeaderSize: 0,
        slotFooterSize: 0,
        overflow: overflow,
        data: data,
    }, onRangeChanged, keyFn, estimateSize)
    let range = $state(virtual.getRange())
    let root: HTMLDivElement
    let shepherd: HTMLDivElement
    let resizeObserver: ResizeObserver

    /**
     * @type {(id: number) => number}
     */
    export function getSize(id: any) {
        return virtual.sizes.get(id)
    }

    /**
     * Count of items
     * @type {() => number}
     */
    export function getSizes() {
        return virtual.sizes.size
    }

    /**
     * @type {() => number}
     */
    export function getOffset() {
        if (pageMode && isBrowser()) {
            return document.documentElement[directionKey] || document.body[directionKey]
        } else {
            return root ? Math.ceil(root[directionKey]) : 0
        }
    }

    /**
     * @type {() => number}
     */
    export function getClientSize() {
        const key = isHorizontal ? "clientWidth" : "clientHeight"
        if (pageMode && isBrowser()) {
            return document.documentElement[key] || document.body[key]
        } else {
            return root ? Math.ceil(root[key]) : 0
        }
    }

    /**
     * @type {() => number}
     */
    export function getScrollSize() {
        const key = isHorizontal ? "scrollWidth" : "scrollHeight"
        if (pageMode && isBrowser()) {
            return document.documentElement[key] || document.body[key]
        } else {
            return root ? Math.ceil(root[key]) : 0
        }
    }

    /**
     * @type {() => void}
     */
    export function updatePageModeFront() {
        if (root && isBrowser()) {
            const rect = root.getBoundingClientRect()
            const {defaultView} = root.ownerDocument
            const offsetFront = isHorizontal ? (rect.left + defaultView!.pageXOffset) : (rect.top + defaultView!.pageYOffset)
            virtual.updateParam("slotHeaderSize", offsetFront)
        }
    }

    /**
     * @type {(offset: number) => void}
     */
    export function scrollToOffset(offset: number) {
        if (!isBrowser()) return
        if (pageMode) {
            document.body[directionKey] = offset
            document.documentElement[directionKey] = offset
        } else if (root) {
            root[directionKey] = offset
        }
    }

    /**
     * @type {(index: number) => void}
     */
    export function scrollToIndex(index: number) {
        if (index >= data.length - 1) {
            scrollToBottom()
        } else {
            const offset = virtual.getOffset(index)
            scrollToOffset(offset)
        }
    }

    /**
     * @type {() => void}
     */
    export function scrollToBottom() {
        if (shepherd) {
            const offset = shepherd[isHorizontal ? "offsetLeft" : "offsetTop"]
            scrollToOffset(offset)

            // check if it's really scrolled to the bottom
            // maybe list doesn't render and calculate to last range,
            // so we need retry in next event loop until it really at bottom
            setTimeout(() => {
                if (getOffset() + getClientSize() + 1 < getScrollSize()) {
                    scrollToBottom()
                }
            }, 3)
        }
    }
    
    onMount(() => {
        resizeObserver = new ResizeObserver(() => onDivScroll());
        onDivScroll()
        tick().then(() => {
            if (start) {
                scrollToIndex(start)
            } else if (offset) {
                scrollToOffset(offset)
            }
        })

        if (pageMode) {
            updatePageModeFront()

            document.addEventListener("scroll", onDivScroll, {
                passive: false,
            })
        }
        resizeObserver.observe(root);
    })

    onDestroy(() => {
        resizeObserver?.disconnect();
        if (pageMode && isBrowser()) {
            document.removeEventListener("scroll", onDivScroll)
        }
    })

    function onItemResized(id: any, size: number, type: string) {
        if (type === "item")
            virtual.saveSize(id, size)
        else if (type === "slot") {
            if (id === "header")
                virtual.updateParam("slotHeaderSize", size)
            else if (id === "footer")
                virtual.updateParam("slotFooterSize", size)

            // virtual.handleSlotSizeChange()
        }
    }

    function onRangeChanged(range_: any) {
        range = range_
        paddingStyle = paddingStyle = isHorizontal ? `0px ${range.padBehind}px 0px ${range.padFront}px` : `${range.padFront}px 0px ${range.padBehind}px`
        displayItems = data.slice(range.start, range.end + 1)
    }

    function onDivScroll(event?: Event) {
        const offset = getOffset()
        const clientSize = getClientSize()
        const scrollSize = getScrollSize()
        virtual.handleScroll(offset, clientSize)
        if (event)
            emitEvent(offset, clientSize, scrollSize, event)
    }

    function emitEvent(offset: number, clientSize: number, scrollSize: number, event: Event) {
        const range = virtual.getRange()
        onScroll(event, range)

        if (offset <= topThreshold) {
            onTop();
        } else if ((scrollSize - offset) - clientSize >= bottomThreshold) {
            onBottom();
        }
    }

    $effect(() => {
        if (offset) {
        }
        untrack(() => scrollToOffset(offset))
    });
    $effect(() => {
        if (start) {
        }
        untrack(() => scrollToIndex(start))
    });

    $effect(() => {
        if (data) {
        }
        untrack(() => handleDataSourcesChange(data))
    });

    async function handleDataSourcesChange(data: T[]) {
        virtual.updateParam("data", data)
    }
</script>

<div bind:this={root} onscroll={onDivScroll} style="overflow-y: auto; height: inherit" class="virtual-scroll-root">
    {#if header}
        <Item onResize={onItemResized} type="slot" uniqueKey="header">
            {@render header?.()}
        </Item>
    {/if}
    <div style="padding: {paddingStyle}" class="virtual-scroll-wrapper">
        {#each displayItems as dataItem, dataIndex (keyFn(dataItem, dataIndex + range.start))}
            <Item
                    onResize={onItemResized}
                    uniqueKey={keyFn(dataItem, dataIndex + range.start)}
                    horizontal={isHorizontal}
                    type="item">
                {@render children?.({ data: dataItem, index: dataIndex + range.start, localIndex: dataIndex })}
            </Item>
        {/each}
    </div>
    {#if footer}
        <Item onResize={onItemResized} type="slot" uniqueKey="footer">
            {@render footer?.()}
        </Item>
    {/if}
    <div bind:this={shepherd} class="shepherd"
         style="width: {isHorizontal ? '0px' : '100%'};height: {isHorizontal ? '100%' : '0px'}"></div>
</div>
