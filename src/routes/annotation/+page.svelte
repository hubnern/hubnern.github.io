<script>
    import {onMount} from "svelte";
    /** @type {{directory:string,action:string,left:number,right:number}[]} */
    import actions from "$lib/exp-breakages.json"
    // import {current} from "$lib/store.js";

    const TIMESTAMP_AND_CONTENT_REGEXP = /\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}.\d{7}Z /g
    const ANSI_COLOR_REGEXP = /\e?\[(\d\d?)?(;\d\d?)*m/g

    /**
     * @param {string} content
     * @return {string[]}
     */
    function parseFileContent(content) {
        return content.replace(/\r\n/g, '\n').replace(/\r/g, "\n").split(/\n/).map((line, i) => (i + 1) + " " + line.replace(TIMESTAMP_AND_CONTENT_REGEXP, '').replace(ANSI_COLOR_REGEXP, ''))
    }


    /** @type string[] */
    let leftContent = []
    /** @type string[] */
    let rightContent = []
    let actionType = "LOADING"

    let current = 0;
    let result = new Map()

    onMount(() => {
        updateLog()
    })

    function updateLog() {
        let action = actions[current];
        let folder = action.directory.split('/')[3];
        actionType = "LOADING"
        console.log(`fetching ${folder} for action ${JSON.stringify(action)}`)
        let promiseLeft = fetch(`/breakages/${folder}/pass.log`)
            .then(response => response.text())
            .then(content => leftContent = parseFileContent(content))
        let promiseRight = fetch(`/breakages/${folder}/fail.log`)
            .then(response => response.text())
            .then(content => rightContent = parseFileContent(content))
        Promise.all([promiseLeft, promiseRight])
            .then(() => {
                actionType = action.action
                document.getElementById("line-" + action.left)?.scrollIntoView({block: "center", behavior: "instant"})
                document.getElementById("header")?.scrollIntoView({block: "start", behavior: "instant"})
            })
    }

    function clickOK() {
        result.set(current, true)
        current++
        updateLog()
    }

    function clickNOTOK() {
        result.set(current, false)
        current++
        updateLog()
    }

    function goBack() {
        current--
        updateLog()
    }


    /** @type HTMLTableElement*/
    let table1
    /** @type HTMLTableElement*/
    let table2

    function syncTable2() {
        // let ignore = ignoreScroll
        // ignoreScroll = false
        // if (ignore) return
        // ignoreScroll = true
        table2.scrollTop = table1.scrollTop
        table2.scrollLeft = table1.scrollLeft
    }

    function syncTable1() {
        // let ignore = ignoreScroll
        // ignoreScroll = false
        // if (ignore) return
        // ignoreScroll = true
        table1.scrollTop = table2.scrollTop
        table1.scrollLeft = table2.scrollLeft
    }

    let inSave = false


</script>
<main>
    {#if inSave}
        <div id="save">
            <textarea>{JSON.stringify(Object.fromEntries(result))}</textarea>
            <button on:click={() => inSave = false}>close</button>
        </div>
    {/if}
    <div id="header">
        <button on:click={goBack}>RETURN</button>
        <button on:click={clickNOTOK}>NOT OK</button>
        <div>
            <p>{current+1} / {actions.length}</p>
            <h2>{actionType}</h2>
            <p>{actions[current].directory.split("/")[3]}</p>
            <p>{actions[current].left} - {actions[current].right}</p>
        </div>
        <button on:click={clickOK}>OK</button>
        <button on:click={() => inSave = true}>SAVE</button>
    </div>
    <div class="flex">
        <table bind:this={table1} on:scroll={syncTable2}>
            {#each leftContent as line,i}
                <tr>
                    <td class:selected={actions[current].left === i + 1} id="line-{i}">{line}</td>
                </tr>
            {/each}
        </table>
        <table bind:this={table2} on:scroll={syncTable1}>
            {#each rightContent as line,i}
                <tr>
                    <td class:selected={actions[current].right === i + 1}>{line}</td>
                </tr>
            {/each}
        </table>
    </div>
</main>

<style>
    .flex {
        display: flex;
        height: 100vh;
    }

    #header {
        border: 1px solid black;
        box-sizing: border-box;
        display: flex;
        text-align: center;
        justify-content: center;
        /*align-items: center;*/
    }

    #save {
        position: absolute;
        width: 100%;
        height: 100%;
        top: 0;
        left: 0;
        text-align: center;
        background-color: white;
    }

    #save textarea {
        margin: 1em;
        width: 95%;
        height: 90%;
    }

    .selected {
        background-color: cornflowerblue;
    }

    table {
        width: 100%;
        overflow-x: scroll;
        border: 1px solid black;
        margin: 0.2em;
    }

    table, td {
        display: block;
    }

    td {
        width: 300%;
    }

    button {
        width: 5em;
        margin: 1em;
    }
    h2, p, td, tr {
        font-family: sans-serif;
    }
</style>