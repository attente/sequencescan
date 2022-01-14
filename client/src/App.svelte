<div id="app">
  <div id="query">
    <input type="text" bind:value={query}>
    <input type="button" value="reset" on:click={() => { query = undefined; value = undefined }}>
  </div>

  {#if value === undefined}
    <div id="splash">
      enter:
      <ul>
        <li>transaction hash (<a on:click={() => { query = 'a2c9e9d366f4540ee0af3d8080576f9eab2da6ca9a45cbe88ef1a578f735181a' }}>example</a>)</li>
        <li>transaction data</li>
        <li>encoded sequence signature</li>
        <li>function signature (<a on:click={() => { query = 'safeBatchTransferFrom(address _from, address _to, uint256[] calldata _ids, uint256[] calldata _values, bytes calldata _data)' }}>example</a>)</li>
        <li>function selector (<a on:click={() => { query = '2eb2c2d6' }}>example</a>)</li>
      </ul>
    </div>
  {:else}
    <Value {value} />
  {/if}
</div>

<style>
  div#app {
    display: flex;
    flex-flow: column;
    height: 100%;
  }

  div#query {
    display: flex;
    flex-flow: row;
    flex-grow: 0;
  }

  input[type=text] {
    flex-grow: 1;
    border-radius: 10px 0px 0px 10px;
  }

  input[type=button] {
    flex-grow: 0;
    border-radius: 0px 10px 10px 0px;
  }

  div#splash {
    flex-grow: 1;
  }

  ul {
    margin-left: 10px;
    padding-left: 10px;
  }
</style>

<script lang="ts">
  import { ethers } from 'ethers'
  import Value from './Value.svelte'

  let query: string | undefined
  let value: string | undefined
  let timeoutID: NodeJS.Timeout | undefined

  $: {
    if (timeoutID !== undefined) {
      clearTimeout(timeoutID)
    }

    if (query === undefined || query === '') {
      value = undefined
    } else {
      timeoutID = setTimeout(handleQuery, 1000)
    }
  }

  function handleQuery() {
    value = ethers.utils.isHexString(`0x${query}`) ? `0x${query}` : query
    timeoutID = undefined
  }
</script>
