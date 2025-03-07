# How to Build a Simple Golang VM

This is part of a series of tutorials for building a Virtual Machine (VM):

- [Introduction to VMs](./introduction-to-vm.md)
- How to Build a Simple Golang VM (this article)
- [How to Build a Complex Golang VM](./create-a-vm-blobvm.md)
- [How to Build a Simple Rust VM](./create-a-simple-rust-vm.md)

## Introduction

In this tutorial, we’ll create a very simple VM called the
[TimestampVM](https://github.com/ava-labs/timestampvm/tree/v1.2.1). Each block in the TimestampVM's
blockchain contains a strictly increasing timestamp when the block was created and a 32-byte payload
of data.

Such a server is useful because it can be used to prove a piece of data existed at the time the
block was created. Suppose you have a book manuscript, and you want to be able to prove in the
future that the manuscript exists today. You can add a block to the blockchain where the block’s
payload is a hash of your manuscript. In the future, you can prove that the manuscript existed today
by showing that the block has the hash of your manuscript in its payload (this follows from the fact
that finding the pre-image of a hash is impossible).

## Prerequisites

Make sure you're familiar with the previous tutorial in this series, which dives into what virtual
machines are.

- [Introduction to VMs](./introduction-to-vm.md)

## TimestampVM Implementation

Now we know the interface our VM must implement and the libraries we can use to build a VM.

Let’s write our VM, which implements `block.ChainVM` and whose blocks implement `snowman.Block`. You
can also follow the code in the [TimestampVM
repository](https://github.com/ava-labs/timestampvm/tree/main).

### Codec

`Codec` is required to encode/decode the block into byte representation. TimestampVM uses the
default codec and manager.

```go title="/timestampvm/codec.go"
const (
	// CodecVersion is the current default codec version
	CodecVersion = 0
)

// Codecs do serialization and deserialization
var (
	Codec codec.Manager
)

func init() {
	// Create default codec and manager
	c := linearcodec.NewDefault()
	Codec = codec.NewDefaultManager()

	// Register codec to manager with CodecVersion
	if err := Codec.RegisterCodec(CodecVersion, c); err != nil {
		panic(err)
	}
}
```

### State

The `State` interface defines the database layer and connections. Each VM should define their own
database methods. `State` embeds the `BlockState` which defines block-related state operations.

```go title="/timestampvm/state.go"
var (
	// These are prefixes for db keys.
	// It's important to set different prefixes for each separate database objects.
	singletonStatePrefix = []byte("singleton")
	blockStatePrefix     = []byte("block")

	_ State = &state{}
)

// State is a wrapper around avax.SingleTonState and BlockState
// State also exposes a few methods needed for managing database commits and close.
type State interface {
	// SingletonState is defined in avalanchego,
	// it is used to understand if db is initialized already.
	avax.SingletonState
	BlockState

	Commit() error
	Close() error
}

type state struct {
	avax.SingletonState
	BlockState

	baseDB *versiondb.Database
}

func NewState(db database.Database, vm *VM) State {
	// create a new baseDB
	baseDB := versiondb.New(db)

	// create a prefixed "blockDB" from baseDB
	blockDB := prefixdb.New(blockStatePrefix, baseDB)
	// create a prefixed "singletonDB" from baseDB
	singletonDB := prefixdb.New(singletonStatePrefix, baseDB)

	// return state with created sub state components
	return &state{
		BlockState:     NewBlockState(blockDB, vm),
		SingletonState: avax.NewSingletonState(singletonDB),
		baseDB:         baseDB,
	}
}

// Commit commits pending operations to baseDB
func (s *state) Commit() error {
	return s.baseDB.Commit()
}

// Close closes the underlying base database
func (s *state) Close() error {
	return s.baseDB.Close()
}
```

#### Block State

This interface and implementation provides storage functions to VM to store and retrieve blocks.

```go title="/timestampvm/block_state.go"
const (
	lastAcceptedByte byte = iota
)

const (
	// maximum block capacity of the cache
	blockCacheSize = 8192
)

// persists lastAccepted block IDs with this key
var lastAcceptedKey = []byte{lastAcceptedByte}

var _ BlockState = &blockState{}

// BlockState defines methods to manage state with Blocks and LastAcceptedIDs.
type BlockState interface {
	GetBlock(blkID ids.ID) (*Block, error)
	PutBlock(blk *Block) error

	GetLastAccepted() (ids.ID, error)
	SetLastAccepted(ids.ID) error
}

// blockState implements BlocksState interface with database and cache.
type blockState struct {
	// cache to store blocks
	blkCache cache.Cacher
	// block database
	blockDB      database.Database
	lastAccepted ids.ID

	// vm reference
	vm *VM
}

// blkWrapper wraps the actual blk bytes and status to persist them together
type blkWrapper struct {
	Blk    []byte         `serialize:"true"`
	Status choices.Status `serialize:"true"`
}

// NewBlockState returns BlockState with a new cache and given db
func NewBlockState(db database.Database, vm *VM) BlockState {
	return &blockState{
		blkCache: &cache.LRU{Size: blockCacheSize},
		blockDB:  db,
		vm:       vm,
	}
}

// GetBlock gets Block from either cache or database
func (s *blockState) GetBlock(blkID ids.ID) (*Block, error) {
	// Check if cache has this blkID
	if blkIntf, cached := s.blkCache.Get(blkID); cached {
		// there is a key but value is nil, so return an error
		if blkIntf == nil {
			return nil, database.ErrNotFound
		}
		// We found it return the block in cache
		return blkIntf.(*Block), nil
	}

	// get block bytes from db with the blkID key
	wrappedBytes, err := s.blockDB.Get(blkID[:])
	if err != nil {
		// we could not find it in the db, let's cache this blkID with nil value
		// so next time we try to fetch the same key we can return error
		// without hitting the database
		if err == database.ErrNotFound {
			s.blkCache.Put(blkID, nil)
		}
		// could not find the block, return error
		return nil, err
	}

	// first decode/unmarshal the block wrapper so we can have status and block bytes
	blkw := blkWrapper{}
	if _, err := Codec.Unmarshal(wrappedBytes, &blkw); err != nil {
		return nil, err
	}

	// now decode/unmarshal the actual block bytes to block
	blk := &Block{}
	if _, err := Codec.Unmarshal(blkw.Blk, blk); err != nil {
		return nil, err
	}

	// initialize block with block bytes, status and vm
	blk.Initialize(blkw.Blk, blkw.Status, s.vm)

	// put block into cache
	s.blkCache.Put(blkID, blk)

	return blk, nil
}

// PutBlock puts block into both database and cache
func (s *blockState) PutBlock(blk *Block) error {
	// create block wrapper with block bytes and status
	blkw := blkWrapper{
		Blk:    blk.Bytes(),
		Status: blk.Status(),
	}

	// encode block wrapper to its byte representation
	wrappedBytes, err := Codec.Marshal(CodecVersion, &blkw)
	if err != nil {
		return err
	}

	blkID := blk.ID()
	// put actual block to cache, so we can directly fetch it from cache
	s.blkCache.Put(blkID, blk)

	// put wrapped block bytes into database
	return s.blockDB.Put(blkID[:], wrappedBytes)
}

// DeleteBlock deletes block from both cache and database
func (s *blockState) DeleteBlock(blkID ids.ID) error {
	s.blkCache.Put(blkID, nil)
	return s.blockDB.Delete(blkID[:])
}

// GetLastAccepted returns last accepted block ID
func (s *blockState) GetLastAccepted() (ids.ID, error) {
	// check if we already have lastAccepted ID in state memory
	if s.lastAccepted != ids.Empty {
		return s.lastAccepted, nil
	}

	// get lastAccepted bytes from database with the fixed lastAcceptedKey
	lastAcceptedBytes, err := s.blockDB.Get(lastAcceptedKey)
	if err != nil {
		return ids.ID{}, err
	}
	// parse bytes to ID
	lastAccepted, err := ids.ToID(lastAcceptedBytes)
	if err != nil {
		return ids.ID{}, err
	}
	// put lastAccepted ID into memory
	s.lastAccepted = lastAccepted
	return lastAccepted, nil
}

// SetLastAccepted persists lastAccepted ID into both cache and database
func (s *blockState) SetLastAccepted(lastAccepted ids.ID) error {
	// if the ID in memory and the given memory are same don't do anything
	if s.lastAccepted == lastAccepted {
		return nil
	}
	// put lastAccepted ID to memory
	s.lastAccepted = lastAccepted
	// persist lastAccepted ID to database with fixed lastAcceptedKey
	return s.blockDB.Put(lastAcceptedKey, lastAccepted[:])
}
```

### Block

Let’s look at our block implementation.

The type declaration is:

<!-- markdownlint-disable MD013 -->

```go title="/timestampvm/block.go"
// Block is a block on the chain.
// Each block contains:
// 1) ParentID
// 2) Height
// 3) Timestamp
// 4) A piece of data (a string)
type Block struct {
	PrntID ids.ID        `serialize:"true" json:"parentID"`  // parent's ID
	Hght   uint64        `serialize:"true" json:"height"`    // This block's height. The genesis block is at height 0.
	Tmstmp int64         `serialize:"true" json:"timestamp"` // Time this block was proposed at
	Dt     [dataLen]byte `serialize:"true" json:"data"`      // Arbitrary data

	id     ids.ID         // hold this block's ID
	bytes  []byte         // this block's encoded bytes
	status choices.Status // block's status
	vm     *VM            // the underlying VM reference, mostly used for state
}
```

<!-- markdownlint-enable MD013 -->

The `serialize:"true"` tag indicates that the field should be included in the byte representation of
the block used when persisting the block or sending it to other nodes.

#### Verify

This method verifies that a block is valid and stores it in the memory. It is important to store the
verified block in the memory and return them in the `vm.GetBlock` method.

```go title="/timestampvm/block.go"
// Verify returns nil iff this block is valid.
// To be valid, it must be that:
// b.parent.Timestamp < b.Timestamp <= [local time] + 1 hour
func (b *Block) Verify() error {
	// Get [b]'s parent
	parentID := b.Parent()
	parent, err := b.vm.getBlock(parentID)
	if err != nil {
		return errDatabaseGet
	}

	// Ensure [b]'s height comes right after its parent's height
	if expectedHeight := parent.Height() + 1; expectedHeight != b.Hght {
		return fmt.Errorf(
			"expected block to have height %d, but found %d",
			expectedHeight,
			b.Hght,
		)
	}

	// Ensure [b]'s timestamp is after its parent's timestamp.
	if b.Timestamp().Unix() < parent.Timestamp().Unix() {
		return errTimestampTooEarly
	}

	// Ensure [b]'s timestamp is not more than an hour
	// ahead of this node's time
	if b.Timestamp().Unix() >= time.Now().Add(time.Hour).Unix() {
		return errTimestampTooLate
	}

	// Put that block to verified blocks in memory
	b.vm.verifiedBlocks[b.ID()] = b

	return nil
}
```

#### Accept

`Accept` is called by the consensus to indicate this block is accepted.

```go title="/timestampvm/block.go"
// Accept sets this block's status to Accepted and sets lastAccepted to this
// block's ID and saves this info to b.vm.DB
func (b *Block) Accept() error {
	b.SetStatus(choices.Accepted) // Change state of this block
	blkID := b.ID()

	// Persist data
	if err := b.vm.state.PutBlock(b); err != nil {
		return err
	}

	// Set last accepted ID to this block ID
	if err := b.vm.state.SetLastAccepted(blkID); err != nil {
		return err
	}

	// Delete this block from verified blocks as it's accepted
	delete(b.vm.verifiedBlocks, b.ID())

	// Commit changes to database
	return b.vm.state.Commit()
}
```

#### Reject

`Reject` is called by the consensus to indicate this block is rejected.

```go title="/timestampvm/block.go"
// Reject sets this block's status to Rejected and saves the status in state
// Recall that b.vm.DB.Commit() must be called to persist to the DB
func (b *Block) Reject() error {
	b.SetStatus(choices.Rejected) // Change state of this block
	if err := b.vm.state.PutBlock(b); err != nil {
		return err
	}
	// Delete this block from verified blocks as it's rejected
	delete(b.vm.verifiedBlocks, b.ID())
	// Commit changes to database
	return b.vm.state.Commit()
}
```

#### Block Field Methods

These methods are required by the `snowman.Block` interface.

```go title="/timestampvm/block.go"
// ID returns the ID of this block
func (b *Block) ID() ids.ID { return b.id }

// ParentID returns [b]'s parent's ID
func (b *Block) Parent() ids.ID { return b.PrntID }

// Height returns this block's height. The genesis block has height 0.
func (b *Block) Height() uint64 { return b.Hght }

// Timestamp returns this block's time. The genesis block has time 0.
func (b *Block) Timestamp() time.Time { return time.Unix(b.Tmstmp, 0) }

// Status returns the status of this block
func (b *Block) Status() choices.Status { return b.status }

// Bytes returns the byte repr. of this block
func (b *Block) Bytes() []byte { return b.bytes }
```

#### Helper Functions

These methods are convenience methods for blocks, they're not a part of the block interface.

```go
// Initialize sets [b.bytes] to [bytes], [b.id] to hash([b.bytes]),
// [b.status] to [status] and [b.vm] to [vm]
func (b *Block) Initialize(bytes []byte, status choices.Status, vm *VM) {
	b.bytes = bytes
	b.id = hashing.ComputeHash256Array(b.bytes)
	b.status = status
	b.vm = vm
}

// SetStatus sets the status of this block
func (b *Block) SetStatus(status choices.Status) { b.status = status }
```

### Virtual Machine

Now, let’s look at our timestamp VM implementation, which implements the `block.ChainVM` interface.

The declaration is:

```go title="/timestampvm/vm.go"
// This Virtual Machine defines a blockchain that acts as a timestamp server
// Each block contains data (a payload) and the timestamp when it was created

const (
  dataLen = 32
	Name    = "timestampvm"
)

// VM implements the snowman.VM interface
// Each block in this chain contains a Unix timestamp
// and a piece of data (a string)
type VM struct {
	// The context of this vm
	ctx       *snow.Context
	dbManager manager.Manager

	// State of this VM
	state State

	// ID of the preferred block
	preferred ids.ID

	// channel to send messages to the consensus engine
	toEngine chan<- common.Message

	// Proposed pieces of data that haven't been put into a block and proposed yet
	mempool [][dataLen]byte

	// Block ID --> Block
	// Each element is a block that passed verification but
	// hasn't yet been accepted/rejected
	verifiedBlocks map[ids.ID]*Block
}
```

#### Initialize

This method is called when a new instance of VM is initialized. Genesis block is created under this method.

```go title="/timestampvm/vm.go"
// Initialize this vm
// [ctx] is this vm's context
// [dbManager] is the manager of this vm's database
// [toEngine] is used to notify the consensus engine that new blocks are
//   ready to be added to consensus
// The data in the genesis block is [genesisData]
func (vm *VM) Initialize(
	ctx *snow.Context,
	dbManager manager.Manager,
	genesisData []byte,
	upgradeData []byte,
	configData []byte,
	toEngine chan<- common.Message,
	_ []*common.Fx,
	_ common.AppSender,
) error {
	version, err := vm.Version()
	if err != nil {
		log.Error("error initializing Timestamp VM: %v", err)
		return err
	}
	log.Info("Initializing Timestamp VM", "Version", version)

	vm.dbManager = dbManager
	vm.ctx = ctx
	vm.toEngine = toEngine
	vm.verifiedBlocks = make(map[ids.ID]*Block)

	// Create new state
	vm.state = NewState(vm.dbManager.Current().Database, vm)

	// Initialize genesis
	if err := vm.initGenesis(genesisData); err != nil {
		return err
	}

	// Get last accepted
	lastAccepted, err := vm.state.GetLastAccepted()
	if err != nil {
		return err
	}

	ctx.Log.Info("initializing last accepted block as %s", lastAccepted)

	// Build off the most recently accepted block
	return vm.SetPreference(lastAccepted)
}
```

##### `initGenesis`

`initGenesis` is a helper method which initializes the genesis block from given bytes and puts into
the state.

```go title="/timestampvm/vm.go"
// Initializes Genesis if required
func (vm *VM) initGenesis(genesisData []byte) error {
	stateInitialized, err := vm.state.IsInitialized()
	if err != nil {
		return err
	}

	// if state is already initialized, skip init genesis.
	if stateInitialized {
		return nil
	}

	if len(genesisData) > dataLen {
		return errBadGenesisBytes
	}

	// genesisData is a byte slice but each block contains an byte array
	// Take the first [dataLen] bytes from genesisData and put them in an array
	var genesisDataArr [dataLen]byte
	copy(genesisDataArr[:], genesisData)

	// Create the genesis block
	// Timestamp of genesis block is 0. It has no parent.
	genesisBlock, err := vm.NewBlock(ids.Empty, 0, genesisDataArr, time.Unix(0, 0))
	if err != nil {
		log.Error("error while creating genesis block: %v", err)
		return err
	}

	// Put genesis block to state
	if err := vm.state.PutBlock(genesisBlock); err != nil {
		log.Error("error while saving genesis block: %v", err)
		return err
	}

	// Accept the genesis block
	// Sets [vm.lastAccepted] and [vm.preferred]
	if err := genesisBlock.Accept(); err != nil {
		return fmt.Errorf("error accepting genesis block: %w", err)
	}

	// Mark this vm's state as initialized, so we can skip initGenesis in further restarts
	if err := vm.state.SetInitialized(); err != nil {
		return fmt.Errorf("error while setting db to initialized: %w", err)
	}

	// Flush VM's database to underlying db
	return vm.state.Commit()
}
```

#### CreateHandlers

Registered handlers defined in `Service`. See [below](create-a-vm-timestampvm.md#api) for more on APIs.

```go title="/timestampvm/vm.go"
// CreateHandlers returns a map where:
// Keys: The path extension for this blockchain's API (empty in this case)
// Values: The handler for the API
// In this case, our blockchain has only one API, which we name timestamp,
// and it has no path extension, so the API endpoint:
// [Node IP]/ext/bc/[this blockchain's ID]
// See API section in documentation for more information
func (vm *VM) CreateHandlers() (map[string]*common.HTTPHandler, error) {
	server := rpc.NewServer()
	server.RegisterCodec(json.NewCodec(), "application/json")
	server.RegisterCodec(json.NewCodec(), "application/json;charset=UTF-8")
    // Name is "timestampvm"
	if err := server.RegisterService(&Service{vm: vm}, Name); err != nil {
		return nil, err
	}

	return map[string]*common.HTTPHandler{
		"": {
			Handler: server,
		},
	}, nil
}
```

#### CreateStaticHandlers

Registers static handlers defined in `StaticService`. See
[below](create-a-vm-timestampvm.md#static-api) for more on static APIs.

```go title="/timestampvm/vm.go"
// CreateStaticHandlers returns a map where:
// Keys: The path extension for this VM's static API
// Values: The handler for that static API
func (vm *VM) CreateStaticHandlers() (map[string]*common.HTTPHandler, error) {
	server := rpc.NewServer()
	server.RegisterCodec(json.NewCodec(), "application/json")
	server.RegisterCodec(json.NewCodec(), "application/json;charset=UTF-8")
	if err := server.RegisterService(&StaticService{}, Name); err != nil {
		return nil, err
	}

	return map[string]*common.HTTPHandler{
		"": {
			LockOptions: common.NoLock,
			Handler:     server,
		},
	}, nil
}
```

#### BuildBock

`BuildBlock` builds a new block and returns it. This is mainly requested by the consensus engine.

```go title="/timestampvm/vm.go"
// BuildBlock returns a block that this vm wants to add to consensus
func (vm *VM) BuildBlock() (snowman.Block, error) {
	if len(vm.mempool) == 0 { // There is no block to be built
		return nil, errNoPendingBlocks
	}

	// Get the value to put in the new block
	value := vm.mempool[0]
	vm.mempool = vm.mempool[1:]

	// Notify consensus engine that there are more pending data for blocks
	// (if that is the case) when done building this block
	if len(vm.mempool) > 0 {
		defer vm.NotifyBlockReady()
	}

	// Gets Preferred Block
	preferredBlock, err := vm.getBlock(vm.preferred)
	if err != nil {
		return nil, fmt.Errorf("couldn't get preferred block: %w", err)
	}
	preferredHeight := preferredBlock.Height()

	// Build the block with preferred height
	newBlock, err := vm.NewBlock(vm.preferred, preferredHeight+1, value, time.Now())
	if err != nil {
		return nil, fmt.Errorf("couldn't build block: %w", err)
	}

	// Verifies block
	if err := newBlock.Verify(); err != nil {
		return nil, err
	}
	return newBlock, nil
}
```

#### NotifyBlockReady

`NotifyBlockReady` is a helper method that can send messages to the consensus engine through
`toEngine` channel.

```go title="/timestampvm/vm.go"
// NotifyBlockReady tells the consensus engine that a new block
// is ready to be created
func (vm *VM) NotifyBlockReady() {
	select {
	case vm.toEngine <- common.PendingTxs:
	default:
		vm.ctx.Log.Debug("dropping message to consensus engine")
	}
}
```

#### GetBlock

`GetBlock` returns the block with the given block ID.

```go title="/timestampvm/vm.go"
// GetBlock implements the snowman.ChainVM interface
func (vm *VM) GetBlock(blkID ids.ID) (snowman.Block, error) { return vm.getBlock(blkID) }

func (vm *VM) getBlock(blkID ids.ID) (*Block, error) {
	// If block is in memory, return it.
	if blk, exists := vm.verifiedBlocks[blkID]; exists {
		return blk, nil
	}

	return vm.state.GetBlock(blkID)
}
```

#### `proposeBlock`

This method adds a piece of data to the mempool and notifies the consensus layer of the blockchain
that a new block is ready to be built and voted on. This is called by API method `ProposeBlock`,
which we’ll see later.

```go title="/timestampvm/vm.go"
// proposeBlock appends [data] to [p.mempool].
// Then it notifies the consensus engine
// that a new block is ready to be added to consensus
// (namely, a block with data [data])
func (vm *VM) proposeBlock(data [dataLen]byte) {
    vm.mempool = append(vm.mempool, data)
    vm.NotifyBlockReady()
}
```

#### ParseBlock

Parse a block from its byte representation.

```go title="/timestampvm/vm.go"
// ParseBlock parses [bytes] to a snowman.Block
// This function is used by the vm's state to unmarshal blocks saved in state
// and by the consensus layer when it receives the byte representation of a block
// from another node
func (vm *VM) ParseBlock(bytes []byte) (snowman.Block, error) {
	// A new empty block
	block := &Block{}

	// Unmarshal the byte repr. of the block into our empty block
	_, err := Codec.Unmarshal(bytes, block)
	if err != nil {
		return nil, err
	}

	// Initialize the block
	block.Initialize(bytes, choices.Processing, vm)

	if blk, err := vm.getBlock(block.ID()); err == nil {
		// If we have seen this block before, return it with the most up-to-date
		// info
		return blk, nil
	}

	// Return the block
	return block, nil
}
```

#### NewBlock

`NewBlock` creates a new block with given block parameters.

<!-- markdownlint-disable MD013 -->

```go title="/timestampvm/vm.go"
// NewBlock returns a new Block where:
// - the block's parent is [parentID]
// - the block's data is [data]
// - the block's timestamp is [timestamp]
func (vm *VM) NewBlock(parentID ids.ID, height uint64, data [dataLen]byte, timestamp time.Time) (*Block, error) {
	block := &Block{
		PrntID: parentID,
		Hght:   height,
		Tmstmp: timestamp.Unix(),
		Dt:     data,
	}

	// Get the byte representation of the block
	blockBytes, err := Codec.Marshal(CodecVersion, block)
	if err != nil {
		return nil, err
	}

	// Initialize the block by providing it with its byte representation
	// and a reference to this VM
	block.Initialize(blockBytes, choices.Processing, vm)
	return block, nil
}
```

<!-- markdownlint-enable MD013 -->

#### SetPreference

`SetPreference` implements the `block.ChainVM`. It sets the preferred block ID.

```go title="/timestampvm/vm.go"
// SetPreference sets the block with ID [ID] as the preferred block
func (vm *VM) SetPreference(id ids.ID) error {
	vm.preferred = id
	return nil
}
```

#### Other Functions

These functions needs to be implemented for `block.ChainVM`. Most of them are just blank functions
returning `nil`.

<!-- markdownlint-disable MD013 -->

```go title="/timestampvm/vm.go"
// Bootstrapped marks this VM as bootstrapped
func (vm *VM) Bootstrapped() error { return nil }

// Bootstrapping marks this VM as bootstrapping
func (vm *VM) Bootstrapping() error { return nil }

// Returns this VM's version
func (vm *VM) Version() (string, error) {
	return Version.String(), nil
}

func (vm *VM) Connected(id ids.ShortID, nodeVersion version.Application) error {
	return nil // noop
}

func (vm *VM) Disconnected(id ids.ShortID) error {
	return nil // noop
}

// This VM doesn't (currently) have any app-specific messages
func (vm *VM) AppGossip(nodeID ids.ShortID, msg []byte) error {
	return nil
}

// This VM doesn't (currently) have any app-specific messages
func (vm *VM) AppRequest(nodeID ids.ShortID, requestID uint32, time time.Time, request []byte) error {
	return nil
}

// This VM doesn't (currently) have any app-specific messages
func (vm *VM) AppResponse(nodeID ids.ShortID, requestID uint32, response []byte) error {
	return nil
}

// This VM doesn't (currently) have any app-specific messages
func (vm *VM) AppRequestFailed(nodeID ids.ShortID, requestID uint32) error {
	return nil
}

// Health implements the common.VM interface
func (vm *VM) HealthCheck() (interface{}, error) { return nil, nil }
```

<!-- markdownlint-enable MD013 -->

### Factory

VMs should implement the `Factory` interface. `New` method in the interface returns a new VM instance.

```go title="/timestampvm/factory.go"
var _ vms.Factory = &Factory{}

// Factory ...
type Factory struct{}

// New ...
func (f *Factory) New(*snow.Context) (interface{}, error) { return &VM{}, nil }
```

### Static API

A VM may have a static API, which allows clients to call methods that do not query or update the
state of a particular blockchain, but rather apply to the VM as a whole. This is analogous to static
methods in computer programming. AvalancheGo uses [Gorilla’s RPC
library](https://www.gorillatoolkit.org/pkg/rpc) to implement HTTP APIs.

`StaticService` implements the static API for our VM.

```go title="/timestampvm/static_service.go"
// StaticService defines the static API for the timestamp vm
type StaticService struct{}
```

#### Encode

For each API method, there is:

- A struct that defines the method’s arguments
- A struct that defines the method’s return values
- A method that implements the API method, and is parameterized on the above 2 structs

This API method encodes a string to its byte representation using a given encoding scheme. It can be
used to encode data that is then put in a block and proposed as the next block for this chain.

```go title="/timestampvm/static_service.go"
// EncodeArgs are arguments for Encode
type EncodeArgs struct {
    Data     string              `json:"data"`
    Encoding formatting.Encoding `json:"encoding"`
    Length   int32               `json:"length"`
}

// EncodeReply is the reply from Encoder
type EncodeReply struct {
    Bytes    string              `json:"bytes"`
    Encoding formatting.Encoding `json:"encoding"`
}

// Encoder returns the encoded data
func (ss *StaticService) Encode(_ *http.Request, args *EncodeArgs, reply *EncodeReply) error {
    if len(args.Data) == 0 {
        return fmt.Errorf("argument Data cannot be empty")
    }
    var argBytes []byte
    if args.Length > 0 {
        argBytes = make([]byte, args.Length)
        copy(argBytes, args.Data)
    } else {
        argBytes = []byte(args.Data)
    }

    bytes, err := formatting.EncodeWithChecksum(args.Encoding, argBytes)
    if err != nil {
        return fmt.Errorf("couldn't encode data as string: %s", err)
    }
    reply.Bytes = bytes
    reply.Encoding = args.Encoding
    return nil
}
```

#### Decode

This API method is the inverse of `Encode`.

```go title="/timestampvm/static_service.go"
// DecoderArgs are arguments for Decode
type DecoderArgs struct {
    Bytes    string              `json:"bytes"`
    Encoding formatting.Encoding `json:"encoding"`
}

// DecoderReply is the reply from Decoder
type DecoderReply struct {
    Data     string              `json:"data"`
    Encoding formatting.Encoding `json:"encoding"`
}

// Decoder returns the Decoded data
func (ss *StaticService) Decode(_ *http.Request, args *DecoderArgs, reply *DecoderReply) error {
    bytes, err := formatting.Decode(args.Encoding, args.Bytes)
    if err != nil {
        return fmt.Errorf("couldn't Decode data as string: %s", err)
    }
    reply.Data = string(bytes)
    reply.Encoding = args.Encoding
    return nil
}
```

### API

A VM may also have a non-static HTTP API, which allows clients to query and update the blockchain's state.

`Service`'s declaration is:

```go title="/timestampvm/service.go"
// Service is the API service for this VM
type Service struct{ vm *VM }
```

Note that this struct has a reference to the VM, so it can query and update state.

This VM's API has two methods. One allows a client to get a block by its ID. The other allows a
client to propose the next block of this blockchain. The blockchain ID in the endpoint changes,
since every blockchain has an unique ID.

#### `timestampvm.getBlock`

Get a block by its ID. If no ID is provided, get the latest block.

##### `getBlock` Signature

```sh
timestampvm.getBlock({id: string}) ->
    {
        id: string,
        data: string,
        timestamp: int,
        parentID: string
    }
```

- `id` is the ID of the block being retrieved. If omitted from arguments, gets the latest block
- `data` is the base 58 (with checksum) representation of the block’s 32 byte payload
- `timestamp` is the Unix timestamp when this block was created
- `parentID` is the block’s parent

##### `getBlock` Example Call

```bash
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "timestampvm.getBlock",
    "params":{
        "id":"xqQV1jDnCXDxhfnNT7tDBcXeoH2jC3Hh7Pyv4GXE1z1hfup5K"
    },
    "id": 1
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/bc/sw813hGSWH8pdU9uzaYy9fCtYFfY7AjDd2c9rm64SbApnvjmk
```

##### `getBlock` Example Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "timestamp": "1581717416",
    "data": "11111111111111111111111111111111LpoYY",
    "id": "xqQV1jDnCXDxhfnNT7tDBcXeoH2jC3Hh7Pyv4GXE1z1hfup5K",
    "parentID": "22XLgiM5dfCwTY9iZnVk8ZPuPe3aSrdVr5Dfrbxd3ejpJd7oef"
  },
  "id": 1
}
```

##### `getBlock` Implementation

```go title="/timestampvm/service.go"
// GetBlockArgs are the arguments to GetBlock
type GetBlockArgs struct {
	// ID of the block we're getting.
	// If left blank, gets the latest block
	ID *ids.ID `json:"id"`
}

// GetBlockReply is the reply from GetBlock
type GetBlockReply struct {
	Timestamp json.Uint64 `json:"timestamp"` // Timestamp of most recent block
	Data      string      `json:"data"`      // Data in the most recent block. Base 58 repr. of 5 bytes.
	ID        ids.ID      `json:"id"`        // String repr. of ID of the most recent block
	ParentID  ids.ID      `json:"parentID"`  // String repr. of ID of the most recent block's parent
}

// GetBlock gets the block whose ID is [args.ID]
// If [args.ID] is empty, get the latest block
func (s *Service) GetBlock(_ *http.Request, args *GetBlockArgs, reply *GetBlockReply) error {
	// If an ID is given, parse its string representation to an ids.ID
	// If no ID is given, ID becomes the ID of last accepted block
	var (
		id  ids.ID
		err error
	)

	if args.ID == nil {
		id, err = s.vm.state.GetLastAccepted()
		if err != nil {
			return errCannotGetLastAccepted
		}
	} else {
		id = *args.ID
	}

	// Get the block from the database
	block, err := s.vm.getBlock(id)
	if err != nil {
		return errNoSuchBlock
	}

	// Fill out the response with the block's data
	reply.ID = block.ID()
	reply.Timestamp = json.Uint64(block.Timestamp().Unix())
	reply.ParentID = block.Parent()
	data := block.Data()
	reply.Data, err = formatting.EncodeWithChecksum(formatting.CB58, data[:])

	return err
}
```

#### `timestampvm.proposeBlock`

Propose the next block on this blockchain.

##### `proposeBlock` Signature

```sh
timestampvm.proposeBlock({data: string}) -> {success: bool}
```

- `data` is the base 58 (with checksum) representation of the proposed block’s 32 byte payload.

##### `proposeBlock` Example Call

```bash
curl -X POST --data '{
    "jsonrpc": "2.0",
    "method": "timestampvm.proposeBlock",
    "params":{
        "data":"SkB92YpWm4Q2iPnLGCuDPZPgUQMxajqQQuz91oi3xD984f8r"
    },
    "id": 1
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/bc/sw813hGSWH8pdU9uzaYy9fCtYFfY7AjDd2c9rm64SbApnvjmk
```

###### `proposeBlock` Example Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "Success": true
  },
  "id": 1
}
```

##### `proposeBlock` Implementation

<!-- markdownlint-disable MD013 -->

```go title="/timestampvm/service.go"
// ProposeBlockArgs are the arguments to ProposeValue
type ProposeBlockArgs struct {
    // Data for the new block. Must be base 58 encoding (with checksum) of 32 bytes.
    Data string
}

// ProposeBlockReply is the reply from function ProposeBlock
type ProposeBlockReply struct{
    // True if the operation was successful
    Success bool
}

// ProposeBlock is an API method to propose a new block whose data is [args].Data.
// [args].Data must be a string repr. of a 32 byte array
func (s *Service) ProposeBlock(_ *http.Request, args *ProposeBlockArgs, reply *ProposeBlockReply) error {
	bytes, err := formatting.Decode(formatting.CB58, args.Data)
	if err != nil || len(bytes) != dataLen {
		return errBadData
	}

	var data [dataLen]byte         // The data as an array of bytes
	copy(data[:], bytes[:dataLen]) // Copy the bytes in dataSlice to data

	s.vm.proposeBlock(data)
	reply.Success = true
	return nil
}
```

<!-- markdownlint-enable MD013 -->

### Plugin

In order to make this VM compatible with `go-plugin`, we need to define a `main` package and method,
which serves our VM over gRPC so that AvalancheGo can call its methods.

`main.go`'s contents are:

```go title="/main/main.go"
func main() {
    log.Root().SetHandler(log.LvlFilterHandler(log.LvlDebug, log.StreamHandler(os.Stderr, log.TerminalFormat())))
    plugin.Serve(&plugin.ServeConfig{
        HandshakeConfig: rpcchainvm.Handshake,
        Plugins: map[string]plugin.Plugin{
            "vm": rpcchainvm.New(&timestampvm.VM{}),
        },

        // A non-nil value here enables gRPC serving for this plugin...
        GRPCServer: plugin.DefaultGRPCServer,
    })
}
```

Now AvalancheGo's `rpcchainvm` can connect to this plugin and calls its methods.

### Executable Binary

This VM has a [build script](https://github.com/ava-labs/timestampvm/blob/v1.2.1/scripts/build.sh)
that builds an executable of this VM (when invoked, it runs the `main` method from above.)

The path to the executable, as well as its name, can be provided to the build script via arguments.
For example:

```text
./scripts/build.sh ../avalanchego/build/plugins timestampvm
```

If no argument is given, the path defaults to a binary named with default VM ID:
`$GOPATH/src/github.com/ava-labs/avalanchego/build/plugins/tGas3T58KzdjLHhBDMnH2TvrddhqTji5iZAMZ3RXs2NLpSnhH`

This name `tGas3T58KzdjLHhBDMnH2TvrddhqTji5iZAMZ3RXs2NLpSnhH` is the CB58 encoded 32 byte identifier
for the VM. For the timestampvm, this is the string "timestampvm" zero-extended in a 32 byte array
and encoded in CB58. 

### VM Aliases

Each VM has a predefined, static ID. For instance, the default ID of the TimestampVM is:
`tGas3T58KzdjLHhBDMnH2TvrddhqTji5iZAMZ3RXs2NLpSnhH`.

It's possible to give an alias for these IDs. For example, we can alias `TimestampVM` by creating a
JSON file at `~/.avalanchego/configs/vms/aliases.json` with:

```json
{
  "tGas3T58KzdjLHhBDMnH2TvrddhqTji5iZAMZ3RXs2NLpSnhH": [
    "timestampvm",
    "timestamp"
  ]
}
```

### Installing a VM

AvalancheGo searches for and registers plugins under the `plugins` [directory](../nodes/maintain/avalanchego-config-flags.md#--plugin-dir-string).

To install the virtual machine onto your node, you need to move the built virtual machine binary
under this directory. Virtual machine executable names must be either a full virtual machine ID
(encoded in CB58), or a VM alias.

Copy the binary into the plugins directory.

```bash
cp -n <path to your binary> $GOPATH/src/github.com/ava-labs/avalanchego/build/plugins/
```

#### Node Is Not Running

If your node isn't running yet, you can install all virtual machines under your `plugin` directory
by starting the node.

#### Node Is Already Running

Load the binary with the `loadVMs` API.

```bash
curl -sX POST --data '{
    "jsonrpc":"2.0",
    "id"     :1,
    "method" :"admin.loadVMs",
    "params" :{}
}' -H 'content-type:application/json;' 127.0.0.1:9650/ext/admin
```

Confirm the response of `loadVMs` contains the newly installed virtual machine
`tGas3T58KzdjLHhBDMnH2TvrddhqTji5iZAMZ3RXs2NLpSnhH`. You'll see this virtual machine as well as any
others that weren't already installed previously in the response.

```json
{
  "jsonrpc": "2.0",
  "result": {
    "newVMs": {
      "tGas3T58KzdjLHhBDMnH2TvrddhqTji5iZAMZ3RXs2NLpSnhH": [
        "timestampvm",
        "timestamp"
      ],
      "spdxUxVJQbX85MGxMHbKw1sHxMnSqJ3QBzDyDYEP3h6TLuxqQ": []
    }
  },
  "id": 1
}
```

Now, this VM's static API can be accessed at endpoints `/ext/vm/timestampvm` and
`/ext/vm/timestamp`. For more details about VM configs, see
[here](../nodes/maintain/avalanchego-config-flags.md#vm-configs).

In this tutorial, we used the VM's ID as the executable name to simplify the process. However,
AvalancheGo would also accept `timestampvm` or `timestamp` since those are registered aliases in
previous step.

## Wrapping Up

That’s it! That’s the entire implementation of a VM which defines a blockchain-based timestamp server.

In this tutorial, we learned:

- The `block.ChainVM` interface, which all VMs that define a linear chain must implement
- The `snowman.Block` interface, which all blocks that are part of a linear chain must implement
- The `rpcchainvm` type, which allows blockchains to run in their own processes.
- An actual implementation of `block.ChainVM` and `snowman.Block`.
