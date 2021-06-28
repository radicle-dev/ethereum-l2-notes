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

Plasma is an L2 solution, which keeps all the data on a sidechain,
but periodically checkpoints its state on Ethereum with an L1 transaction.

It's almost completely independent from Ethereum block creation rules,
the sidechain can produce blocks with a different frequency, size and consensus rules,
there's even no requirement that smart contracts need to be EVM-compatible.
The checkpointing system brings additional consensus layer to the sidechain improving security.
It also dictates what's the canonical state of Plasma from Ethereum point of view,
which facilitates communication between L1 and L2.
Finality on Plasma comes with 2 levels of confidence:
1. when a sidechain reaches consensus, which is fast, but doesn't borrow Ethereum safety
2. when a checkpoint on Ethereum is made, verified and Ethereum reaches consensus, which is slower,
but makes laying about a Plasma state at least as hard as lying about an Ethereum state

## Polygon

Polygon is an implementation of Plasma running on a PoS sidechain.
Currently it's one of the most widely used L2 solutions with a big ecosystem and good tooling.
One of Polygon innovations is that honesty when creating a checkpoint is incentivised
by requirement of staking funds by a large group of validators.
This gives enough confidence that the network lets its users withdraw their funds
to Ethereum as soon as a checkpoint is created, which is usually in just a few minutes,
there's no need to wait a few days for the fishermen to find potential frauds.

### Developer perspective

I've successfully deployed attestations and funding pool contracts
on Polygon Mumbai, the official testnet, which is an L2 of Goerli.
Polygon runs regular EVM-compiled smart contracts, so there was no need
to alter their code or even modify the compilation pipeline, everything just works.
The orgs contract is dependant on Gnosis, which is yet to be deployed to Polygon.
Based on my previous experiences I don't expect any issues with migrating orgs.

#### Clients interaction

Polygon clients support standard Ethereum JSON-RPC.
The existing tooling only needs to change the endpoint URL to switch from L1 to L2.
Because of that deployment and interactions with smart contracts are trivial,
all it takes is connecting to a different client URL.

Unfortunately most currently running clients are unusable for upstream event queries,
because they reject any requests touching more than a certain number of blocks.
For the official https://rpc-mumbai.matic.today RPC it's 100K, for Infura it's also 100K,
but their tech support suggests limiting to 10K, for MaticVigil it's just 1K.
The only client allowing unlimited queries is Chainstack, but they tend
to be slow and I've had issues with reliability.
Sometimes events would not being found, I even had problems with transaction history not loading.
I've tried both the public and the billable endpoints.
According to MaticVigil devs the problem lays in Bor, the official Polygon client,
which can't handle queries on wide ranges of blocks.
It's a big problem, because Polygon creates almost 40K blocks per day.
Making a query covering a few months of running a contract could require
dozens or hundreds of requests to a client if limitations stay this tight.
An alternative approach to event queries could be The Graph,
it already officially supports Polygon and we're relying on it anyway.

#### Test tokens

The funding pool works with ERC-20 tokens, the goal for it is to run on DAI.
Polygon mainnet is officially supported by DAI.
On Mumbai there are many deployments of DAI, but I couldn't find any
that would have both a public faucet and Polygon bridge support.
The latter is especially important to test the whole user experience
starting from having funds on Ethereum and transferring them to Polygon.

I've decided to use the official Mumbai test ERC-20 token.
It has [an official faucet](https://faucet.matic.network) and supports the bridge.
The faucet has temporary downtimes, but it's usable, it seems to be periodically refilled.

### User perspective

The user needs to perform a few steps to use smart contracts on Polygon.
We may be able to hide from the users some of them like bridging and buying MATIC to pay for gas.
The latter problem can be worked around elegantly by integrating our smart contracts
with Gas Station Network, the official Polygon gasless transactions system.
We could spend users' DAI to run the transactions, this way the whole Radicle
ecosystem could be operated with a single token type simplifying the UI.
Another approach is Biconomy, where the user transactions are submitted
by a relayer, who gets paid by us to cover gas costs.
This isn't necessarily a financial suicide, because Polygon transactions are very cheap.

#### Usage steps

I tried the manual route with no modifications of smart contracts.

First, I needed to add Polygon network to my browser extension MetaMask to perform
any transactions from outside of upstream.
I also needed to do the same with my mobile MetaMask as I'm using it for upstream.
Any Polygon client address will be fine, the wallets don't require any demanding queries.

Then, I obtained some MATIC from the faucet.
The real users will need to buy it from an exchange and either
receive it directly on Polygon or push it through the bridge.

Next, I pushed my tokens through the bridge, which I wanted to send over a token stream.
For test purposes it was the generic test ERC-20 obtained from the faucet,
but the real users will be using DAI.
The bridge is accessible from [Polygon Wallet](https://wallet.matic.today/) and it's easy to use.
It should take around a minute to move tokens from L1 to L2.
One of my bridge transfers got stuck for almost a day before it showed up on Polygon,
which for testing was confusing, but for a real user would be dreadful.

Finally, I used upstream to stream some tokens, which functions exactly the same as on Ethereum.
WalletConnect just works, it wasn't less reliable than usually.
MetaMask for Android, which was signing transactions on the other end, ran fine as well.

Using the token stream receiver account I collected the sent tokens
and using the bridge transferred them to L1.
The whole process took just a few minutes and went smoothly.

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

Arbitrum is very similar to Optimism in principle.
There are some technical differences between them

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


mint.zksync.dev

account activation is required

# Starkware
- zk rollup
- cairo lang
- already working?








                | SNARK/STARK   | fraud proofs
data on-chain   | ZK rollup     | optimistic rollup
data off-chain  | Validium      | Plasma
