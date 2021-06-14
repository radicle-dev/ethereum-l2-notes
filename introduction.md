# Alternative L1 blockchains

Dropping of Ethereum and usage of a different blockchain creates opportunities to use
radically different technologies and leave behind any legacy limitations.
This may be inconvenient to our users, because they are less likely to know these networks,
which will force them to use new technologies and manage more asset types.
The software integrations may be problematic too as there's less time
and effort put into these blockchains and their ecosystems than into Ethereum.

## NEAR

NEAR is an automatically sharded PoS blockchain.

Every account and every deployed smart contract may be in a separate shard.
Because of that assumption it's possible to transparently move them between shards
and balance the network depending on the number of nodes and resource usage.

The additional complexity is that any interactions between smart contracts, e.g. token transfers,
needs to be split into separate transactions in separate blocks.
When smart contract A calls a function in smart contract B, the execution of A is stopped,
and an internal transaction to B is added to the pool.
When the shard holding B finishes execution of the transaction, it sends a transaction back to A,
which continues the flow.
This has some side effects for interactions between smart contracts:

- The state of the world may change during execution of a call
- There's no global transaction rollback, if something goes wrong, each contract in the call stack
must clean up its state manually
- The state of a contract must be left valid when making a call

These execution differences are reflected in how the ecosystem operates.
For example to use tokens in a smart contract an Ethereum flow is to send `approve` from a wallet
and then `sendFrom` from a contract.
A NEAR way is to call `ft_transfer_call` on a token and pass an encoded message.
The token contract will move the tokens from the caller to the smart contract
and than call that contract's `ft_on_transfer` using the encoded message.
It's very similar to ERC-777's `send` and `tokensReceived`.

I suspect that non-atomicity of complex transactions opens up a whole class of possible bugs,
especially around manual state rollbacks, and attack vectors.

The NEAR smart contracts are written in either Rust or AssemblyScript and then compiled to WASM.
AssemblyScript is a language based on TypeScript and it's explicitly designed to run on WASM.
Usage of Rust makes the developer workflow efficient with its ecosystem of crates and tooling.

### Aurora

Aurora is an EVM environment running on NEAR.

It acts as a regular Ethereum chain and can execute Solidity smart contracts without any changes.
It supports all Ethereum features like multiple calls in a single transaction and rollbacks.
These features are possible, because Aurora acts as a single NEAR contract,
so it can't be split into multiple shards.
Contracts deployed to Aurora can call only each other, they can't use the rest of NEAR ecosystem.
This negates any scalability advantages of NEAR, without automatic sharding
Aurora isn't more congestion-proof than regular L1 Ethereum.
It may be faster and cheaper only as long as it's not widely used.

In my opinion Aurora isn't a good direction for Radicle.

## Solana

Solana is a PoS blockchain.

It uses an innovative proof of history (PoH) mechanism,
which measures time in a process similar to mining.
By accepting proofs of computation the network can measure time
without relying on reaching consensus between the nodes.
This pseudo-mining is single-threaded by design, so it can't use much energy
and doesn't incentivize usage of expensive hardware.

Solana has very high throughput, but doesn't have any scalability mechanisms.
This means that when it becomes more popular, there will be nothing stopping it from congesting, and
as time passes the network will be more and more expensive to participate leading to centralization.

I don't think that Solana would be a good fit for Radicle.

# Plasma

## Polygon
- plasma
- sidechain (nah)
walletconnect :O

# Optimistic rollups

The optimistic rollups allow fitting more transactions into an Ethereum block.
This is done by saving the data from multiple transactions into a single calldata.
The effects of these transactions are kept off-chain, so to query the state one must
use alternative endpoints, but regular one like Infura may still be able to provide events.

The rollups submitted to the Ethereum chain may be invalid or even fraudulent.
The system relies on validators, who will spot these issues
and announce which transactions should be ignored along a proof of an error.
This creates a rolling issue, where fresh transactions can't be trusted,
because they may be invalid, but their error proofs still haven't been announced.
This blurriness of the current state is problematic when treating a blockchain
as a source of truth, e.g. about an organization or its project's state.
I think that usage of optimistic rollups in Radicle ecosystem could work,
but will introduce complexity and may impact the user experience.

## Optimism
- optimistic rollup - has state
- no mainnet yet

## Arbitrum
- optimistic rollup

# ZK-SNARK rollups

The ZK-SNARK rollups are very similar to optimistic rollups, but they solve the uncertainty issue.
All transactions are submitted with a proof that they were applied on a valid latest state
and that they were executed correctly.
The rollup smart contract verifies these proofs and rejects invalid rollups.
The 3rd party observers can trust the latest state immediately
after its the changes are included in an Ethereum block.
This is convenient for the Radicle smart contracts, because the on-chain data can be considered
a source of truth and because the end users can quickly withdraw their funds from L2.

## zkSync

zkSync is an L2 Ethereum rollup based on zk-SNARKs.

Matter Labs (zkSync)
- zk rollup
- Zinc, in a few months Solidity
- For now turing-incomplete
- can be used with eip-712 signed messages


# Starkware
- zk rollup
- cairo lang
- already working?








                | SNARK/STARK   | fraud proofs
data on-chain   | ZK rollup     | optimistic rollup
data off-chain  | Validium      | Plasma
