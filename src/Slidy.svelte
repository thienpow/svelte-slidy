<script>
    import { wheel } from './actions/wheel.js'
    import { pannable } from './actions/pannable.js'
    import { resizeobserver } from './actions/resizeobserver.js'
    import Spinner from './Spinner.svelte'

    export let slides = []
    export let wrap = {
        id: null,
        width: '100%',
        height: '50vh',
        padding: '0',
    }
    export let slide = {
        gap: 0,
        class: '',
        width: '50vw',
        height: '100%',
        backimg: false,
        imgsrckey: 'src',
    }
    export let controls = {
        dots: true,
        dotsnum: true,
        dotsarrow: true,
        dotspure: false,
        arrows: true,
        keys: true,
        drag: true,
        wheel: true,
    }
    export let duration = 350
    export let axis = 'x'
    export let loader = {
        color: 'red',
        size: 75,
        thickness: 1,
        speed: duration,
    }
    export let index = Math.round(slides.length / 2)

    $: slidyinit && slidyTo(index)

    function slidyTo(i) {
        if (i < 1) {
            index = arr.length
            slidyIndex(i)
        } else if (i > arr.length) {
            index = 1
            slidyIndex(i)
        } else {
            slidyIndex(i)
        }
    }

    // SLIDY-INIT ---------------------------------------------------
    let arr = slides,
        dots = arr,
        nodes = []

    $: element = {
        active: arr[Math.floor(arr.length / 2)],
        first: arr[0],
        firstwidth: arr[0].width + slide.gap,
        firstheight: arr[0].height + slide.gap,
        last: arr[arr.length - 1],
        lastwidth: arr[arr.length - 1].width + slide.gap,
        lastheight: arr[arr.length - 1].height + slide.gap,
        beforewidth: arr.map((a, i) => (i < Math.floor(arr.length / 2) ? a.width + slide.gap : null)).reduce((p, v) => p + v),
        beforeheight: arr.map((a, i) => (i < Math.floor(arr.length / 2) ? a.height + slide.gap : null)).reduce((p, v) => p + v),
        afterwidth: arr.map((a, i) => (i > Math.floor(arr.length / 2) ? a.width + slide.gap : null)).reduce((p, v) => p + v),
        afterheight: arr.map((a, i) => (i > Math.floor(arr.length / 2) ? a.height + slide.gap : null)).reduce((p, v) => p + v),
    }

    $: firstsize = axis === 'y' ? element.firstheight : element.firstwidth
    $: lastsize = axis === 'y' ? element.lastheight : element.lastwidth
    $: diff = axis === 'y' ? (element.beforeheight - element.afterheight) / 2 : (element.beforewidth - element.afterwidth) / 2

    export let slidyinit = false
    function slidyLoad() {
        arr = slides.map((s, i) => {
            return {
                ix: i + 1,
                ...s,
                width: nodes[s.id].clientWidth,
                height: nodes[s.id].clientHeight,
            }
        })
        dots = arr
        slidyinit = true
    }

    // RESIZE-OBSERVER ----------------------------------------------
    function resizeWrap() {
        arr = slides.map((s, i) => {
            return {
                ix: i + 1,
                ...s,
                width: nodes[s.id].clientWidth,
                height: nodes[s.id].clientHeight,
            }
        })
    }

    // CONTROLS & ANIMATION -----------------------------------------
    let sly = 0,
        pos = 0,
        comp = 0,
        translate = 0,
        transition = 0

    $: move = () => {
        if (axis === 'y') {
            return `transform: translate(0, ${translate - diff}px); transition: transform ${transition}ms; top: ${comp}px;`
        } else {
            return `transform: translate(${translate - diff}px, 0); transition: transform ${transition}ms; left: ${comp}px;`
        }
    }

    $: translate = pos
    $: comp = -sly

    function prev() {
        arr = [arr[arr.length - 1], ...arr.slice(0, -1)]
    }
    function next() {
        arr = [...arr.slice(1), arr[0]]
    }

    function slidyPrev() {
        pos += lastsize
        transition = duration
        slidyX()
    }
    function slidyNext() {
        pos -= firstsize
        transition = duration
        slidyX()
    }

    let count = 0
    async function slidyX() {
        if (count === pos) {
            return
        } else if (count < pos) {
            sly = pos
            prev()
        } else if (count > pos) {
            sly = pos
            next()
        }
        count = pos
    }

    // SLIDY ------------------------------------------------------
    async function slidy() {
        if (pos >= lastsize) {
            prev()
            pos = 0
        } else if (pos <= -firstsize) {
            next()
            pos = 0
        }
        index = element.active.ix
    }

    async function slidyStop() {
        transition = duration / 3
        const nulled = () => {
            pos = sly = speed = 0
            setTimeout(() => (index = element.active.ix), transition)
        }
        if (pos >= lastsize / 3 || speed <= -0.005) {
            pos += lastsize - pos
            setTimeout(() => (prev(), nulled(), (transition = 0)), transition)
        } else if (pos <= -firstsize / 3 || speed >= 0.005) {
            pos -= firstsize + pos
            setTimeout(() => (next(), nulled(), (transition = 0)), transition)
        } else {
            nulled()
        }
    }
    function toDefault() {
        if (sly !== 0) sly = pos = count = 0
    }

    // WHEELL -----------------------------------------------------
    let iswheel
    function slidyWheel(e) {
        if (axis === 'y') {
            pos += -e.detail.dy
        } else {
            pos += -e.detail.dx
        }
        transition = 0
        toDefault()
        slidy()
        if (iswheel !== null) {
            clearTimeout(iswheel)
        }
        iswheel = setTimeout(() => {
            clearTimeout(iswheel)
            slidyStop()
        }, duration / 3)
    }

    // DRAG -------------------------------------------------------
    let isdrag = false,
        htx,
        tracker,
        speed
    function dragStart() {
        isdrag = true
        transition = 0
        toDefault()
        if (tracker !== null) {
            clearInterval(tracker)
        }
    }
    function dragSlide(e) {
        if (isdrag) {
            if (axis === 'y') {
                pos += e.detail.dy
            } else {
                pos += e.detail.dx
            }
            slidy()
            tracker = setInterval(() => (htx = pos), duration / 3)
            speed = (htx - pos) / duration / 3
        }
    }
    function dragStop() {
        isdrag = false
        clearInterval(tracker)
        slidyStop()
    }

    // KEYS -------------------------------------------------------
    function slidyKeys(e) {
        if (e.keyCode === 37 || e.keyCode === 38) {
            index--
        } else if (e.keyCode === 39 || e.keyCode === 40) {
            index++
        }
    }

    // INDEX -----------------------------------------------------
    function slidyIndex(id) {
        let i = element.active.ix
        if (id < i && i !== element.first.ix) {
            i = element.active.ix - 1
            while (i >= id) {
                slidyPrev()
                i--
            }
        } else if (id > i && i !== element.last.ix) {
            i = element.active.ix + 1
            while (i <= id) {
                slidyNext()
                i++
            }
        }
    }
    $: stateCheck = setInterval(() => {
        if (document.readyState === 'complete') {
            clearInterval(stateCheck)
            slidyLoad()
        }
    }, 100)
