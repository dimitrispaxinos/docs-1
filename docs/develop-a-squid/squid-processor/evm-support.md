---
sidebar_position: 5
description: >-
  Additional support for indexing EVM smart contract data
---

# EVM support

This section describes additional options available for Substrate chains with the Frontier EVM pallet like Moonbeam or Astar. Follow the [EVM squid tutorial](/tutorials/create-an-evm-processing-squid) for a step-by-step tutorial on building an EVM-processing. We recommend using [squid-evm-template](https://github.com/subsquid/squid-evm-template) as a reference.

## Subscribe to EVM events

Use `addEvmLog(contract, options)` to subscribe to the EVM log data (event) emitted by a specific EVM contract: 

```typescript
const processor = new SubstrateBatchProcessor()
  .setDataSource({
    archive: lookupArchive("moonbeam", { release: "FireSquid" }),
  })
  .setTypesBundle("moonbeam")
  .addEvmLog("0xb654611f84a8dc429ba3cb4fda9fad236c505a1a", {
    filter: [erc721.events["Transfer(address,address,uint256)"].topic],
  });
```

The `option` argument supports the same selectors as for `addEvent` and additionally a set of topic filters:

```typescript
{
   range?: DataRange,
   filter?: EvmTopicSet[],
   data?: {} // same as the data selector for `addEvent` 
}
```

Note, that the topic filter follows the [Ether.js filter specification](https://docs.ethers.io/v5/concepts/events/#events--filters). For example, for a filter that accepts the ERC721 topic `Transfer(address,address,uint256)` AND `ApprovalForAll(address,address,bool)` use a double array: 
```ts
processor.addEvmLog('0xb654611f84a8dc429ba3cb4fda9fad236c505a1a', {
  filter: [[
    erc721.events["Transfer(address,address,uint256)"].topic, 
    erc721.events["ApprovalForAll(address,address,bool)"].topic
  ]]
})
```


## Typegen 

`squid-evm-typegen` is used to generate type-safe facade classes to call the contract state and decode the log events. By convention, the generated classes and the ABI file is kept in `src/abi`.
```
npx squid-evm-typegen --abi=src/abi/ERC721.json --output=src/abi/erc721.ts
```

The file generated by `squid-evm-typegen` defines the `events` object with methods for decoding EVM logs into a typed object:

```typescript title="src/abi/erc721.ts"
export const events = {
  // for each topic defined in the ABI
  "Transfer(address,address,uint256)":  {
    topic: abi.getEventTopic("Transfer(address,address,uint256)"),
    decode(data: EvmEvent): Transfer0Event {
      const result = abi.decodeEventLog(
        abi.getEvent("Transfer(address,address,uint256)"),
        data.data || "",
        data.topics
      );
      return  {
        from: result[0],
        to: result[1],
        tokenId: result[2],
      }
    }
  }
  //...
}
```

It can be the used in the handler in the following way:
```typescript
for (const block of ctx.blocks) {
    for (const item of block.items) {
      if (item.name === "EVM.Log") {
        const { from, to, tokenId } = erc721.events["Transfer(address,address,uint256)"].decode(item.event.args)
      }
    }
  }
```

## Access the contract state

The EVM contract state is accessed using the generated `Contract` class that takes the handler context and the contract address as constructor arguments. The state is always accessed at the context block height unless explicitly defined in the constructor.
```typescript title="src/abi/erc721.ts"
export class Contract  {
  constructor(ctx: BlockContext, address: string) { 
    //...
  }
  private async call(name: string, args: any[]) : Promise<ReadonlyArray<any>>  {
    //...
  }
  async balanceOf(account: string, id: ethers.BigNumber): Promise<ethers.BigNumber> {
    const result = await this.call("balanceOf", [owner])
    return result[0]
  }
}
```

It then can be used in a handler in a straightforward way, see [squid-evm-template](https://github.com/subsquid/squid-evm-template).

For more information on EVM Typegen, see this [dedicated page](/develop-a-squid/typegen/squid-evm-typegen).

## Factory contracts

It some cases the set of contracts to be indexed by the squid is not known in advance. For example, a DEX contract typically
creates a new contract for each trading pair added, and each such trading contract is of interest. 

While the set of handler subscriptions is static and defined at the processor creation, one can leverage the wildcard subscriptions and filter the contract of interest in runtime. 

Let's consider how it works in a DEX example, with a contract emitting `'PairCreated(address,address,address,uint256)'` log when a new pair trading contract is created by the main contract. The full code (used by BeamSwap) is available in this [repo](https://github.com/subsquid/beamswap-squid/blob/master/src/processor.ts).

```typescript
// subscribe to events when a new contract is created by the parent 
// factory contract
const processor = new SubstrateBatchProcessor()
    .addEvmLog(FACTORY_ADDRESS, {
        filter: [PAIR_CREATED_TOPIC],
    })
// Subscribe to all contracts emitting the events of interest, and 
// later filter by the addresses deployed by the factory
processor.addEvmLog('*', {
    filter: [
        [
            pair.events['Transfer(address,address,uint256)'].topic,
            pair.events['Sync(uint112,uint112)'].topic,
            pair.events['Swap(address,uint256,uint256,uint256,uint256,address)'].topic,
            pair.events['Mint(address,uint256,uint256)'].topic,
            pair.events['Burn(address,uint256,uint256,address)'].topic,
        ],
    ],
})

async function handleEvmLog(ctx: BatchContext<Store, unknown>, block: SubstrateBlock, event: EvmLogEvent) {
    const contractAddress = event.args.address
    if (contractAddress === FACTORY_ADDRESS && event.args.topics[0] === PAIR_CREATED_TOPIC) {
        // updated the list of contracts to whatch
    } else if (await isPairContract(ctx.store, contractAddress)) {
        // the contract has been created by the factory,
        // index the events
    }
}
```

Since the handlers are lazy (meaning that the database transaction is not opened if `ctx.store` is not hit), the performance overhead is minimal.