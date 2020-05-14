# Node architecture

Not a complete reference, just introductory material to the key concepts and components in the code. Seen from the perspective of a syncing (not miner) node.

We start from the [Join Testnet](https://docs.lotu.sh/en+join-testnet) documentation. When we run `lotus daemon` the node starts and connects to (bootstrap) peers to download (sync) the chain.

# CLI, API

Basic description of files layout.

Explain the different node interfaces, particularly `FullNode` in `api/api_full.go`.

In `cmd/lotus/daemon.go` the node is created through [dependency injection](https://godoc.org/go.uber.org/fx), most notably the `Online()` and `Repo()` functional options (`node/builder.go`).

The node type, like the `FullNode` we are running to sync, is actually the  `RepoType` (`node/repo/fsrepo.go`). In general the node forms its identities from the different subsystems that would be useful to identify, like the type for the repo, the ID from the libp2p peer network, etc.

Discuss the different stores, point to an IPFS document with the basic definitions of blocks, DAG, CID, and how do we stack stores together wrapping one another. We basically have a single store, in the repository, with different interfaces. The `ChainBlockstore` is the API of the store (with the `/blocks` prefix, a reference to Filecoin blocks or IPFS ones?) where we save all chain related information we fetch in the sync (maybe just mention `ChainStore`). In general we see the IPLD store everywhere, what do we need to know about IPLD in particular, can we just assume everything we save are raw key-value blocks?

In `Repo()`, the private key comes from `libp2p`, which means that our key should be thought of in the network model. The Filecoin key that identifies us is the key we use to communicate in the network.

The `Online()` option has basically all of the important configurations. Skipping `libp2p()` (network related, should be discussed briefly), in the context of the sync, we have `HandleIncomingBlocks()` registered to process network messages (clarify terminology of `libp2p` message which contains anything, and the Filecoin block which contains messages which contains instructions to the VM, the first contains the second that contains the third). We should explain the route of verification starting on `sub.HandleIncomingBlocks()` (already done in the security doc, import or reference it here).

We are ignoring the `HandleIncomingMessages()` route that seems more related with miner activity (although it's registered in the full node), does a non-miner care about them?.

We're skipping the `StorageMiner` section entirely in `Online()`. It is confusing because the *full* in `FullNode` gives the impression of encompassing *all* functionality of the protocol, is `StorageMiner` a subset of it, or are they mutually exclusive with a massive if in `Online()` to just share the `libp2p()` call?.

# Sync

We follow the `NewSyncer` path in `Online()` to the `chain/` directory. `chain.NewSyncer()` in `chain/sync.go` creates the `Syncer` structure (with some attributes that will be explained as needed) which is started together with the application (see `fx.Hook`). When the `Syncer` is created it also creates in its turn a `SyncManager`, what is the domain separation between the two? The `Syncer` starts `SyncManager` which starts the different goroutines (`(*SyncManager).syncWorker()`) that actually process the incoming blocks, with the `doSync` function which is normally the `(*Syncer).Sync()` (going back to the previous structure now).

We receive blocks through the gossipsub topic (explain in a separate network section) but we later fetch the blocks needed through a different protocol in `(*BlockSyncService).HandleStream()` (is this correct? expand on graphsync).

In `(*SyncManager).Start()` we ignore the scheduling and focus just on the worker, `syncWorker()`, which will wait from the scheduler (`syncTargets`) new blocks (encapsulated in `TipSet`s) and call `doSync` (`(*Syncer).Sync`, set in `NewSyncer`) on them.

## Filecoin blocks

Normally we prefix them with the Filecoin qualifier as block is normally used in lower layers like IPFS to refer to a raw piece of binary data (normally associated with the stores).

At this point the reader should be familiarized with the IPFS model of storing and retrieving data by CIDs in a distributed manner. We normally don't transfer data itself in the model but their CIDs identifiers with which the node can fetch the actual data.

In `sub.HandleIncomingBlocks` (`chain/sub/incoming.go`, not to be confused with the one in the DI list), ignoring the verification steps, what arrives is the message with the CID of a block we should sync to (`chain/types/blockmsg.go`):

```Go
type BlockMsg struct {
	Header        *BlockHeader
	BlsMessages   []cid.Cid
	SecpkMessages []cid.Cid
}
```

The only actual data received is the header of the block, with the messages represented by their CIDs. Once we fetch the messages (through the normal IPFS channels, unrelated to the sync "topics", see later) we form the actual Filecoin block (`chain/types/fullblock.go`):

```Go
type FullBlock struct {
	Header        *BlockHeader
	BlsMessages   []*Message
	SecpkMessages []*SignedMessage
}
```

Messages from the same round are collected into a block set (`chain/store/fts.go`):

```Go
type FullTipSet struct {
	Blocks []*types.FullBlock
	tipset *types.TipSet
	cids   []cid.Cid
}
```

The "tipset" denomination might be a bit misleading as it doesn't refer *only* to the tip, the block set from the last round in the chain, but to *any* set of blocks, depending on the context the tipset is the actual tip or not. From its own perspective any block set is always the tip because it assumes nothing from following blocks.

FIXME: How are the tipsets connected? Explain the role of the CIDs and reference an IPFS document about hash chaining in general, how we cannot modify one without modifying all in the chain (Merkle tree).

## Back to sync

In `(*Syncer).collectChain` (which does more than just fetching), called from `(*Syncer).Sync`, we will have a (partial) tipset with the new block received from the network. We will first call `collectHeaders()` to fetch all the block sets that connect it to our current chain (handling potential fork cases).

Assuming a "fast forward" case where the received block connects directly to our current tipset (is that the *head* of the chain?) we will call `syncMessagesAndCheckState` to execute the messages inside the new blocks and update our state up to the new head.

The validations functions called thereafter (mainly `(*Syncer).ValidateBlock()`) will have as a side effect the execution of the messages which will generate the new state, see `(*StateManager).computeTipSetState()` accessed through the `Syncer` structure, and in turn `ApplyBlocks` and `(*VM).ApplyMessage`.

## What happens when we sync to a new head

Expand on how do we arrive to `(*ChainStore).takeHeaviestTipSet()`.

# Genesis block

Seems a good way to start exploring the VM state though the instantiation of its different actors like the storage power.

Explain where do we load the genesis block, the CAR entries, and we set the root of the state. Follow the daemon command option, `chain.LoadGenesis()` saves all the blocks of the CAR file into the store provided by `ChainBlockstore` (this should already be explained in the previous section). The CAR root (MT root?) of those blocks is decoded into the `BlockHeader` that will be the Filecoin (genesis) block of the chain, but most of the information was stored in the raw data (non-Filecoin, what's the correct term?) blocks forwarded directly to the chain, the block header just has a pointer to it.

`SetGenesis` block with name 0. `(ChainStore).SetGenesis()` stores it there.

`MakeInitialStateTree` (`chain/gen/genesis/genesis.go`, used to construct the genesis block (`MakeGenesisBlock()`), constructs the state tree (`NewStateTree`) which is just a "pointer" (root node in the HAMT) to the different actors. It will be continuously used in `(*StateTree).SetActor()` an `types.Actor` structure under a certain `Address` (in the HAMT). (How does the `stateSnaps` work? It has no comments.)

From this point we can follow different setup function like:

* `SetupInitActor()`: see the `AddressMap`.

* `SetupStoragePowerActor`: initial (zero) power state of the chain, most important attributes.

* Account actors in the `template.Accounts`: `SetActor`.

Which other actor type could be helpful at this point?

# Basic concepts

What should be clear at this point either from this document or the spec.

## Addresses

## Accounts

# Sync Topics PubSub

Gossip sub spec and some introduction.

# VM

Do we have a detailed intro to the VM in the spec?

The most important fact about the state is that it is just a collection of data (see actors later) stored under a root CID. Changing a state is just changing the value of that CID pointer (`StateTree.root`, `chain/state/statetree.go`). There is a different hierarchy of "pointers" here. Which is the top one? `ChainStore.heaviest`? (actually a pointer tipset)

This should continue the sync section at the part of the code flow where the messages from the blocks we are syncing are being applied.

In `ValidateBlock`, to check the messages we need to have the state from the previous tipset, we follow `(*StateManager).TipSetState` (`StateManager` structure seen before). Bypassing the cache logic, we would call `computeTipSetState()` on the tipset blocks. Note `ParentStateRoot` as the parent's parent on which we call `(*StateManager).ApplyBlocks`.

We first construct the VM (through an indirect pointer to `NewVM` in `chain/vm/vm.go`), which is at its most basic form the state tree (derived from the parent's state CID, now called `base`) and the `Store` where we will save the new information (state) generated to which the state tree will point.

For each message *independently* we call `(*VM).ApplyMessage()`, after many checks it calls `(*VM).send()` on the actor to which the message is targeted to (explain the send terminology or link to spec, send is the analog of a method call on an object). In the `vm.Runtime` (`(*VM).makeRuntime()`) all the contextual information is saved (including the VM itself), the runtime is the way we pass information to the actors on which we send the messages, used in ` vm.Invoke()`.

## Invoker

How is a message translated to an actual function? (Ugly and undocumented Go reflection ahead, we need to soften this.) The VM contains an `invoker` structure which is the responsible for making the transition, mapping a message code directed to an actor (each actor has its *own* set of codes defined in `specs-actors/actors/builtin/methods.go`) to an actual function (`(*invoker).Invoke()`) stored in itself (`invoker.builtInCode`), `invokeFunc`, which takes the runtime (state communication) and the serialized parameters (for the actor's logic):

```
type invoker struct {
	builtInCode  map[cid.Cid]nativeCode
	builtInState map[cid.Cid]reflect.Type
}

type invokeFunc func(rt runtime.Runtime, params []byte) ([]byte, aerrors.ActorError)
type nativeCode []invokeFunc
```

`(*invoker).Register()` stores for each actor available its set of methods exported through its interface.

```
type Invokee interface {
	Exports() []interface{}
}
```

Basic layout (without reflection details) of `(*invoker).transform()`. From each `Actor` structure take its `Exports()` methods converting them to `invokeFunc`. The actual method (`meth`) is wrapped in another function, that takes care of decoding the serialized parameters and the runtime, `shimCall` will contain the actors code which runs it inside a `defer` to `recover()` from panics (*we fail in the actors code with panics*). The return values will then be (CBOR) marshaled and returned to the VM.

## Back to the VM

`(*VM).ApplyMessage` will receive back from the invoker the serialized response and the `ActorError`. The returned message will be charged the corresponding gas to be stored in `rt.chargeGasSafe(rt.Pricelist().OnChainReturnValue(len(ret)))`, we don't process the return value any further than this (is anything else done with it?)

Besides charging the gas for the returned response, in `(*VM).makeRuntime()` the block store is wrapped around in another `gasChargingBlocks` store responsible for charging any other state information generated (through the runtime):

```
func (bs *gasChargingBlocks) Put(blk block.Block) error {
	bs.chargeGas(bs.pricelist.OnIpldPut(len(blk.RawData())))
```

(What happens when there is an error? Is that just discarded?)

# Look at the constructor of a miner

Follow the `lotus-storage-miner` command to see how a miner is created, from the command to the message to the storage power logic.

# Directory structure so far, main structures seen, their relation

List what are the main directories we should be looking at (e.g., `chain/`) and the most important structures (e.g., `StateTree`, `Runtime`, etc.)

# Tests

Run a few messages and observe state changes. What is the easiest test that also let's us "interact" with it (modify something and observe the difference).
