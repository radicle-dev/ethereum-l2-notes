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
Experimental Mumbai support for radicle-contracts is added on branch
[test-polygon](https://github.com/radicle-dev/radicle-contracts/tree/test-polygon)
and for upstream on branch
[igor/test-polygon](https://github.com/radicle-dev/radicle-upstream/tree/igor/test-polygon).
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
For example Goerli DAI from `https://app.compound.finance/` doesn't have a bridge.

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
The system relies on fishermen, who will spot these issues
and announce which transactions should be ignored along a proof of an error.
This creates a rolling issue, where fresh transactions can't be trusted,
because they may be invalid, but their error proofs still haven't been announced.
This blurriness of the current state is problematic when treating a blockchain
as a source of truth, e.g. about an organization or its project's state.
I think that usage of optimistic rollups in Radicle ecosystem could work,
but will introduce complexity and may impact the user experience.

## Optimism

Optimism is an implementation of an optimistic rollup.
It runs smart contracts on OVM, which is similar to EVM, but it lacks
some opcodes, which are substituted with precomiled smart contracts.
This is needed to make every L2 transaction runnable inside a special L1
smart contract when a proof of fraud is published, which makes them immediately

Optimism hasn't been officially deployed for Ethereum Mainnet, it's still in the testing phase.

### Developer perspective

I've successfully deployed attestations and funding pool contracts on Kovan Optimism.
Deployment works exactly the same as on L1, even the scripts can be reused without modification.
Experimental Optimism support for radicle-contracts is added on branch
[test-optimism](https://github.com/radicle-dev/radicle-contracts/tree/test-optimism)
and for upstream on branch
[igor/test-optimism](https://github.com/radicle-dev/radicle-upstream/tree/igor/test-optimism).

Optimism comes with a modified version of the Solidity compiler, which produces the OVM bytecode.
The compiler API is unchanged and it's compatible with existing tooling like Hardhat and Truffle.
All it takes to make the switch is addition of a dependency
and modification of the configuration file.

The tooling for running tests is more problematic than with L1 Ethereum.
Because the contracts are compiled to a slightly different bytecode,
regular test clients like Ganache can't be used.
Optimism provides a docker-compose set of images, which serves that purpose.
It's as easy to use as possible, simple `docker-compose build && docker-compose run` is enough.
Unfortunately it's very large, it took around 2 hours
to build and run for the first time on my machine.
The build time and requirement of docker-compose is going to make it challenging to use it in a CI.

When a local Optimism client is running in the background, many of the Hardhat tests just work.
Unfortunately most of funding pool tests and many of radicle token tests don't, because they rely
on test RPC commands like `evm_mine`, `evm_setNextBlockTimestamp` and `evm_snapshot`, which are
[not present yet](https://github.com/ethereum-optimism/optimism/issues/1194) on the client.
All registrar tests keep failing too due to a
[known issue](https://community.optimism.io/docs/developers/integration.html#constructor-arguments-might-be-unsafe)
with the compiler.

Optimism has a single official public client and it just works providing all necessary features.
If there's need for higher loads, Infura has recently introduced support for Optimism.
Due to the way the optimistic rollups work, the clients provide state,
which may be invalidated in the future by a fraud proof.
This is acceptable for operations, which have only on-chain effects, like funding pools,
but can be disastrous when an off-chain system is affected by the chain state, like attestations.
The only solution would be to wait a few days before respecting the state,
waiting for the fishermen to catch any frauds, but the UX of such system would be terrible.

Optimism has great support for Kovan Dai, which I've used as the ERC-20 token in the funding pool.
It provides a testnet bridge, which works with deployment of Dai used in the
`https://app.compound.finance/` faucet and in Kovan Uniswap.

### User perspective

Optimism runs on gas paid in Ether, so there's no need to buy any special tokens.
Unfortunately both Eth and Dai needs to be transferred over a bridge to use it.
It should be possible to handle this step in the app, so less savvy users don't get confused.

Support for Optimism in mobile wallets isn't great.
The transactions need to be handled slightly differently and sometimes it causes problems.
The main difference is that all gas estimations have two values encoded in them, one for transaction
cost on L1 and one on L2, which makes the final value huge, in range of tens of million of gas.
The other detail is that gas price should be always set to 0.015 GWei.
This is problematic, because it requires wallets to implement special handling
of Optimism transactions, which wouldn't make sense on L1.

- WallEth works fine. It needs gas price to be manually set for each transaction to 0.015 GWei.
- MetaMask for Android gets confused by both of these differences, it refuses to sign a transaction
seemingly using more gas than the whole block and can't handle fractions in gas price.
- Alpha wallet always uses WalletConnect with Ethereum Mainnet and can't switch to Optimism.
- ImToken silently fails, the transactions just don't show up on the network.

#### Usage steps

First, I needed to add Optimism network to my browser extension MetaMask to perform
any transactions from outside of upstream.
I also needed to do the same with my mobile WallEth as I'm using it for upstream.

Next, I pushed my Ether and Dai through [the bridge](https://gateway.optimism.io/).
It takes around a minute to move tokens from L1 to L2.
I used upstream to stream some tokens, which functions exactly the same as on Ethereum.

Finally, using the token stream receiver account I collected the sent tokens.
Using the bridge I transferred them from L2 to L1, they will appear on L1 in around a week.

## Arbitrum

Arbitrum is a direct competitor of Optimism.
There are some technical differences between them, like Arbitrum running vanilla EVM
instead of simplified OVM and a longer fraud proving process, but nothing substantial.
It's a less popular solution with fewer projects
and a lower chance of having our users already onboarded.
If we decide to use an optimistic rollup, I think that Optimism should be the way to go.

# ZK rollups

The ZK rollups are very similar to optimistic rollups, but they solve the uncertainty issue.
All transactions are submitted with a proof that they were applied on a valid latest state
and that they were executed correctly.
The rollup smart contract verifies these proofs and rejects invalid rollups.
The 3rd party observers can trust the latest state immediately
after its the changes are included in an Ethereum block.
This is convenient for the Radicle smart contracts, because the on-chain data can be considered
a source of truth and because the end users can quickly withdraw their funds from L2.

## StarNet

StarkNet is a programmable ZK rollup L2 solution developed by StarkWare.
It isn't released on Mainnet yet and WIP versions keep getting released on Ropsten.
The company developed another ZK rollup system called StarkEx, which is already running on Mainnet.
It provides some services like transfers and exchanges which are running on ZK rollups,
but they have fixed functionality, you can't deploy custom logic on StarkEx.

It's unclear when StarkNet will be released on Mainnet.
Right now it can run custom smart contracts, but every ZK rollup posted on Ropsten
can hold only changes coming from a single contract.
Because of that there are no cross-contract calls yet,
e.g. you can't make an ERC-20 transfer from within a funding pool.
This situation is supposed to change in a near future and the whole StarkNet state changes
will be kept in single rollups.
There are many more missing pieces, like designing usage pricing scheme, opening source of
all components including the proof generator and decentralization of the network.

### Cairo

Cairo is the only currently supported smart contracts language on StarkNet.
Its VM (execution rules and memory model) is somewhat similar to that of a regular CPU,
but it has some exotic limitations.

The most important one is that the memory is a stack, which can only be appended and never modified.
For example when function A calls function B and B returns, A is left with a stack containing
first its own variables, then B's variables and then probably some more of its own.
There's no heap and no global variables except constants.
Despite this dynamic allocation of arrays is possible, because Cairo VM supports multiple stacks.
A memory block can be allocated on an auxiliary stack, which is then glued to the main one
when the smart contract is executed to preserve the single-stack memory model.
I wouldn't be surprised if in the future that trick would be used to hide more limitations.
Some built-in functions like hashing are used by writing and reading from special purpose stacks.

Cairo is a mixture of assembly-like and syntax sugared commands,
of which all translate directly to the lower-level ones.
For example you can call `return (my_val)` or push `my_val` to the stack and call `ret`.
Even stack pointer management can be done automatically or manually.
If you want to use a built-in, you need to pass around a reference to its special purpose stack.
There is some syntax sugar around it, but it only saves keystrokes,
you still need to remember and understand what's happening.
Working around memory model limitations is barely hidden at all.
For example when you make a function call, you need to keep track of which variables will
stop being accessible because the function may append an arbitrary amount of data to the stack.

The tooling of Cairo is bare bones, but it does its job.
Test need to be written in Python and there's no package management.
Experimental work has been started to create an EVM to Cairo transpiler.
Time will tell if it'll be a viable option, but I suspect that working around
differences between EVM and StarkNet VM may produce some extremely bloated code.

All these properties make Cairo a difficult and arduous language, especially for newcomers.
It's not very expressive, it's full of implicit behaviors
and it requires deep understanding of its VM to read or modify even simple code.

## zkSync

zkSync is a programmable ZK rollup L2 solution developed by Matter Labs.
Its 1.x version is released on Mainnet and it enables transfers and exchanges on ZK rollups,
but these features are fixed, you can't run a custom smart contract there.
A programmable version is being developed and test releases are available on Rinkeby.
Programmable zkSync will be released on Mainnet only when 2.x version is finished,
but it's unclear when will it happen.
Currently 2.0 is said to be polished and almost ready for premiere,
but because until then its source code will be kept closed, it's hard to verify this claim.

### Zinc

Zinc is a language created for zkSync smart contracts and as of 1.x it's the only supported one.
With 2.0 an official EVM transpiler will be released, but its developers
are expecting it to generate less efficient bytecode than Zinc.

Zinc is based on Rust, with simplified syntax and cut down features.
Its biggest limitation is that it's not turing-complete
because all logic paths must be statically determinable.
There are conditional branches and loops, but they must have fixed number of iterations.
It's impossible to allocate and use dynamic memory, because any meaningful
operations performed on it wouldn't have number of steps known at compilation time.
In zkSync 2.0 the VM will become turing-complete and this limitation will be lifted.

Zinc is easy for newcomers, it takes less than a day to learn it thoroughly.
Its syntax is trivial for developers familiar with Rust and expressive despite simplification.
There are few hidden behaviors and the limitations are easy to learn.
The tooling is decent, you can write unit tests in Zinc and there's innovative package management.
Unlike on Ethereum where source codes and interfaces are usually fetched from NPM and actual
addresses are written by hand, on zkSync the only source of truth is the network itself,
which provides the list of currently deployed contracts, their interfaces and source codes.
