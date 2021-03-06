# Grove: Merkle trees of zkopru

Zkopru's grove consists of UTXO trees, nullifier tree, and withdrawal trees.

UTXO trees are append-only usage Merkle trees containing UTXOs. Users can spend the UTXOs as the inflow of the transaction by submitting the inclusion Merkle proof. And the output results of transactions are appended back into the latest UTXO tree.

Also, if the zk-transaction creates withdrawal outputs, Zkopru appends them into the latest withdrawal tree. Once the tree's root is marked as finalized, the owner can withdraw assets by proving the ownership.

Afterward, by the commitment-nullifier scheme, the nullifier of the spent UTXOs are marked as used in the nullifier tree, a unique sparse Merkle tree. If a transaction tries to use an already nullified leaf, it becomes invalidated, and the block proposer gets slashed by the challenge system.

![grove](https://docs.google.com/drawings/d/e/2PACX-1vTXH2aQr0HZmWZbbA2ENOISW5hT5XsG2dda90RBSaQH-2ClqlZvFKlSTppR38b2ixUbqQeMOTztC-LA/pub?w=1031&h=564)

## Specification of Merkle trees

|  | UTXO Tree | Nullifier Tree | Withdrawal Tree |
| :--- | :--- | :--- | :--- |
| Type | Sparse Merkle Tree | Sparse Merkle Tree | Sparse Merkle Tree |
| Depth | 31 | 256 | 31 |
| Hash | Poseidon | Keccak256 | Keccak256 |
| How to update | append-only with subtree rollup with 5 depth tree | SMT rollup | append-only with subtree rollup with 5 depth tree |
| Cost \(gas/leaf\) | 180k | 351k | 5.2k |

{% hint style="warning" %}
UTXO tree & withdrawal tree will have 64 depth in the Burrito version. [https://github.com/zkopru-network/zkopru/issues/35](https://github.com/zkopru-network/zkopru/issues/35)
{% endhint %}

## How to manage the UTXO Trees

A single UTXO tree is a sparse Merkle tree for the membership proof. It uses Poseidon hash, one of the cheapest hash function inside the SNARK, to generate a zk SNARK proof to hide the spending hash and its path.

To append new leaves to the UTXO tree, the coordinator performs the following steps. 1. Prepare an array. 2. Coordinator picks MassDeposits to include, and append every deposit in the MassDeposits into the array. 3. L2 transactions generate new UTXOs. Append newly generated UTXOs to the array. 4. Split the prepared array with the chunk size 32. 5. Construct subtrees and perform the sub-tree rollup.

Suppose the UTXO tree is fully filled with \(2^31\) items, the system archive the filled tree and start a new tree. The archived trees are also allowed to be used to reference the inclusion proofs of the transactions.

Zkopru optimistically updates the trees' root and verifies when only there exists a challenge. For the challenge, it generates an on-chain fraud-proof using the subtree rollup methodology. Subtree rollup appends a fixed size of subtrees instead of appending items one by one. When if the subtree's depth is 5, it will append 32 items at once. If it contains only 18 items, the remaining 14 items will have remained forever as zeroes. This subtree rollup dramatically reduces the gas cost about 20 times compared to the rollup. To check the source code, please go to [contracts/controllers/challenges/RollUpChallenge.sol](https://github.com/zkopru-network/zkopru/blob/master/packages/contracts/contracts/controllers/challenges/RollUpChallenge.sol). And to see how the subtree works please go to [packages/contracts/contracts/libraries/Tree.sol](https://github.com/zkopru-network/zkopru/blob/9a2e7fe361bdba7be7241d9f80f9e9afbf8ae76c/packages/contracts/contracts/libraries/Tree.sol#L275)

![subtree-rollup](https://docs.google.com/drawings/d/e/2PACX-1vRoBO2Nl-M8elgz55jzgxuux1f6FbrZIiUCeY3Z9Sf7haKF8kSP548L8pXcdh_VCfT_gcW1gywY4VKY/pub?w=959&h=417)

## Nullifier Tree

Every transfer, withdrawal, and migration transaction spends UTXOs with the inclusion proofs and marks the derived nullifiers used on the nullifier tree. Thus, the nullifier tree is a very big sparse Merkle tree that records every spent UTXOs in the 254-depth of Sparse Merkle Tree. Therefore, Zkopru uses keccak256, the cheapest hash function, for the nullifier tree's hash function.

To update the nullifier tree, the coordinator performs the following steps.

1. Pick transactions \(transfer, withdrawal, migration\) and collect all nullifiers from the transactions.
2. Check there does not exist any already used nullifiers.
3. Mark every nullifier as used. During the update process, if any of the nullifiers does not change the nullifier tree root, discard the transaction because it tries a double-spending.

Just like the UTXO tree, Zkopru optimistically updates the nullifier tree's root. If there is any problem, we can prove a nullifier used more than twice by generating a fraud-proof on-chain. To see how it works, please see [RollUpChallenge.sol](https://github.com/zkopru-network/zkopru/blob/master/packages/contracts/contracts/controllers/challenges/RollUpChallenge.sol) and [SMT.sol](https://github.com/zkopru-network/zkopru/blob/master/packages/contracts/contracts/libraries/SMT.sol).

## Withdrawal Tree

The only difference with the withdrawal tree and UTXO tree is that the withdrawal tree uses keccak256 for the hash function. The reason why it uses keccak256 is that Zkopru needs the withdrawal tree's Merkle proof on the smart contract while it needs UTXO tree's Merkle proof inside the SNARK circuit. The leaves in the withdrawal trees become withdrawable on the layer-1 smart contract after its tree's root becomes finalized.

To update the withdrawal tree, the coordinator performs the following steps.

1. Collect every withdrawal leaf of the picked transactions.
2. Split the collected withdrawal array with the chunk size 32.
3. Construct subtrees and perform the sub-tree rollup.

