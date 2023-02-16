# geth-mamoru-core-sdk

This implementation of the geth-mamoru-core-sdk requires the use of `go-ethereum` version <= `1.10.0`. 

## Usage
Install the package in an Ethereum-based project

```shell
go get github.com/Mamoru-Foundation/geth-mamoru-core-sdk
```

### For light mode (--syncmode light)

Add the following to import statements in the file `go-ethereum/light/lightchain.go`:

```go
import (
    mamoru "github.com/Mamoru-Foundation/geth-mamoru-core-sdk"
    "github.com/Mamoru-Foundation/geth-mamoru-core-sdk/call_tracer"
)
```
    
Then, paste the following code into the Ethereum light client file `go-ethereum/light/lightchain.go`:

```go
	////////////////////////////////////////////////////////////////////////////
    if !mamoru.IsSnifferEnable() {
		return 0, nil
    }
    ctx := context.Background()
    
    lastBlock, err := lc.GetBlockByNumber(ctx, block.NumberU64())
    if err != nil {
        return 0, err
    }
    
    parentBlock, err := lc.GetBlockByHash(ctx, block.ParentHash())
    if err != nil {
        return 0, err
    }
    
    stateDb := NewState(ctx, parentBlock.Header(), lc.Odr())
    receipts, err := GetBlockReceipts(ctx, lc.Odr(), lastBlock.Hash(), lastBlock.Number().Uint64())
    if err != nil {
        return 0, err
    }
    
    startTime := time.Now()
    log.Info("Mamoru Sniffer start", "number", block.NumberU64())
    tracer := mamoru.NewTracer(mamoru.NewFeed(lc.Config()))
    
    tracer.FeedBlock(block)
    tracer.FeedTransactions(block, receipts)
    tracer.FeedEvents(receipts)
    
    //Launch EVM and Collect Call Trace data
    callFrames, err := call_tracer.TraceBlock(ctx, call_tracer.NewTracerConfig(stateDb.Copy(), lc.Config(), lc), lastBlock)
    if err != nil {
        log.Error("Mamoru Sniffer Tracer Error", "err", err)
        return 0, err
    }
    for _, call := range callFrames {
        result := call.Result
        tracer.FeedCalTraces(result, block.NumberU64())
    }
    
    tracer.Send(startTime, block.Number(), block.Hash())
////////////////////////////////////////////////////////////////////////////
```

### For full/snap mode  (--syncmode full|snap)

Add the following to import statements in the file `go-ethereum/core/blockchain.go`

```go
import (
    mamoru "github.com/Mamoru-Foundation/geth-mamoru-core-sdk"
)
```

Enable debug mode and insert tracer instance to function `func NewBlockChain()`

```go
    ...
        diffPeersToDiffHashes: make(map[string]map[common.Hash]struct{}),
    }
    
    bc.prefetcher = NewStatePrefetcher(chainConfig, bc, engine)
    bc.forker = NewForkChoice(bc, shouldPreserve)
    bc.validator = NewBlockValidator(chainConfig, bc, engine)
    bc.processor = NewStateProcessor(chainConfig, bc, engine)
    
    var err error
//////////////////////////////////////////////////////////////
    tracer, err := mamoru.NewCallTracer(true)
    if err != nil {
        return nil, err
    }
    bc.vmConfig.Tracer = tracer
    bc.vmConfig.Debug = true
//////////////////////////////////////////////////////////////
...
```

Insert the main tracer code at the end of the function `func (bc *BlockChain) writeBlockAndSetHead()`

```go
    ...
	////////////////////////////////////////////////////////////
	if !mamoru.IsSnifferEnable() {
		return 0, nil
	}

	startTime := time.Now()
	log.Info("Mamoru Sniffer start", "number", block.NumberU64())
	tracer := mamoru.NewTracer(mamoru.NewFeed(bc.chainConfig))

	tracer.FeedBlock(block)
	tracer.FeedTransactions(block, receipts)
	tracer.FeedEvents(receipts)
	// Collect Call Trace data  from EVM
	if callTracer, ok := bc.GetVMConfig().Tracer.(*mamoru.CallTracer); ok {
		result, err := callTracer.GetResult()
		if err != nil {
			log.Error("Mamoru Sniffer Tracer Error", "err", err)
			return 0, err
		}
		tracer.FeedCalTraces(result, block.NumberU64())
	}
	tracer.Send(startTime, block.Number(), block.Hash())

	////////////////////////////////////////////////////////////
	return status, nil
}
```

### Build the project:

```shell
make geth
```

Export Mamoru Environment Variables

```shell
export MAMORU_CHAIN_TYPE=<chain-type>
export MAMORU_SNIFFER_ENABLE=true
export MAMORU_PRIVATE_KEY=<private-key>
export MAMORU_CHAIN_ID=<validations-chain-id>
export MAMORU_ENDPOINT="https://validation-chain-url"
```


Run the following command to check if it works

```shell
 ./build/bin/geth --syncmode light --goerli
```

It's really easy, enjoy