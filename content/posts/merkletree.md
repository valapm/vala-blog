---
title: "Scaling scripts with Merkle Trees"
date: 2022-11-01T10:00:00+03:00
draft: false
tags: ["sCrypt"]
cover:
  image: "/tree.jpg"
  alt: "tree"
  relative: false
  hidden: false
---

A while back [this article](https://xiaohuiliu.medium.com/scalable-state-storage-in-bsv-smart-contracts-60f9aeb3b1f) introduced the concept of minizing on-chain state through the usage of merkle tree validation in Bitcoin Script. At Vala, we use a version of this and now published an independent [library](https://github.com/valapm/bsv-merkletree) for easy usage, validation and updating of merkle trees in Script.

Each prediction market in Vala has to track a ledger of balances in its current output. Without a mechanism to hide the ledger, this would lead to uncontrollable utxo growth.

Instead, the ledger is hashed into a merkle tree and only the merkle root is contained in each utxo. If a trader wants to buy or sell shares, he has to provide a merkle proof to the script, proving his current balance. Similarly, if a new trader wants to add a new balance to the ledger, he has to provide the merkle proof for the last entry in the ledger to the script. From there, the new merkle root can be calculated.

This also means that to generate the entire ledger of balances, one has to go through all transactions of the market, looking at the changes in each input.

Here are some examples:

Let's say we have 3 entries in our ledger.

```ts
import { getMerkleRoot, sha256d } from "bsv-merkletree";

const entries = ["01", "02", "03"];

const leafs = entries.map(sha256d);

const merkleRoot = getMerkleRoot(leafs);
```

Now let's add another entry to it.

```ts
import { getMerklePath, addLeaf } from "bsv-merkletree";

const lastMerklePath = getMerklePath(2, leafs);

const newMerkleRoot = addLeaf(
  "03", // last entry in the ledger
  lastMerklePath,
  merkleRoot,
  "04" // Our new entry
);
```

To validate our update in Script, we have to provide the merkle proof/path and the new merkle root to it.

```sCrypt
import "./merkleTree.scrypt";

bytes newCalculatedMerkleRoot = MerkleTree.addLeaf(b'03', lastMerklePath, merkleRoot, b'04');

require(newCalculatedMerkleRoot == newMerkleRoot);
```

## The library

The full library is open sourced on Github and contains Typescript and [sCrypt](https://scrypt.io/) implementations.

https://github.com/valapm/bsv-fixmath

## A critical vulnerability (Now fixed)

Recently, [NikamotoSV](https://twitter.com/NikamotoS) discovered a [critical vulnerability](https://nikamotosv.substack.com/p/your-type-is-not-type) in the merkle tree validation logic. The exploit even allowed for extraction of all funds out of the markets! The way it worked was smart: When providing a new entry for the ledger, it was possible to provide data that could be interpreted as a new branch in the merkle tree. That way one could spend fake shares by generating a merkle proof for the fake branch.

No money was lost and the exploit is fixed in market version 0.6.5.
