---
date: 2026-05-09
tags: [evm, storage, keccak256, merkle-patricia-trie, dos, eip-2929]
status: draft
---

# Why doesn't `mapping` slot prediction break Ethereum?

I was reading through a `Bank` contract line by line and got stuck
on the very first state variable:

```solidity
mapping(address => uint256) public balances;
```

It looks innocent. But once you ask *where does this actually live
in storage*, you end up pulling on a thread that goes through
keccak256, the Merkle Patricia Trie, and a real DoS attack from 2016.
This note is me trying to write down what I figured out.

## What I thought I knew

A contract has 2^256 storage slots, each holding a 32-byte word.
Simple state variables get assigned slots starting from 0 — `balances`
in the contract above would be at slot 0. Easy.

But `balances` is a mapping. It's supposed to hold a value for
*every possible address* — there are 2^160 of them. You obviously
can't pre-allocate 2^160 slots. So how does it actually work?

## The answer: hash the key with the slot number

For `mapping(address => uint256)` declared at slot `p`, the value
for key `k` lives at:
storage_slot = keccak256(k . p)
where `.` is byte concatenation (with both padded to 32 bytes).

So `balances[0xAlice]` doesn't sit at slot 0 or slot 1 or anywhere
near `balances` itself. It sits at some pseudo-random 256-bit slot
determined by hashing Alice's address together with the number 0.

This was my first "wait, what?" moment. The mapping declaration at
slot 0 doesn't *contain* anything. It's just a label that tells the
compiler what number to hash with the key.

## First worry: can someone predict slots and mess with them?

If `keccak256` is deterministic and `p` is known (slot 0 is public
information for any contract), then anyone can compute exactly which
slot holds `balances[0xAlice]`. That sounds like an exposure.

But it isn't, and the reason matters:

- **Predicting a slot ≠ being able to write to it.** Storage writes
  in the EVM go through the contract's own code. Even if I know
  exactly which slot stores Alice's balance, I can't write there
  unless the contract's code lets me. Slot location is public,
  authorization is enforced by the bytecode.
- **Reading is fine.** Anyone can read any storage slot of any
  contract via `eth_getStorageAt`. The privacy of mapping data was
  never a security guarantee — it just feels private because the
  slot location is non-obvious without computing the hash.

So the slot prediction itself isn't a problem. Good.

## Second worry: what if I want to *find* a key that lands on a specific slot?

This is the more interesting direction. Can I find an address `k`
such that `keccak256(k . p)` equals some slot I want to target?

This would require finding a **preimage** of keccak256 — given a
desired output, find an input that hashes to it. Keccak256 is
designed to be preimage-resistant: there's no known method better
than brute force, and brute force on a 256-bit output is
computationally infeasible (~2^256 work for a full preimage,
~2^128 for collisions).

So no, you can't aim hash outputs at chosen slots.

## But wait — keccak256 isn't actually random

Here's where I started suspecting something. `keccak256` is
**deterministic**, not random. It's a specific function. The output
distribution looks random in aggregate, but every input maps to
exactly one fixed output. That's a strong property, not a weak one.

But it does mean: *if I'm willing to grind through a lot of inputs,
I can find inputs whose outputs share a chosen prefix.* I can't hit
a specific 256-bit slot, but I might be able to find many keys that
all hash to slots starting with `0xdeadbeef...`.

Why would I want that? Because EVM storage isn't just a flat array
of 2^256 slots. It's stored as a **Merkle Patricia Trie** (MPT) on
actual disk (LevelDB or RocksDB underneath). And tries care about
prefixes.

## Quick detour: what's the Merkle Patricia Trie doing here?

The 2^256 storage space is sparse — almost all slots are zero. You
can't store 2^256 entries on disk. So Ethereum stores only the
non-zero slots, organized into an MPT keyed by slot number.

The trie:

- Branches on hex nibbles of the key
- Lets clients verify a single slot's value with a Merkle proof
  (log-depth in the number of stored keys)
