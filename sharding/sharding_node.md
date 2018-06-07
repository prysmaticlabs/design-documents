# Sharding Node Design Doc
**Status: Final**
**Authors: Preston Van Loon, Raul Jordan, Terence Tsao, Nishant Das**
**Created: 2018-06-03**

[Original Google doc](https://docs.google.com/document/d/1J4AEHTSKDGJpNzWS7ZXPqD0yEjqecb-xjdVkjGrxRMk/edit)
 
## Objective
We aim to provide a robust design to leverage much of the existing work within the go-ethereum project while supporting the complexity of a L1 sharded universe. This doc specifically defines the building blocks for the sharding node or server such that any sharding actor can effectively participate.
## Background
The Ethereum research team and leading community members have identified sharding as a desirable scaling solution that may be able to provide immediate scalability improvements to the order of x10 - x100 in transaction throughput. The fundamental attributes of sharding Ethereum are not a new paradigm in the software world. Databases and other distributed systems have been using sharding for decades with consistent success. Researchers and community members have been working tirelessly to come up with a spec for sharding Ethereum and, in early 2018, the specification has materialized to the point that implementation can begin. Prysmatic Labs, among many other teams, has taken the initiative to implement a sharding client.

The implementation approach chosen by Prysmatic Labs is to extend the existing popular Ethereum client go-ethereum. A long term goal of this project is to eventually merge back upstream with go-ethereum with a seamless migration path for the full mainnet launch of a sharded Ethereum. 

## Overview

As we have started development on a sharding implementation, we have been writing code in an isolated sharding package without much formal design or forward thinking into the framework of a sharding node. The prior work can be accurately described as a discovery and experimentation phase. Now, as we work towards our first major release, we have identified a need to have a formal design for how we bootstrap a sharded node, register different components/services of the application, interact with the mainchain, and run the sharding actor functionality. In short, the problem we are trying to solve is that the code is becoming messy and unstructured. We must resolve this before our first release, Ruby.

This design looks into three major components of the geth sharding node framework. 

**The entrypoint** - This section outlines how we go from the command line to calling .Start() on the geth sharding node.

**Construction of the node** - What are the necessary properties of a node? This section dives into how we construct the node with the required components. Here we also introduce a ShardEthereum struct to act as a container for other critical components and dependencies of the shard actors.

**Service registry** - How should we manage all of the various components and their goroutines? Here we look deep at the geth implementation as an example of how to model our service registry. We explore using geth’s existing service registry and node while considering our rolling our own implementation for the specific needs a sharding node.

Further into the doc, we consider the ProtocolManager concept for sharding. A protocol manager is useful for geth’s many Ethereum sub-protocols, but is it right for initial sharding implementation?

Finally, we outline clear and actionable tasks for core team members and eager contributors from the community. 

## Detailed Design
*Note: all of the code links below are taken from commits on or around June 2nd, 2018. The code for go-ethereum and geth-sharding is rapidly changing so those links may not reflect the latest code.*

### Shard Node

Questions to answer are: what do we gain from using Geth’s main entry point for creating and starting a node vs. our own sharding entry point? What do we lose from this?

#### Entrypoint
We have a few options to bootstrap the shard node and start the service.

**Option 1 -  Using the Main Geth Entry Point**

Upon launching geth, [go-ethereum/cmd/geth/main.go](https://github.com/ethereum/go-ethereum/blob/master/cmd/geth/main.go) configures a full node and then proceeds to start all of its associated services.

Associated services:


| Service | Helpful for a sharding client? |
|---------|--------------|
|[Dashboard](https://github.com/ethereum/go-ethereum/tree/HEAD/dashboard)| Not in its current state.|
|[Support for reading config from file](https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/cmd/geth/config.go#L121)| Probably, yes.|
|[Support for loading context flags into node.Config](https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/cmd/geth/config.go#L127)|No, unless we adopted/needed [node.Config](https://godoc.org/github.com/ethereum/go-ethereum/node#Config).|
|[Whisper service](https://github.com/ethereum/go-ethereum/blob/HEAD/cmd/geth/config.go#L171)|Most likely no. Whisper service is P2P and P2P is going to be quite different.|
|[Eth status service](https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/cmd/geth/config.go#L176) |Not in its current state.|
|[Unlocking an account indefinitely via flag](https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/cmd/geth/main.go#L235) |Yes.|
|[Hardware wallet support](https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/cmd/geth/main.go#L242-L281)|Yes.|
|[Miner](https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/cmd/geth/main.go#L283-L306) |No. (Except for development with a sidecar geth full/light node.)|

Its [makeFullNode](https://github.com/prysmaticlabs/geth-sharding/blob/b5ae2202d6e42fc910f11c3f807e27674123e365/cmd/geth/config.go#L153) function sets up all the default and custom configuration options based on the command line context, including setting up a full node or a light Ethereum node. It calls a complex function called [makeConfigNode](https://github.com/prysmaticlabs/geth-sharding/blob/b5ae2202d6e42fc910f11c3f807e27674123e365/cmd/geth/config.go#L110) that sets up all cli flags or loads existing saved configs from file. It returns a completely transformed config object containing all configuration fields that can be accessed by a node.

A function called [utils.registerEthService](https://github.com/ethereum/go-ethereum/blob/HEAD/cmd/utils/flags.go#L1124-L1124) is also called in this case, which registers either a full node or a light ethereum node:

```golang
// RegisterEthService adds an Ethereum client to the stack.
func RegisterEthService(stack *node.Node, cfg *eth.Config) {
  var err error
  if cfg.SyncMode == downloader.LightSync {
    err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
      return les.New(ctx, cfg)
    })
  } else {
    err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
      fullNode, err := eth.New(ctx, cfg)
      return fullNode, err
    })
  }
  if err != nil {
    Fatalf("Failed to register the Ethereum service: %v", err)
  }
}
```

Then, after all of these services are registered and configuration options are setup, geth starts out the node with a function in main.go that uses gives us access to items related to mining, wallet handling, account handling, and starting a node as defined in the `node` package within the go-ethereum repo.

The node package’s .Start() function is useful for setting up a node database of static/trusted nodes, starting a peer to peer service, and calling the start function of every service registered to geth (i.e. les/eth/sharding services) as well as an RPC server.

In order for the sharding node to be supported in the geth entrypoint, the sharding node must be an instance for the node.Node which is tightly coupled with the current implementation for P2P, among other things. 

We do gain a lot from using geth’s main entry point instead of our own node implementation, but many of the things this entry point kicks off are unnecessary for sharding. At least, they would add a lot of logging bloat as geth currently has and our implementation does not rely on things such as whisper or wallet integration at the moment.

**Option 2) Sharding Specific Entrypoint**

In our first implementations, we have a specific entrypoint for sharding in the [shardingcmd.go](https://github.com/prysmaticlabs/geth-sharding/blob/master/cmd/geth/shardingcmd.go) to avoid the complexity of the geth main entrypoint. The key benefits here are that we can iterate faster on developing the sharding client at the cost of missing many of the auxiliary functionality of the geth full node such as RPC client, ethstats reporting, etc.

This entrypoint might have a similar “makeFullNode” that returns a Sharding node then calls startNode similar to how the Geth entrypoint works.

A simple entry point would create our own shardNode without access to all these other complex features:

```golang
// shardingCmd is the main cmd line entry point for starting a sharding-enabled node.
// A sharding node launches a suite of services including notary services,
// proposer services, and a shardp2p protocol.
func shardingCmd(ctx *cli.Context) error {
  // configures a sharding-enabled node using the cli's context.
  shardingNode, err := sharding.New(ctx)
  if err != nil {
    return fmt.Errorf("could not initialize sharding node instance: %v", err)
  }
  defer shardingNode.Stop()
  // starts a connection to a geth node and kicks off every registered service.
  return shardingNode.Start()
}
```

If we do decide to integrate these other features geth includes, we can use the current geth entry point with heavy modifications such as modifying the RegisterEthService function to register a sharding.New instead of eth.New or les.New when passed in a certain cli flag.For reference, take a look at how complex the startNode function in the current geth node implementation looks in [go-ethereum/cmd/geth/main.go](https://github.com/ethereum/go-ethereum/blob/master/cmd/geth/main.go).

**Decision - Option 2**

Ultimately, we would like to have the auxiliary functionality of the geth entrypoint for a sharded node. However, this should not be considered a blocker or delay for initial implementations of Sharding in this project. We choose to hold off on this issue until after the preliminary Ruby release.

Potential future github issues to create for support of option 1:

- Refactor startNode to accept an interface rather than the specific geth node struct.
- Refactor the startNode process to omit properties that do not apply to sharding. (i.e. mining, genesis block, etc)
- Adapt the sharding node to support startable node interface in the issues above.

Before creating issues for the above, we should evaluate whether the work is worth the 2 or 3 associated services that benefit sharding rather than adding those services to option 2 entrypoint.

#### Constructing a Sharding-Enabled Node

*In go-ethereum/sharding/backend.go*

When Geth first starts, it starts every single service registered to it that complies to a Service interface as defined in the [go-ethereum/node/service.go](https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/node/service.go#L84). One of the most important services is one called the [EthService](https://github.com/ethereum/go-ethereum/blob/HEAD/cmd/utils/flags.go#L1124), which defines an instance of either a full or light Ethereum protocol.

Along the same vein as a light Ethereum node or full Ethereum node, our sharding-enabled Ethereum node needs to return an instance of a struct that complies with this service interface. The les package calls this struct [LightEthereum](https://github.com/ethereum/go-ethereum/blob/master/les/backend.go) while the eth package simply calls it [Ethereum](https://github.com/ethereum/go-ethereum/blob/master/eth/backend.go). In our case, we will define a struct named ShardEthereum that complies with the current system.

We initialize an instance in a function named `.New()` in our sharding package. The responsibilities of this function are:

- Return a `*ShardEthereum` instance, which implements the Service interface
- Creating a new `shardChainDb` instance
- Creating a chain configuration
- Initializing access to important objects related to p2p, events, and more.
- Sets up proposers, notaries, or observers dependent on CLI flags

**The Shard Node’s .Start() Function**

- Starts the main event loops for p2p, discovery, and peer handling
- Starts the main event loops notary/proposer/observer actors in a sharded system
- In the case of a proposer, starts the main event loop for tx relaying and shard tx pool objects

**What Could ShardEthereum Look Like?**
 
A proposed struct type would look as follows:

```golang
type ShardEthereum struct {
  shardConfig     *params.ShardConfig
  txPool          *ShardTxPool
  actor           ShardingActor // this can either be a notary, proposer, or observer
  shardChainDb    ethdb.Database // Shardchain database
  eventFeed       *event.Feed
  accountManager *accounts.Manager
}
```
*In go-ethereum/sharding/backend.go*

**ShardConfig**
The shard config is similar to the [ChainConfig](https://godoc.org/github.com/ethereum/go-ethereum/params#ChainConfig) used by the LES and full geth node. The purpose of this object is hold the necessary information to configure the shard. Some properties might include SMC contract address, any sharding values, and any soft fork block numbers. This struct is meant to contain static properties of the shard configuration.

Initially, the shardConfig might look something like this:

```golang
package params;

var (
	MainnetConfig = &ShardConfig{
		SMCAddress: common.Address(...),
      }
      // Other configurations
)

type ShardConfig struct {
      SMCAddress common.Address
}
```
*sharding/params/shardConfig.go*

**TxPool**

This will define the sharding specific tx pool (tentative; yet to be designed). The TxPool will be needed in the near future. P2P implementation may want to make use of receiving actual test transactions into the transaction pool. In either case, the proposer can insert fake trades into the TxPool for the purpose of unblocking implementation of proposer functionality.

**Actor**

Actor provides access to either a notary/proposer/observer operating the shard node. Actor just needs to implement a .Start() and .Stop() function that this ShardEthereum instance will run. In our minimal system, actor can either be a:

- Notary
- Proposer
- Observer

The actor being attached in this .New() function will be determined based on cli flags and initialized within this .New() function. 

**ShardChainDB**

ShardChainDb provides access to the underlying kv store for sharding objects such as Collations. This is required creating [Shard instances](https://github.com/prysmaticlabs/geth-sharding/blob/b5ae2202d6e42fc910f11c3f807e27674123e365/sharding/shard.go#L20) and saving collation information to persistent storage. This will use a db that implements the [ethdb.Database](https://godoc.org/github.com/ethereum/go-ethereum/ethdb#Database) interface as defined in [go-ethereum/ethdb/interface.go](https://github.com/ethereum/go-ethereum/blob/master/ethdb/interface.go). More info on the Database interface can be found in the docs [here](https://godoc.org/github.com/ethereum/go-ethereum/ethdb).

**EventFeed**

EventFeed deals with event handling in the sharded system. This eventFeed can deal with shared events throughout the system. The event feed will mostly be used to enable P2P related interactions via the different sharding actors. The P2P component will provide subscription channels for clients to hook into for various P2P events. While this property is not immediately necessary for the foundation of the sharding framework, it will be necessary in the very near future for P2P. More details on this will be flushed out in a future P2P design document. 

**Testing**

This shard node will be minimally responsible for what its different components do. It will simply be an entry point that defines all of the responsibilities of a sharded system and calls the .Start() functions of the main event loops. Tests can be kept within each of these components’ respective packages or folders.

#### Services, Service Registry, and Sharding Actors
So now that we have the necessary pieces for a sharding node/server, let’s dive into how all of the pieces fit together and how the sharding actor has access to the services it needs. In order to have proper lifecycle management with the many goroutes, we will use a similar service implementation found in [node/service.go](https://github.com/ethereum/go-ethereum/blob/HEAD/node/service.go).This service interface expects each service to provide the following functions:

- `Start()` The layer to spawn goroutines required by the service. This should be non-blocking.
- `Stop()` This is a blocking operation to terminate all goroutines belonging to the service.

Future implementations may include the Protocols() and APIs() methods that you see in the existing interface, but this is not needed for Ruby. Additionally, we cannot use the exact interface of node/service.go because it has the current p2p.Server struct as an argument for Start() and this is soon to be replaced.

The initial components for the sharded service registry will be registered as follows:

|#|Service|Description|
|--|-----|-----|
|1|ShardEthereum|The primary ethereum client|
|2|P2P|Coming soon™| 
|3|Mainchain RPC for SMC interactions|A struct which holds the SMC related functionality with a ethclient.Client.|
|4|Sharding Actor|One of the following actors: Proposer, Notary|
|5|Other|This could be a console, RPC, metrics endpoint etc.|

The idea here is that the necessary dependencies are registered in order. ShardEthereum lays down the foundation service, P2P references the config and maybe some components of ShardEthereum, and the sharding actor depends on all of the prior services.

**How are the components registered?**

We simplify the responsibilities of the sharding node to simply act as the service registry. It’s API should look like this:

```golang
New(...) Node
// Register adds the service constructor to the registry.
(n *Node) Register(constructor ServiceConstructor) error
// Start calls Start for all of the registered services.
(n *Node) Start() error
// Stop calls Stop for all of the registered services.
(n *Node) Stop() error
```
*sharding/node/node.go*

The role and responsibility of the shard node is clear in this design. In our current implementation, the node is quickly growing in size and responsibilities. As such, we will rip out the functionality for interacting with the mainchain SMC via RPC into a new service and leave the node to act as a simple service registry. 

Each service defines a service constructor which is a function with a [ServiceContext](https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/node/service.go#L32).  The service context provides a [Service()](https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/node/service.go#L61) method which accepts an interface as an argument then uses the reflect library to find the correct registered service from the stack. This is how services can reference their dependencies during construction.

For example, the Eth Stats service requires access to the eth.Ethereum and les.LightEthereum services. As part of the service registry context object, the callback that registers the Eth Stats service can access those services as such: 

```golang
stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
	// Retrieve both eth and les services
	var ethServ *eth.Ethereum
	ctx.Service(&ethServ)

	var lesServ *les.LightEthereum
	ctx.Service(&lesServ)

	return ethstats.New(url, ethServ, lesServ)
});
```
*[Source]( https://github.com/ethereum/go-ethereum/blob/c8dcb9584e1ea77c9b967d2bec5d167c7df410f0/cmd/utils/flags.go#L1163)*

**SMCClient**
The sharding actors need access to a geth node running on the mainchain. The topic of how does the sharding client reach the mainchain is outside the scope of this design, but we know for certain that we can and should use the ethclient.Client and SMC functions. The SMCClient will contain bindings that will be used to interact directly the the SMC.  Also since it the notary/proposer will need to continuously interact with the main chain through RPC calls, we should define this as a Service for consumption of the notary and/or proposer.

```golang
// SMCClient implements Service interface and provides 
// necessary API for interacting with the main chain.
type SMCClient struct {
	ethclient    *ethclient.Client
	smc          *contracts.SMC
      ...
}
```
*sharding/mainchain/smcclient.go*

The SMCClient should serve all of the same purposes as the existing sharding [Node interface](https://github.com/prysmaticlabs/geth-sharding/blob/b5ae2202d6e42fc910f11c3f807e27674123e365/sharding/node/node.go#L36) without any cli.Context related features (i.e. flag values).

**Notary/Proposer ServiceConstructor**
The notary and proposer should be rewritten to accept a specific config for their use case and the appropriate constructor params such as SMCClient, P2PClient, chainDb, etc.

**Notary Config Properties**

- Should deposit? (Previously [n.node.DepositFlagSet()](https://github.com/prysmaticlabs/geth-sharding/blob/b5ae2202d6e42fc910f11c3f807e27674123e365/sharding/notary/service.go#L36))
- ChainDb (Previously [database.NewShardDB](https://github.com/prysmaticlabs/geth-sharding/blob/b5ae2202d6e42fc910f11c3f807e27674123e365/sharding/notary/service.go#L22))
- SMCClient (Previously some components of n.node)

Proposer Config Properties

- ChainDb (Previously [database.NewShardDB](https://github.com/prysmaticlabs/geth-sharding/blob/b5ae2202d6e42fc910f11c3f807e27674123e365/sharding/proposer/service.go#L22))
- SMCClient
- More, yet to be designed

```golang
stack := node.New()

stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
    var p2pClient *p2p.Client
    ctx.Service(&p2pClient)
    var smcClient *mainchain.SMCClient
    ctx.Service(&smcClient)
    
    chainDB, _ := database.NewShardingDB(...) // Or from ShardEthereum
    shouldDeposit := DepositFlag()
    
    // if notary
    return notary.New(&notary.Config{p2pClient, smcClient, chainDB, shouldDeposit})
    
    // if proposer
    return proposer.New(&proposer.Config{p2pClient, chainDB})
});
```

### Shard ProtocolManager

**Note: This component is not necessary for sharding initial implementation!**

The shard protocol manager, would contain much of the same functionality that exists in the ShardEthereum service. Perhaps if the ShardEthereum service needs to be broken into smaller pieces then we can revisit the idea of a shard protocol manager, but this is not necessary for initial implementations.

Looking at the [doc comment of the existing ProtocolManager](https://godoc.org/github.com/ethereum/go-ethereum/eth#NewProtocolManager):

> NewProtocolManager returns a new Ethereum sub protocol manager. The Ethereum sub protocol manages peers capable with the Ethereum network.

This is specifically for sub-protocols, which are not present in initial sharding. Furthermore, these details could be abstracted away as part of the P2P component. 

## Work Estimates
Criteria for resolving the issues below include: a clear understanding of the design outlined in this document, robust tests, design best practices, and excellent GoDocs. 

### Sharded TxPool
*Scope: Minor detail. May be critical to proposer implementation.*

Issue: [prysmaticlabs/geth-sharding#161](https://github.com/prysmaticlabs/geth-sharding/issues/161)

Identified in the section on Constructing the sharding node, we may need a sharding specific transaction pool. The issue here is to design and implement a transaction pool for a sharded node. 

Some questions to answer:
*What are the use cases?*
*How will it work?*
*Why is it necessary rather than a global TxPool with geth’s current implementation?*

### Refactoring sharding node, notary, proposer
*Scope: Large refactoring, breaking changes.*

Done in [prysmaticlabs/geth-sharding#149](https://github.com/prysmaticlabs/geth-sharding/pull/149).

The current implementation of the node holds too much responsibility. As such, we have outlined a plan to refactor this into two specific components: Node (as a service registry only) and the mainchain SMCClient. The bulk of this work lives in refactoring the notary and proposer access to the SMCClient. As part of this refactoring, we should rename NewNotary and NewProposer to “New” for less redundancy. This is idiomatic when calling notary.New or proposer.New. Additionally, the Start functions for notary and proposer need to be updated to be non-blocking.

Furthermore, this task should coincide or at least consider the next issue. Likely, these two issues will be done at the same time by the same developer.

### Implementing service registry & sharding endpoint
*Scope: Minor detail. Dependent on the refactoring above.*

Done in [prysmaticlabs/geth-sharding#149](https://github.com/prysmaticlabs/geth-sharding/pull/149).

After the refactoring above or as part of the above refactoring, modify the sharding entrypoint and service registry as outlined in this document. 

### Replace shard config
*Scope: minor details.*

Issue: [prysmaticlabs/geth-sharding#160](https://github.com/prysmaticlabs/geth-sharding/issues/160).

Rather than having global variables, use a sharding config object as outlined in the shard node construction section.

