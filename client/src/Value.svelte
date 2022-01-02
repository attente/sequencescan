{#await result}
{:then result}
  {#if typeof result === 'object'}
    {#if isFunctionCall(result)}
      <div class="function">
        {result.function}
        <ol start="0">
          {#each result.arguments as [key, value]}
            <li>{key}:&nbsp;<svelte:self {value} /></li>
          {/each}
        </ol>
      </div>
    {:else if isSequenceTransaction(result) && typeof result[0] !== 'undefined'}
      <div class="transaction">
        <svelte:self value={Object.fromEntries(Object.entries(result).filter(([key, _]) => ['delegateCall', 'revertOnError', 'gasLimit', 'target', 'value', 'data'].includes(key)))} />
      </div>
    {:else if sequence.config.isDecodedEOASigner(result) && sequence.config.isDecodedEOASplitSigner(result)}
      <svelte:self value={Object.fromEntries(Object.entries(result).filter(([key, _]) => !['r', 's', 'v', 't'].includes(key)))} />
    {:else if isSequenceSignature(result)}
      <div class="signature">
        <ul>
          {#each Object.entries(result) as [key, value]}
            <li>{key}:&nbsp;<svelte:self {value} /></li>
          {/each}
        </ul>
      </div>
    {:else if result instanceof Uint8Array}
      {ethers.utils.hexlify(result)}
    {:else if result instanceof Array}
      {#if isEntryArray(result)}
        <ul>
          {#each result as [key, value]}
            <li>{key}:&nbsp;<svelte:self {value} /></li>
          {/each}
        </ul>
      {:else}
        <ol start="0">
          {#each result as element}
            <li><svelte:self value={element} /></li>
          {/each}
        </ol>
      {/if}
    {:else}
      <ul>
        {#each Object.entries(result) as [key, value]}
          <li>{key}:&nbsp;<svelte:self {value} /></li>
        {/each}
      </ul>
    {/if}
  {:else}
    {result}
  {/if}
{:catch}
  {value}
{/await}

<style>
  div.function {
    display: inline-block;
    vertical-align: top;
    padding: 10px;
    border: 1px solid #0000ff;
    border-radius: 10px;
    background-color: #bbbbff;
    margin: 10px 0px;
  }

  div.transaction {
    display: inline-block;
    vertical-align: top;
    padding: 10px;
    border: 1px solid #00ff00;
    border-radius: 10px;
    background-color: #bbffbb;
    margin: 10px 0px;
  }

  div.signature {
    display: inline-block;
    vertical-align: top;
    padding: 10px;
    border: 1px solid #ff0000;
    border-radius: 10px;
    background-color: #ffbbbb;
    margin: 10px 0px;
  }

  ol, ul {
    margin: 0px 10px;
    padding-left: 10px;
  }

  li {
    white-space: nowrap;
  }
</style>

<script lang="ts">
  import * as ethers from 'ethers'
  import { sequence } from '0xsequence'

  export let value: any

  const extraSignatures: Map<string, string[]> = new Map()

  for (const contract of Object.values(sequence.abi.walletContracts)) {
    const abi = new ethers.utils.Interface(contract.abi)

    for (const fragment of Object.values(abi.functions)) {
      const selector = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(fragment.format())).substring(2, 10)

      let signatures = extraSignatures.get(selector)
      if (!signatures) {
        signatures = []
        extraSignatures.set(selector, signatures)
      }

      let signature = fragment.format(ethers.utils.FormatTypes.full)
      if (signature.startsWith('function ')) {
        signature = signature.substring('function '.length)
      }

      signatures.push(signature)
    }
  }

  async function getSignatures(selector: ethers.utils.BytesLike): Promise<string[]> {
    selector = ethers.utils.hexlify(selector).substring(2)

    const signatures = extraSignatures.has(selector) ? [...extraSignatures.get(selector)] : []

    const responses = await Promise.allSettled([
      fetch(`https://raw.githubusercontent.com/ethereum-lists/4bytes/master/with_parameter_names/${selector}`),
      fetch(`https://raw.githubusercontent.com/ethereum-lists/4bytes/master/signatures/${selector}`)
    ])

    for (const response of responses) {
      if (response.status === 'fulfilled' && response.value.ok) {
        signatures.push(...(await response.value.text()).split(';'))
      }
    }

    return signatures
  }

  function isFunctionCall(value: any): value is { function: string, arguments: Array<[string, any]> } {
    if (typeof value === 'object') {
      if (typeof value.function === 'string') {
        if (typeof value.arguments === 'object') {
          if (isEntryArray(value.arguments)) {
            return true
          }
        }
      }
    }

    return false
  }

  function isSequenceTransaction(value: any): value is sequence.transactions.Transaction {
    if (typeof value === 'object') {
      if (typeof value.delegateCall === 'boolean') {
        if (typeof value.revertOnError === 'boolean') {
          if (ethers.BigNumber.isBigNumber(value.gasLimit)) {
            if (ethers.utils.isAddress(value.target)) {
              if (ethers.BigNumber.isBigNumber(value.value)) {
                if (ethers.utils.isBytesLike(value.data)) {
                  return true
                }
              }
            }
          }
        }
      }
    }

    return false
  }

  function isSequenceSignature(value: any): value is sequence.config.DecodedSignature {
    if (typeof value === 'object') {
      if (typeof value.threshold === 'number') {
        if (typeof value.signers === 'object') {
          if (value.signers instanceof Array) {
            if (value.signers.every((signer: any) => sequence.config.isDecodedAddress(signer) || sequence.config.isDecodedEOASigner(signer) || sequence.config.isDecodedEOASplitSigner(signer) || sequence.config.isDecodedFullSigner(signer))) {
              return true
            }
          }
        }
      }
    }

    return false
  }

  function isEntryArray(value: any): value is Array<[string, any]> {
    if (typeof value === 'object') {
      if (value instanceof Array) {
        if (value.every(element => typeof element === 'object' && element instanceof Array && element.length === 2 && typeof element[0] === 'string')) {
          return true
        }
      }
    }

    return false
  }

  $: result = (async () => {
    if (ethers.BigNumber.isBigNumber(value)) {
      return value.toString()
    } else if (ethers.utils.isBytesLike(value)) {
      const data = ethers.utils.arrayify(value)

      try {
        return sequence.config.decodeSignature(ethers.utils.hexlify(data))
      } catch {
        try {
          return sequence.config.decodeSignature(ethers.utils.hexlify(data.subarray(0, data.length - 1)))
        } catch {
        }
      }

      if (data.length >= 4) {
        const selector = data.subarray(0, 4)
        const signatures = await getSignatures(selector)
        const decoder = new ethers.utils.Interface(signatures.map(signature => `function ${signature}`))

        for (const signature of signatures) {
          try {
            const dataArguments = decoder.decodeFunctionData(signature, data)
            const signatureFunction = decoder.getFunction(signature)

            return {
              function: signature,
              arguments: dataArguments.map((argument, i) => [signatureFunction.inputs[i].format(ethers.utils.FormatTypes.full), argument])
            }
          } catch {
          }
        }
      }

      return value
    } else {
      return value
    }
  })()
</script>