- Stays roughly balanced *because keccak256 outputs are uniformly
  distributed* — random-looking keys mean random-looking trie paths

That last point is what I want to push on. The trie's performance
guarantees rest on an **assumption about the distribution of keys**,
not on a structural property of the trie itself.

## So: could an attacker force trie imbalance?

In theory, yes. If you grind enough inputs to find many keys that
share a long prefix in their keccak256 outputs, all those keys will
end up in the same deep branch of the trie. Operations on that
branch become more expensive — more nodes to traverse, more disk
reads, more hashing.

In practice this is hard:

- Finding a k-bit prefix collision takes roughly 2^k work
- Even modest prefix forcing (say 32-40 bits) is expensive
- And the payoff is just *slowing things down*, not stealing money

But "in theory + the right gas pricing mistake" is exactly what
happened in 2016.

## The 2016 Shanghai DoS attacks

In September–October 2016, Ethereum was hit by a series of DoS
attacks during the Shanghai DevCon. The core issue wasn't that
keccak256 was broken — it was that **storage-touching opcodes were
underpriced relative to their actual cost on a real node's disk**.

Attackers crafted transactions that did a lot of cheap-looking
operations (mostly `EXTCODESIZE`, `SUICIDE` of empty accounts,
storage reads against many addresses) which forced nodes to do
huge amounts of trie traversal and disk I/O. Block processing
slowed dramatically. Some clients fell out of sync.

The response was a sequence of EIPs:

- **EIP-150** (Tangerine Whistle, Oct 2016) — repriced opcodes that
  touched state, especially `EXTCODESIZE` and friends
- **EIP-158** (Spurious Dragon, Nov 2016) — cleared out empty
  accounts that had been spammed into existence
- **EIP-2929** (Berlin, 2021) — introduced the **cold/warm access**
  distinction. First access to an address or storage slot in a
  transaction is expensive ("cold"); subsequent accesses are cheap
  ("warm"). This properly prices the cost of *bringing data into
  cache from disk* vs. *reading from cache*.
- **EIP-3529** (London, 2021) — reduced refunds that had been
  exploitable via gas-token tricks

What I take away from this: **gas pricing is not just an economic
mechanism, it's a security mechanism**. When the price of an
operation diverges from its real cost on validator hardware, you
get a DoS surface. The whole cold/warm framework exists because
of this lesson.

## What about keccak256 specifically?

The Shanghai attacks weren't really keccak256 attacks — they were
gas mispricing attacks. Targeted prefix collisions to force trie
imbalance is a *more refined* version of the same idea, but as far
as I know it has never been exploited at scale. Why not?

My guess (worth checking):

- The economic logic doesn't work. Breaking keccak256 prefixes
  requires real compute, and the payoff is "slow down the network
  a bit." A nation-state could fund it ideologically, but a profit-
  seeking attacker has much better targets (reentrancy, oracle
  manipulation, etc.).
- Modern gas pricing (EIP-2929) makes trie traversal cost more
  proportional to actual work. The exploit window narrowed.
- Clients have gotten much better at caching and trie compaction.

So in some sense the answer to *"why doesn't mapping slot
prediction break Ethereum?"* is: it's predictable, but predictability
doesn't give you write access; collision-finding doesn't give you
useful targets; and the only meaningful attack (forcing imbalance
to amplify gas mispricing) is now expensive both to mount and to
profit from.

## What I'm still unsure about

- I should check whether `keccak256(k . p)` is the exact encoding
  for all mapping types, or if address keys get special treatment.
  I've seen `abi.encodePacked` mentioned in some sources.
- The relationship between MPT depth and actual gas cost post-2929
  — I assume cold/warm is the main lever now, but I'd like to
  trace it through a concrete example.
- Whether stateless Ethereum / Verkle trees change any of this
  reasoning. They use a different commitment structure, and I
  haven't read enough yet.

## Sources I used

- Solidity docs: Layout of State Variables in Storage
- EIP-150, EIP-158, EIP-2929, EIP-3529
- https://learnevm.com/chapters/evm/overview