</script>

<section
    role="region"
    tabindex="0"
    aria-label="Slidy"
    id="{wrap.id}"
    class="slidy"
    class:loaded="{slidyinit}"
    class:axisy="{axis === 'y'}"
    class:autowidth="{slide.width === 'auto'}"
    use:resizeobserver
    on:resize="{resizeWrap}"
    use:wheel
    on:wheels="{controls.wheel ? slidyWheel : null}"
    on:keydown="{controls.keys ? slidyKeys : null}"
    style="--wrapw: {wrap.width}; --wraph: {wrap.height}; --wrapp: {wrap.padding || 0}; --slidew: {slide.width}; --slideh: {slide.height}; --slideg: {axis === 'y' ? `${slide.gap / 2}px 0` : `0 ${slide.gap / 2}px`}; --dur: {duration}ms;"
>
    {#if !slidyinit}
        <slot name="loader">
            <Spinner size="{loader.size}" speed="{loader.speed}" color="{loader.color}" thickness="{loader.thickness}" gap="25" />
        </slot>
    {/if}

    <ul class="slidy-ul" use:pannable on:panstart="{controls.drag ? dragStart : null}" on:panmove="{controls.drag ? dragSlide : null}" on:panend="{controls.drag ? dragStop : null}" on:contextmenu="{() => (isdrag = false)}" style="{move()}">
        {#if arr.length > 0}
            {#each arr as item (item.id)}
                <li bind:this="{nodes[item.id]}" class="{slide.class}" class:active="{item.id === element.active.id}" style="{slide.backimg === true ? `background-image: url(${item[slide.imgsrckey]})` : null}">
                    <slot item="{item}">
                        {#if slide.backimg === false}<img alt="{item.id}" src="{slide.imgsrckey ? item[slide.imgsrckey] : item.src}" width="{item.width}" height="{item.height}" />{/if}
                    </slot>
                </li>
            {/each}
        {/if}
    </ul>

    {#if controls.arrows && slidyinit}
        <button class="arrow-left" on:click="{() => index--}">
            <slot name="arrow-left">&#8592;</slot>
        </button>
        <button class="arrow-right" on:click="{() => index++}">
            <slot name="arrow-right">&#8594;</slot>
        </button>
    {/if}

    {#if controls.dots && slidyinit}
        <ul class="slidy-dots" class:pure="{controls.dotspure}">
            {#if controls.dotsarrow}
                <li class="dots-arrow-left" on:click="{() => index--}">
                    <slot name="dots-arrow-left"><button>&#8592;</button></slot>
                </li>
            {/if}
            {#each dots as dot (dot.id)}
                <li class:active="{dot.id === element.active.id}" on:click="{() => (index = dot.ix)}">
                    <slot name="dot" dot="{dot}"><button>{controls.dotsnum && !controls.dotspure ? dot.ix : ''}</button></slot>
                </li>
            {/each}
            {#if controls.dotsarrow}
                <li class="dots-arrow-right" on:click="{() => index++}">
                    <slot name="dots-arrow-right"><button>&#8594;</button></slot>
                </li>
            {/if}
        </ul>
    {/if}
</section>

<style>
    .slidy {
        display: flex;
        align-items: center;
        justify-content: center;
        position: relative;
        overflow: hidden;
        width: var(--wrapw);
        height: var(--wraph);
        outline: 0;
    }
    .slidy ul {
        display: flex;
        align-items: center;
        justify-content: center;
        list-style: none;
        margin: 0;
        border: 0;
        user-select: none;
        -webkit-user-select: none;
    }
    .slidy-ul {
        width: 100%;
        height: 100%;
        padding: var(--wrapp);
        position: absolute;
        touch-action: pan-y;
        will-change: transform;
        -webkit-backface-visibility: hidden;
        backface-visibility: hidden;
        -webkit-perspective: 1000;
        perspective: 1000;
    }
    .slidy.loaded .slidy-ul li {
        opacity: 1;
    }
    .slidy.axisy .slidy-ul {
        flex-direction: column;
    }
    :global(.slidy-ul li img) {
        pointer-events: none;
        object-fit: cover;
        display: block;
        width: 100%;
        height: 100%;
    }
    :global(.slidy.autowidth .slidy-ul li img) {
        width: auto;
    }
    .slidy-ul li {
        flex-shrink: 0;
        max-width: 100%;
        max-height: 100%;
        position: relative;
        opacity: 0;
        transition: color var(--dur), opacity var(--dur);
        width: var(--slidew);
        height: var(--slideh);
        margin: var(--slideg);
        box-sizing: border-box;
        background-repeat: no-repeat;
        background-attachment: scroll;
        background-position: center;
        background-size: cover;
        background-color: rgba(0, 0, 0, 0);
    }
    .slidy button {
        margin: 0;
        border: 0;
        padding: 0;
        border-radius: 0;
        width: 50px;
        height: 50px;
        line-height: 50px;
        color: white;
        background: rgba(0, 0, 0, 0.18);
        cursor: pointer;
        outline: 0;
        overflow: hidden;
        display: flex;
        align-items: center;
        justify-content: center;
    }
    .slidy button:active {
        outline: 0;
    }
    .slidy li.active,
    .slidy li.active button {
        color: red;
        z-index: 1;
    }
    .slidy-dots {
        position: absolute;
        bottom: 0;
        height: 50px;
        padding: 0;
    }
    .slidy.axisy .slidy-dots {
        bottom: auto;
        right: 0;
        width: 50px;
        height: 100%;
        flex-direction: column;
    }
    .slidy-dots li {
        display: flex;
        align-items: center;
        justify-content: center;
        flex-shrink: 0;
    }
    .slidy.axisy .dots-arrow-left,
    .slidy.axisy .dots-arrow-right {
        transform: rotate(90deg);
    }
    .slidy-dots.pure li {
        width: 27px;
        height: 27px;
        background: none;
    }
    .slidy-dots.pure li button {
        border-radius: 50%;
        color: red;
        width: 9px;
        height: 9px;
        transform: scale(1);
        transition: color calc(var(--dur) / 2), transform calc(var(--dur) / 2);
    }
    .slidy-dots.pure li.active button {
        transform: scale(1.8);
        background: red;
    }
    .arrow-left,
    .dots-arrow-left {
        left: 0;
    }
    .arrow-right,
    .dots-arrow-right {
        right: 0;
    }
    .arrow-right,
    .arrow-left {
        position: absolute;
    }
    .slidy-dots.pure .dots-arrow-left button,
    .slidy-dots.pure .dots-arrow-right button {
        background: none;
        width: auto;
        height: auto;
    }
    .dots-arrow-left,
    .dots-arrow-right {
        width: 50px;
        height: 50px;
    }
</style>
