

```
  BIP: ?
  Layer: API/RPC
  Title: Optional Built-in Mining for Full Nodes
  Authors: Barya <baryalangpo@proton.me>
  Status: Draft
  Type: Specification
  Assigned: ?
  License: BSD-2-Clause
  Version: 0.1.0
```

## Abstract

This BIP specifies an optional, off-by-default mining facility for full nodes: the `setgenerate` and `gethashespersec` RPCs, extra fields in `getmininginfo`, and the `-gen`, `-genproclimit` and `-miningaddress` options. It restores, with an explicit payout address, the built-in CPU miner that Bitcoin Core removed in v0.13 (2016). It makes no consensus or relay-policy change. Its purpose is to put block production back within reach of every node, so that a node can build blocks from its own mempool and policy, and, in later work, contribute that work to pools such as DATUM without delegating template construction.

## Motivation

The Bitcoin white paper does not separate "nodes" from "miners". Its nodes run the network and do the proof-of-work:

- "Proof-of-work is essentially one-CPU-one-vote." (Section 4\)  
- "Each node works on finding a difficult proof-of-work for its block." (Section 5\)  
- "This adds an incentive for nodes to support the network" (Section 6\)  
- "As long as a majority of CPU power is controlled by nodes that are not cooperating to attack the network, they'll generate the longest chain and outpace attackers." (Abstract)

Over time the roles split. The relevant dates are:

| Date | Event |
| :---- | :---- |
| November 2010 | The first public mining pool (Slush Pool) launches. |
| Late 2012 | Stratum (v1) extends the pool protocol as a replacement for getwork. It was never a BIP. |
| March 14, 2016 | Bitcoin Core merges PR \#7507, "Remove internal miner". |
| August 23, 2016 | Bitcoin Core 0.13.0 ships without `setgenerate`, `-gen` and `-genproclimit`. The release notes say "CPU mining has been useless for a long time". |
| September 29, 2024 | OCEAN announces DATUM, in which miners build templates on their own nodes while a pool only coordinates the reward split. |

Pools predate Stratum v1. Under Stratum v1, however, the pool constructs the work and the miner only hashes it. The author borrows a remark by Chris Guida to name what this drift is an instance of:

> "Centralization is like entropy, right? It just always creeps in. And so you have to always be pushing against it. You know the fight against tyranny never ends. The fight against entropy never ends. The fight against centralization never ends. And so yeah, I think we have to just keep pushing to decentralized development."  
>   
> Chris Guida, video at 54:10, [https\://youtu.be/HblOs3nZLyU?t=3250](https://youtu.be/HblOs3nZLyU?t=3250) (transcribed by the author; to be verified against the recording before submission)

Guida's remark is general. Applying it to Stratum v1 and to the removal of node mining is this author's reading, not a claim about what he meant.

DATUM answers the first part of that drift at the miner level. This BIP addresses the second part at the node level:

Mining pools (Stratum v1) \-\> Miners (DATUM) \-\> Nodes (built-in miner, DATUM-capable)

## Specification

All behavior below is optional and non-consensus. A node implementing it MUST NOT change how blocks or transactions are validated, relayed or accepted.

### Options

- `-gen` (boolean, default `0`): start the built-in miner at startup.  
- `-genproclimit=<n>` (default `1`): number of mining threads. `-1` means one thread per hardware thread.  
- `-miningaddress=<addr>`: address that receives the coinbase reward.

If `-gen` is set without a valid `-miningaddress`, the node MUST fail to start with an error. The miner MUST be off unless `-gen` or `setgenerate true` enables it.

### RPC: `setgenerate`

`setgenerate generate [genproclimit] [address]`

- `generate` (boolean, required): `true` starts mining, `false` stops it. Calling with `true` while running restarts it with the new arguments.  
- `genproclimit` (integer, optional, default `1`): as for `-genproclimit`. Values below `-1` MUST be rejected.  
- `address` (string, optional, default `-miningaddress`): where the reward is paid. If neither is available the call MUST fail. An invalid address MUST be rejected.  
- Returns `null`. After `setgenerate false` returns, no further blocks may be found by the built-in miner.

### RPC: `gethashespersec`

Returns the built-in miner's recent hash rate in hashes per second, `0` when stopped.

### RPC: `getmininginfo`

Adds `generate` (boolean), `genproclimit` (number, `0` when stopped) and `hashespersec` (number).

### Behavior

- The miner MUST build blocks with the same block assembly and block policy settings as `getblocktemplate`, using the node's own mempool, and MUST submit solved blocks through the node's normal block validation path.  
- The coinbase MUST pay the configured address. The miner MUST NOT depend on a wallet.  
- Threads SHOULD scan disjoint nonce ranges and SHOULD rebuild work when the chain tip changes or the template becomes stale.  
- On chains that are not exclusively for testing, the miner MUST NOT mine during initial block download.  
- When started on a chain that is not exclusively for testing, the node SHOULD log a warning that CPU hash rate is negligible there.

### Pool participation (non-normative)

The facility above mines solo: any block it finds pays the configured address in full. Contributing work to a pool, for example through DATUM, is out of scope here and would need its own specification. In DATUM, as described in the DATUM Gateway documentation, the pool supplies payout outputs that the gateway places in the coinbase of templates built on the miner's own node, and the node reserves block weight for that transaction (`blockmaxsize` and `blockmaxweight` of 3985000).

## Rationale

- **Return to the white paper's model.** The white paper gives every node a part in producing blocks. A node that can build and attempt blocks keeps that role available, even when most hash rate comes from specialized hardware.  
- **Template sovereignty.** Node-built templates keep transaction selection with the node operator. This is what DATUM provides for miners, and the same property lets a node's work stay tied to its own policy.  
- **Explicit address.** The original `setgenerate` drew its reward key from the wallet. Requiring an address removes the wallet dependency and makes payouts explicit.  
- **Non-consensus.** Because nothing is validated differently, the facility can be adopted, ignored or removed by any implementation without a fork.  
- **Why a BIP.** A common name and behavior for these RPCs and options helps scripts, documentation and tests work across implementations.

### Limitations and objections

- CPU hash rate on mainnet is negligible, and the 0.13.0 release notes removed the miner for that reason. This BIP does not claim to change mining economics. Its direct uses today are regtest, signet, testnet and experiments.  
- Solo mining on a CPU will almost never find a block. Meaningful reward sharing needs pool participation, which this BIP leaves to later work.  
- The author's hypothesis is that nodes with a stake in block production, paid by a distributed payout scheme such as OCEAN's TIDES through DATUM, would blur the line between miner-activated and user-activated soft forks into a single "node-activated" model. This is speculative. This BIP does not specify it and does not depend on it.

## Backward Compatibility

No consensus or network protocol change. The facility reuses the names `setgenerate`, `-gen` and `-genproclimit` that were removed in Bitcoin Core 0.13.0. Differences from the removed behavior:

- It requires a payout address, from the argument or `-miningaddress`.  
- It is off by default.  
- Scripts that relied on `setgenerate` in 0.12 and earlier need to supply an address.

On regtest, the `generate`, `generatetoaddress` and related RPCs are unaffected.

## Reference Implementation

- Bitcoin Knots: built-in CPU miner, [https\://github.com/knots-dev/bitcoin/pull/2](https://github.com/knots-dev/bitcoin/pull/2), including the functional test `test/functional/mining_setgenerate.py`.  
- Companion defaults for running behind the DATUM Gateway: [https\://github.com/knots-dev/bitcoin/pull/3](https://github.com/knots-dev/bitcoin/pull/3).

## Copyright

This BIP is licensed under the BSD 2-clause license.