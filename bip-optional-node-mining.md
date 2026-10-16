```
  BIP: ?
  Layer: API/RPC
  Title: Optional Built-in Mining for Full Nodes
  Authors: Salenga <author-email@example.com>
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

- "Proof-of-work is essentially one-CPU-one-vote." (Section 4)
- "Each node works on finding a difficult proof-of-work for its block." (Section 5)
- "This adds an incentive for nodes to support the network" (Section 6)
- "As long as a majority of CPU power is controlled by nodes that are not cooperating to attack the network, they'll generate the longest chain and outpace attackers." (Abstract)

Over time the roles split. The relevant dates are:

| Date | Event |
|---|---|
| November 2010 | The first public mining pool (Slush Pool) launches. |
| Late 2012 | Stratum (v1) extends the pool protocol as a replacement for getwork. It was never a BIP. |
| March 14, 2016 | Bitcoin Core merges PR #7507, "Remove internal miner". |
| August 23, 2016 | Bitcoin Core 0.13.0 ships without `setgenerate`, `-gen` and `-genproclimit`. The release notes say "CPU mining has been useless for a long time". |
| September 29, 2024 | OCEAN announces DATUM, in which miners build templates on their own nodes while a pool only coordinates the reward split. |

Pools predate Stratum v1. Under Stratum v1, however, the pool constructs the work and the miner only hashes it. The author borrows a remark by Chris Guida to name what this drift is an instance of:

> "Centralization is like entropy, right? It just always creeps in. And so you have to always be pushing against it. You know the fight against tyranny never ends. The fight against entropy never ends. The fight against centralization never ends. And so yeah, I think we have to just keep pushing to decentralized development."
>
> Chris Guida, video at 54:10, <https://youtu.be/HblOs3nZLyU?t=3250> (transcribed by the author; to be verified against the recording before submission)

Guida's remark is general. Applying it to Stratum v1 and to the removal of node mining is this author's reading, not a claim about what he meant.

DATUM answers the first part of that drift at the miner level. This BIP addresses the second part at the node level:

Mining pools (Stratum v1) -> Miners (DATUM) -> Nodes (built-in miner, DATUM-capable)

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

### Objections and responses

Scope note: the facility specified above mines solo. The responses below concern the pool-participation mode described as out of scope, in which a node's work is credited by a pool using a distributed reward scheme such as OCEAN's TIDES through DATUM. They are arguments about whether that later work is worth doing. They do not describe behavior this BIP specifies.

What OCEAN's TIDES documentation (<https://ocean.xyz/docs/tides>, read October 2026) says, and which bears on every response below:

- Each accepted proof is credited as shares equal to its target difficulty, and the block reward is split by each miner's share of a window of eight times the found block's difficulty worth of shares. Rewards are rounded down to the nearest satoshi.
- "TIDES itself has no concept of a minimum payout itself." Pools may set their own minimum thresholds, and the documentation says there is generally one.
- "Generally a tiny payout isn't worthwhile." If a miner has not earned at least one full satoshi when a block is found, "those earnings are lost."
- Miners "should be paid via the generation transaction", that is, directly in the coinbase.
- The documentation states that this "does effectively price out miners that are simply not effective at mining, such as CPU miners."

**Objection 1: CPU hash rate on mainnet is negligible, and the 0.13.0 release notes removed the miner for that reason.**

Response, in part: The objection is largely correct and this BIP concedes it for mainnet today. Under TIDES a node's expected reward per block is roughly its fraction of the work in the window times the block reward. For a CPU against the whole network that fraction is many orders of magnitude below one satoshi, so by TIDES's own rule rounding down gives a payout of zero, and the documentation names CPU miners as priced out. The argument that a distributed scheme turns "zero" into "small but non-zero" therefore does not hold on mainnet under TIDES as documented.

What survives is narrower. TIDES removes the requirement to find a block, so the objection is no longer about luck, only about size. The facility remains useful on regtest, signet and testnet, and for experiments with a node that builds its own templates. If a pool, a payout scheme or a future arrangement credited sub-satoshi contributions, for example by accruing them across blocks, the economics would change. No such arrangement exists today, and TIDES does not provide one. This BIP proposes that they be changed to provide one; see "Proposals to pools and payout schemes" below.

**Objection 2: Solo mining on a CPU will almost never find a block.**

Response, in part: Correct for solo mining. In a share-based scheme the probability of finding a block is not what determines reward, so this objection loses its force there. It is replaced by the objection of Objection 1: a CPU's credited work is too small to produce a whole satoshi in a block's payout. This BIP does not promise any payout amount, and it does not claim that finding a block "does not matter" for a node whose contribution rounds to zero.

**Objection 3 (the author's hypothesis): nodes with a stake in block production would blur the line between miner-activated and user-activated soft forks.**

Response: Under the white paper's model, nodes are the participants in proof of work. Since then full nodes have narrowed to validation and relay while pools and specialized hardware carry production. Restoring a built-in miner puts block assembly and submission back within every node's reach, even when its hash rate is negligible, and DATUM-style node-built templates keep transaction selection with the operator. That is the emphasis this BIP defends, and it does not rest on payouts. The stronger claim, that a TIDES payout would give every node an incentive and so merge miner-activated and user-activated soft forks into a "node-activated" model, is speculative and is not supported by TIDES as documented, because tiny contributions earn nothing there. A CPU's hash rate also carries almost no weight in miner signaling. This BIP neither specifies nor depends on that outcome, and reviewers are invited to challenge it.

### Proposals to pools and payout schemes

The conclusion above, that a CPU node earns nothing under TIDES as documented, follows from rules that pools and payout schemes chose and can change. This BIP proposes that pool operators, DATUM's maintainers and OCEAN consider changes so that any node whose work is credited is paid, or has the credit recorded, however small the amount. The aim is adoption: a node that is credited something, rather than nothing, has a reason to take part. These are suggestions to other projects. This BIP does not specify them, cannot require them, and none of them is implemented in the reference implementation.

1. **Pool payout rules.** Pools can set their minimum payout threshold for node-built-template miners at a level low enough not to exclude them, or can accrue earnings below the threshold across blocks in an account until the threshold is reached, instead of discarding them.
2. **DATUM.** DATUM's documentation could describe a recommended way for a pool to credit very low hash-rate gateways, and the gateway could report to the operator how much credited work is awaiting payout.
3. **Lightning or other off-chain settlement.** Where a coinbase output below the dust limit cannot be paid on chain, accrued amounts could be settled over Lightning or batched into a later on-chain payment. Pool-side accrual sits in tension with TIDES's stated preference that miners "be paid via the generation transaction", since an accrued balance is held by the pool until paid.
4. **TIDES itself.** TIDES rounds each reward down to the nearest satoshi and says earnings below one satoshi are lost. A variant could carry the fractional remainder forward per miner, for example in millisatoshis or as an accrued balance, so that small contributions add up across blocks. Whether that can be done without weakening TIDES's properties, and what the accounting costs the pool, is for OCEAN to evaluate.

Costs and risks that proponents should expect to address:

- Custody: accrual means a pool holds funds owed to many tiny accounts, which brings back the trust and operational risk that paying directly in the coinbase avoids.
- Coinbase space and dust: more payees, or payees below the dust limit, make the generation transaction larger or non-standard, and DATUM reserves a fixed block weight for it.
- Abuse: if every credited node is paid regardless of size, an operator could create very many small identities to collect rounding benefits or to burden the pool. Rules on identity, minimum credited work for accrual to begin, or pool fees may be needed.
- Cost to the pool: tracking and settling very small balances costs the operator more than they are worth, so pools may reasonably decline.
- Economics: even with accrual, a CPU's expected earnings stay tiny against the electricity it uses. The incentive proposed is the principle of being credited, not income.

This BIP's specification and reference implementation stand without any of these changes. If none is adopted, Objections 1 and 2 stand as conceded above.

## Backward Compatibility

No consensus or network protocol change. The facility reuses the names `setgenerate`, `-gen` and `-genproclimit` that were removed in Bitcoin Core 0.13.0. Differences from the removed behavior:

- It requires a payout address, from the argument or `-miningaddress`.
- It is off by default.
- Scripts that relied on `setgenerate` in 0.12 and earlier need to supply an address.

On regtest, the `generate`, `generatetoaddress` and related RPCs are unaffected.

## Reference Implementation

- Bitcoin Knots: built-in CPU miner, <https://github.com/knots-dev/bitcoin/pull/2>, including the functional test `test/functional/mining_setgenerate.py`.
- Companion defaults for running behind the DATUM Gateway: <https://github.com/knots-dev/bitcoin/pull/3>.

## Copyright

This BIP is licensed under the BSD 2-clause license.
