# The Mini Chain
Mini Chain is a small proof of work blockchain. In its current state, it lacks both block verification and chain selection algorithms.

## Architecture 
The basic architecture of MiniChain is defined in three traits: `Proposer`, `Miner`, and `Verifier`.

### `Proposer`
The proposer reads transactions from the mempool and proposes blocks.

```rust
#[async_trait]
pub trait Proposer {

    // Builds a block from the mempool
    async fn build_block(&self) -> Result<Block, String>;

    // Sends a block to the network for mining
    async fn send_proposed_block(&self, block : Block)->Result<(), String>;

    // Propose a block to the network and return the block proposed
    async fn propose_next_block(&self) -> Result<Block, String> {
        let block = self.build_block().await?;
        self.send_proposed_block(block.clone()).await?;
        Ok(block)
    }

}
```

### `Miner`
The miner receives proposed blocks and submits them for verification.
```rust
#[async_trait]
pub trait Miner {

    // Receives a block over the network and attempts to mine
    async fn receive_proposed_block(&self) -> Result<Block, String>;

    // Sends a block to the network for validation
    async fn send_mined_block(&self, block : Block) -> Result<(), String>;

    // Mines a block and sends it to the network
    async fn mine_next_block(&self) -> Result<Block, String> {
        let block = self.receive_proposed_block().await?;
        // default implementation does no mining
        self.send_mined_block(block.clone()).await?;
        Ok(block)
    }

}
```

### `Verifier`
The verifier receives mined blocks, verifiers them, and updates its representation of the chain.
```rust
#[async_trait]
pub trait Verifier {

    // Receives a block over the network and attempts to verify
    async fn receive_mined_block(&self) -> Result<Block, String>;

    // Verifies a block
    async fn verify_block(&self, block: &Block) -> Result<bool, String>;

    // Synchronizes the chain
    async fn synchronize_chain(&self, block: Block) -> Result<(), String>;

    // Verifies the next block
    async fn verify_next_block(&self) -> Result<(), String> {
        let block = self.receive_mined_block().await?;
        let valid = self.verify_block(&block).await?;
        if valid {
            self.synchronize_chain(block).await?;
        }
        Ok(())
    }

}
```

### `FullNode`
A `FullNode` combines the `Proposer`, `Miner`, `Verifier`, and more to provide a complete utility for running the blockchain.
```rust
#[async_trait]
pub trait FullNode 
    : Initializer + 
    ProposerController + MinerController + VerifierController + 
    MempoolController + MempoolServiceController +
    BlockchainController + Synchronizer + SynchronizerSerivce {

    /// Runs the full node loop
    async fn run_full_node(&self) -> Result<(), String> {
        self.initialize().await?;
        loop {
            let error = try_join!(
                self.run_proposer(),
                self.run_miner(),
                self.run_verifier(),
                self.run_mempool(),
                self.run_blockchain(),
                self.run_mempool_service(),
                self.run_synchronizer_service()
            ).expect_err("One of the controllers stopped without error.");

            println!("Error: {}", error);
        }

    }

}
```

## Helpful Code
Generally speaking, you'll find the following useful

### `BlockchainOperations::pretty_print_tree(&self)`
This will show you a marked-up version of the block tree. It annotates the main chain, the chain tip, leaves, block height, etc. It will produce output like the below.
```
Full Node #1
└── (0) Block: f1534392279bddbf 
    └── (1) Block: 001c9f15b438d405 
        ├── (2) Block: 00b53870fa564e32 < TIP
        ├── (2) Block: 002fcec250aa0e43
        ├── (2) Block: 000815e5bd5161bf
        └── (2) Block: 00bf6f30ed06136f
```

### Tuning `fast_chain` default for `BlockchainMetadata`
`BlockchainMetadata::fast_chain` is supposed to contain a set of parameters for the chain that enable it to be run by several full nodes on the same device. Tuning these parameters will help you produce different outcomes and ideally pass the simulation tests.
```rust
impl BlockchainMetadata {
    pub fn fast_chain() -> Self {
        BlockchainMetadata {
            query_slots: 4,
            slot_secs: 2,
            fork_resolution_slots : 2,
            block_size: 128,
            maximum_proposer_time: 1,
            mempool_reentrancy_secs: 2,
            transaction_expiry_secs: 8,
            difficulty: 2,
            proposer_wait_ms : 50
        }
    }
}
```

### The tests
Generally speaking, the tests which are not simulations are quite useful. Most of them should not be affected by your work and serve more as indications of how various systems work. However, the tests in `mini_chain/chain.rs` will likely be very relevant to confirming the desired behavior.
