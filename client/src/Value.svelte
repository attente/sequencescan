{#await result}
{:then result}
  {#if typeof result === 'object'}
    {#if isSequenceWallet(result)}
      {#each result as { addressURL, config }}
        <div class="wallet">
          <a href={addressURL} target="_blank">{addressURL}</a>
          <svelte:self value={config} depth={depth + 1} />
        </div>
      {/each}
    {:else if isTransactionHash(result)}
      <div class="transaction">
        <a href={result.transactionURL} target="_blank">{result.transactionURL}</a>
        <svelte:self value={result.value} depth={depth + 1} />
      </div>
    {:else if isFunctionCall(result)}
      <div class="function">
        {result.function}
        <ol start="0">
          {#each result.arguments as [key, value]}
            <li>{key}:&nbsp;<svelte:self {value} depth={depth + 1} /></li>
          {/each}
        </ol>
      </div>
    {:else if isSequenceTransaction(result) && typeof result[0] !== 'undefined'}
      <div class="transaction">
        <svelte:self value={Object.fromEntries(Object.entries(result).filter(([key, _]) => ['delegateCall', 'revertOnError', 'gasLimit', 'target', 'value', 'data'].includes(key)))} depth={depth + 1} />
      </div>
    {:else if sequence.config.isDecodedEOASigner(result) && sequence.config.isDecodedEOASplitSigner(result)}
      <svelte:self value={Object.fromEntries(Object.entries(result).filter(([key, _]) => !['r', 's', 'v', 't'].includes(key)))} depth={depth + 1} />
    {:else if isSequenceSignature(result)}
      <div class="signature">
        <ul>
          {#each Object.entries(result) as [key, value]}
            <li>{key}:&nbsp;<svelte:self {value} depth={depth + 1} /></li>
          {/each}
        </ul>
      </div>
    {:else if isFunctionSignature(result)}
      {result.signature}
      <ul>
        <li>selector: {result.selector}</li>
      </ul>
    {:else if result instanceof Uint8Array}
      {ethers.utils.hexlify(result)}
    {:else if result instanceof Array}
      {#if isEntryArray(result)}
        <ul>
          {#each result as [key, value]}
            <li>{key}:&nbsp;<svelte:self {value} depth={depth + 1} /></li>
          {/each}
        </ul>
      {:else}
        <ol start="0">
          {#each result as element}
            <li><svelte:self value={element} depth={depth + 1} /></li>
          {/each}
        </ol>
      {/if}
    {:else}
      <ul>
        {#each Object.entries(result) as [key, value]}
          <li>{key}:&nbsp;<svelte:self {value} depth={depth + 1} /></li>
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

  div.wallet {
    display: inline-block;
    vertical-align: top;
    padding: 10px;
    border: 1px solid #00ff00;
    border-radius: 10px;
    background-color: #bbffbb;
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

<script context="module" lang="ts">
  import * as ethers from 'ethers'
  import { sequence } from '0xsequence'

  const extraSignatures: Map<string, string[]> = new Map()
  const remoteSignatures: Map<string, Promise<string[]>> = new Map()

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

    if (!remoteSignatures.has(selector)) {
      remoteSignatures.set(selector, (async () => {
        const fetchSignatures = async (url: string): Promise<string[]> => {
          try {
            const response = await fetch(url)
            if (response.ok) {
              return (await response.text()).split(';')
            }
          } catch {
          }

          return []
        }

        return (await Promise.all([
          `https://raw.githubusercontent.com/ethereum-lists/4bytes/master/with_parameter_names/${selector}`,
          `https://raw.githubusercontent.com/ethereum-lists/4bytes/master/signatures/${selector}`
        ].map(fetchSignatures))).flat()
      })())
    }

    signatures.push(...(await remoteSignatures.get(selector)))

    return signatures
  }

  function isTransactionHash(value: any): value is { transactionURL: string, value: string } {
    if (typeof value === 'object') {
      if (typeof value.transactionURL === 'string' && typeof value.value === 'string') {
        return true
      }
    }

    return false
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

  function isSequenceWallet(value: any): value is Array<{ addressURL: string, config: sequence.config.WalletConfig }> {
    if (typeof value === 'object') {
      if (value instanceof Array) {
        if (value.every(value => {
          if (typeof value === 'object') {
            if (typeof value.addressURL === 'string') {
              if (typeof value.config === 'object') {
                return true
              }
            }
          }

          return false
        })) {
          return true
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

  function isFunctionSignature(value: any): value is { signature: string, selector: string } {
    if (typeof value === 'object') {
      if (typeof value.signature === 'string' && typeof value.selector === 'string') {
        return true
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
</script>

<script lang="ts">
  export let value: any
  export let depth: number = 0

  const networks = [
    {
      name: 'polygon',
      chainId: 137,
      nodeURL: 'https://dev-nodes.sequence.app/polygon',
      addressURL: (address: string) => `https://polygonscan.com/address/${address}`,
      transactionURL: (transactionHash: string) => `https://polygonscan.com/tx/${transactionHash}`
    },
    {
      name: 'mainnet',
      chainId: 1,
      nodeURL: 'https://dev-nodes.sequence.app/mainnet',
      addressURL: (address: string) => `https://etherscan.io/address/${address}`,
      transactionURL: (transactionHash: string) => `https://etherscan.io/tx/${transactionHash}`
    },
    {
      name: 'mumbai',
      chainId: 80001,
      nodeURL: 'https://dev-nodes.sequence.app/mumbai',
      addressURL: (address: string) => `https://mumbai.polygonscan.com/address/${address}`,
      transactionURL: (transactionHash: string) => `https://mumbai.polygonscan.com/tx/${transactionHash}`
    },
    {
      name: 'rinkeby',
      chainId: 4,
      nodeURL: 'https://dev-nodes.sequence.app/rinkeby',
      addressURL: (address: string) => `https://rinkeby.etherscan.io/address/${address}`,
      transactionURL: (transactionHash: string) => `https://rinkeby.etherscan.io/tx/${transactionHash}`
    }
  ]

  $: result = (async () => {
    if (depth === 0) {
      if (ethers.utils.isBytesLike(value)) {
        const data = ethers.utils.arrayify(value)
        if (data.length === 4) {
          const signatures = await getSignatures(data)
          if (signatures.length !== 0) {
            return signatures
          }
        } else if (data.length === 20) {
          const address = ethers.utils.getAddress(ethers.utils.hexlify(value))
          const context = sequence.network.sequenceContext
          const configs = []

          for (const network of networks) {
            const provider = new ethers.providers.JsonRpcProvider(network.nodeURL, network.chainId)
            const code = await provider.getCode(address)
            if (code === '0x363d3d373d3d3d363d30545af43d82803e903d91601857fd5bf3') {
              const finder = new sequence.config.SequenceUtilsFinder(provider)
              const { config } = await finder.findCurrentConfig({ address, provider, context })
              if (config !== undefined) {
                configs.push({ addressURL: network.addressURL(address), config })
              }
            }
          }

          if (configs.length !== 0) {
            return configs
          }
        } else if (data.length === 32) {
          const transactionHash = ethers.utils.hexlify(data)

          for (const { nodeURL, transactionURL } of networks) {
            try {
              const response = await fetch(nodeURL, {
                method: 'POST',
                headers: {
                  'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                  jsonrpc: '2.0',
                  method: 'eth_getTransactionByHash',
                  params: [transactionHash],
                  id: 1
                })
              })

              const body = await response.json()
              if (typeof body.result === 'object') {
                return {
                  transactionURL: transactionURL(transactionHash),
                  value: body.result.input
                }
              }
            } catch {
            }
          }
        }
      } else if (typeof value === 'string') {
        try {
          const fragment = new ethers.utils.Interface([value.startsWith('function ') ? value : `function ${value}`]).fragments[0]
          const signature = fragment.format(ethers.utils.FormatTypes.sighash)
          const selector = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(signature)).substring(0, 10)

          return { signature, selector }
        } catch {
        }
      }
    }

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
