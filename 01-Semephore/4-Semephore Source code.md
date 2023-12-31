
### Semephore Source code

#### packages/Identity

This library provides a class that can be used to create identities **compatible** with the Semaphore [circuits](https://github.com/semaphore-protocol/semaphore/tree/main/circuits). 

Each identity contains two secret values: _trapdoor_ and _nullifier_, and one public value: _commitment_. The Poseidon hash of the secret values is the identity secret, and its hash is the identity commitment.


使用 Semaphore 的每个账户需要创建私钥 `secret` 和公钥 `commitment`

 - `trapdoor`:   陷门, 是个随机数, 在本地浏览器生成和存储, 不存到合约
 - `nullifier`: 无效符, 是个随机数, 在本地浏览器生成和存储, 不存到合约 
 - `secret`:  可看做私钥, 是个随机数, 在本地浏览器生成和存储, 不存到合约
 - `commitment`:  承诺, 身份标识, 类似以太坊公钥, 存储到合约里.

对  `trapdoor`   和  `nullifier`  做 Poseidon Hash 得到 `secret` 
对  `secret` 再做 Poseidon Hash 得到  `commitment` 

`packages/identity/src/identity.ts` : 

```ts
export function genRandomNumber(numberOfBytes = 31): bigint {
    return BigNumber.from(randomBytes(numberOfBytes)).toBigInt()
}

// semaphore/packages/identity/src/identity.ts
export default class Identity {
    private _trapdoor: bigint // 陷门, 是个随机数, 在本地浏览器生成, 不存到合约
    private _nullifier: bigint // 无效符, 是个随机数, 在本地浏览器生成, 不存到合约
    private _secret: bigint  // 私钥, 是个随机数, 在本地浏览器生成, 不存到合约
    private _commitment: bigint // 承诺, 身份标识, 类似以太坊公钥, 存储到合约里.

    constructor(identityOrMessage?: string) {
        if (identityOrMessage === undefined) {
            this._trapdoor = genRandomNumber()
            this._nullifier = genRandomNumber()
            this._secret = poseidon2([this._nullifier, this._trapdoor]) //hash
            this._commitment = poseidon1([this._secret]) //hash

            return
        }

        /// ...
        
        this._trapdoor = BigNumber.from(trapdoor).toBigInt()
        this._nullifier = BigNumber.from(nullifier).toBigInt()
        this._secret = poseidon2([this._nullifier, this._trapdoor])
        this._commitment = poseidon1([this._secret])
    }
```

`hash.sha512(identityOrMessage).padStart(128, "0")` : 
 - 生成了一个 512 位的哈希值（h），它包含了128个十六进制字符。然后通过 `slice` 方法将这个哈希值切分成两部分，每部分为64个字符，对应的位数为64 * 4=256 位
`alt_bn128 is 253.6 bits, so we can safely use 253 bits.`
 - `alt_bn128` 是 Ethereum 使用的一种配对友好的椭圆曲线，在该曲线上进行的运算，所涉及到的数字的位数不能超过这个上限。因此，为了确保得到的 `_trapdoor` 和 `_nullifier` 位数不会超过 253，对生成的 256 位数进行右移操作，移位 3 位，即除以 2 的 3 次方，相当于取前 253 位

#### packages/proof

Proofs are generated `off-chain`, and can be **verified** either `on-chain` or `off-chain`.
##### 1 | generateProof

Source: `packages/proof/src/generateProof.ts`

 - `snarkArtifacts`: SNARK-related artifacts like wasm (WebAssembly binary) and zkey (zk-SNARK encryption key) file paths.

```ts
import { groth16 } from "snarkjs"

export default async function generateProof(
    const { proof, publicSignals } = await groth16.fullProve(
        {
            identityTrapdoor: trapdoor,
            identityNullifier: nullifier,
            treePathIndices: merkleProof.pathIndices,
            treeSiblings: merkleProof.siblings,
            externalNullifier: hash(externalNullifier),
            signalHash: hash(signal)
        },
        snarkArtifacts.wasmFilePath,
        snarkArtifacts.zkeyFilePath
    )
    return {
        merkleTreeRoot: publicSignals[0],
        nullifierHash: publicSignals[1],
        signal: BigNumber.from(signal).toString(),
        externalNullifier: BigNumber.from(externalNullifier).toString(),
        proof: packProof(proof)
    }
```

##### 2 | verifyProof

**main.groth16.vkey.json**:
 - This is the JSON representation of the `Verification Key`.
 - Unlike `.zkey`, this verification key is *public* . It is used to verify proofs generated by the corresponding zkey.
 - Anyone can use this key to **verify proofs**, but not to generate proofs.

```rust
import { groth16 } from "snarkjs"
import verificationKeys from "./verificationKeys.json"

export default function verifyProof(
    { merkleTreeRoot, nullifierHash, externalNullifier, signal, proof }: FullProof,
    treeDepth: number
): Promise<boolean> {
    if (treeDepth < 16 || treeDepth > 32) {
        throw new TypeError("The tree depth must be a number between 16 and 32")
    }

    const verificationKey = {
        ...verificationKeys,
        vk_delta_2: verificationKeys.vk_delta_2[treeDepth - 16],
        IC: verificationKeys.IC[treeDepth - 16]
    }

    return groth16.verify(
        verificationKey,
        [merkleTreeRoot, nullifierHash, hash(signal), hash(externalNullifier)],
        unpackProof(proof)
    )
}
```

##### 3 | calculateNullifierHash

> `identityNullifier` is `nullifier`

```ts
export default function calculateNullifierHash(
    identityNullifier: number | bigint | string,
    externalNullifier: BytesLike | Hexable | number | bigint
): bigint {
    //identityNullifier that is nullifier
    return poseidon2([hash(externalNullifier), identityNullifier])
}
```

##### 4 | index.test.ts

Without `commitment` , you can't be proved in the group.

```TS
beforeAll(async () => { curve = await getCurveFromName("bn128") })
afterAll(async () => { await curve.terminate() })

describe("# generateProof", () => {
	it("Shouldn't generate Semaphore proofs if the identity is not part of the group", async () => {
		const group = new Group(treeDepth)
		group.addMembers([BigInt(1), BigInt(2)])  // no commitment here.

		const fun = () =>
			generateProof(identity, group, externalNullifier, signal, {
				wasmFilePath,
				zkeyFilePath
			})

		await expect(fun).rejects.toThrow("The identity is not part of the group")
	})
```

- without `snark artifacts` (wasmFilePath / zkeyFilePath),  you can't generate a Semaphore proof.
- with `Identity commitment`  &  `snark artifacts` , you can generate a proof.

```ts
	it("Shouldn't gen a Sema proof with default snark artifacts(with Node.js)", async () => {
		const group = new Group(treeDepth)
		group.addMembers([BigInt(1), BigInt(2), identity.commitment])
		// no `{ wasmFilePath, zkeyFilePath }`
		const fun = () => generateProof(identity, group, externalNullifier, signal)
		await expect(fun).rejects.toThrow("ENOENT: no such file or directory")
	})

	it("Should gene a Sema proof passing a group as parameter", async () => {
		const group = new Group(treeDepth)
		group.addMembers([BigInt(1), BigInt(2), identity.commitment])
		fullProof = await generateProof(identity, group, externalNullifier, signal, {
			wasmFilePath,
			zkeyFilePath
		})
		expect(typeof fullProof).toBe("object")
		expect(fullProof.merkleTreeRoot).toBe(group.root.toString())
	}, 20000)
```

The *tree depth* should be between `16` and `32`
```ts
describe("# verifyProof", () => {
	it("Should not verify a proof if the tree depth is wrong", () => {
	    // depth 3 is too low
		const fun = () => verifyProof(fullProof, 3)

		expect(fun).toThrow("The tree depth must be a number between 16 and 32")
	})

	it("Should verify a Semaphore proof", async () => {
		const response = await verifyProof(fullProof, treeDepth)

		expect(response).toBe(true)
	})
})
```


#### packages/circuits

The [circuit](https://semaphore.appliedzkp.org/docs/technical-reference/circuits) structures how the ZKP inputs and outputs are generated, hashed and verified. It has 3 main components:

- **Proof of membership:** An `identity commitment` is generated from the hash of the identity `trapdoor` and identity `nullifier`, then verifies the membership proof against the Merkle root and identity commitment.
	- **Private inputs:**  `treeSiblings[nLevels]`  / `treePathIndices[nLevels]` / `identityNullifier` / `identityTrapdoor`
	- **Public outputs:**  `root`: The Merkle root of the tree.
- **Nullifier hash:** nullifier hashes are saved in a Semaphore *smart contract,* so that the smart contract itself can **reject** a proof with an *already used nullifier hash* . The circuit hashes the identity nullifier and the external nullifier, then checks that it matches the given nullifier hash.
	- **Private inputs:** `identityNullifier`: the 32-byte identity secret used as a nullifier.
	- **Public inputs:**  `externalNullifier`: the 32-byte external nullifier.
	- **Public outputs:**  `nullifierHash`: the hash of the identity nullifier and the external nullifier; used to prevent double-signaling.
- **Signal:** The circuit calculates a dummy square of the signal hash to prevent any tampering with the proof; if the public input changes then verification will fail.
	- **Public inputs:**  `signalHash`: the hash of the user's signal.


`packages/circuits/semaphore.circom`

```rust
// this._secret = poseidon2([this._nullifier, this._trapdoor])
template CalculateSecret() {
    signal input identityNullifier;
    signal input identityTrapdoor;

    signal output out;

    component poseidon = Poseidon(2);

    poseidon.inputs[0] <== identityNullifier;
    poseidon.inputs[1] <== identityTrapdoor;

    out <== poseidon.out;
}
```


```rust
// this._commitment = poseidon1([this._secret])
template CalculateIdentityCommitment() {
    signal input secret;
    signal output out;

    component poseidon = Poseidon(1);
    poseidon.inputs[0] <== secret;
    out <== poseidon.out;
}
```


##### 2 | Nullifier Hash

`Nullifier Hash` 是用来证明某个 Identity 在某个 App 里对应 commitment 存在一个 merkle 树上，并生成的标示
`Nullifier Hash` is used to prove that an Identity exists in a merkle tree corresponding to a commit in an App, and the generated mark.

```rust
template CalculateNullifierHash() {
    signal input externalNullifier;  // 不同应用 app 有不同的 externalNullifier, 能让用户的 identityNullifier 进行不同 app 里的复用
    signal input identityNullifier;  // 用户自己的身份证明
    signal output out;

    component poseidon = Poseidon(2);
    poseidon.inputs[0] <== externalNullifier;
    poseidon.inputs[1] <== identityNullifier;
    out <== poseidon.out;  // 对 externalNullifier 和 identityNullifier 做 hash
}

template Semaphore(nLevels) {
    signal output nullifierHash;
	// ...
	component calculateNullifierHash = CalculateNullifierHash();
    calculateNullifierHash.externalNullifier <== externalNullifier;
    calculateNullifierHash.identityNullifier <== identityNullifier;
    // ...    
    nullifierHash <== calculateNullifierHash.out;
}
component main {public [signalHash, externalNullifier]} = Semaphore(20);
```

相对比  `TornadoCash` 中的一个 `Nullifier` , 额外加入 `externalNullifier` 的原因是，`同一个Identity`，在输入  `externalNullifier`  不同的情况下，能生成不同的 `nullifierHash`。

也就是说，一个账户可以多次“消费”。这样设计的原因是为了 `Signal` 的业务需求

Compared with a `Nullifier` in `TornadoCash`, the reason for adding `externalNullifier` is that `the same Identity` can generate different `nullifierHash` when the input `externalNullifier` is different.

In other words, an account can be "consumed" multiple times. The reason for this design is for the business requirements of `Signal`
##### 3 | MerkleTreeInclusionProof

i.e. Merkle 存在证明

the circuit checks if a given *leaf*, along with its *Merkle path* and *siblings*, correctly hashes up to a specified `Merkle root`, providing **proof of its inclusion in the Merkle tree** without revealing its exact position or the contents of the other leaves.

```rust
template MerkleTreeInclusionProof(nLevels) {
    signal input leaf;
    signal input pathIndices[nLevels];
    signal input siblings[nLevels];
    signal output root;
    // ...
    hashes[0] <== leaf;
    for (var i = 0; i < nLevels; i++) {
        pathIndices[i] * (1 - pathIndices[i]) === 0;
        hashes[i + 1] <== poseidons[i].out;
    }
    root <== hashes[nLevels];
}
```

```rust
// in Semaphore.circom
    component inclusionProof = MerkleTreeInclusionProof(nLevels); // 20
    inclusionProof.leaf <== calculateIdentityCommitment.out;

    for (var i = 0; i < nLevels; i++) {
        inclusionProof.siblings[i] <== treeSiblings[i];
        inclusionProof.pathIndices[i] <== treePathIndices[i];
    }

    root <== inclusionProof.root;
```
##### 4 | template Semaphore

Take note of each of the following constraints:

```rust
// The current Semaphore smart contracts require nLevels <= 32 and nLevels >= 16.
template Semaphore(nLevels) {
    signal input identityNullifier;
    signal input identityTrapdoor;
    signal input treePathIndices[nLevels];
    signal input treeSiblings[nLevels];

    signal input signalHash;
    signal input externalNullifier;

    signal output root;
    signal output nullifierHash;

    component calculateSecret = CalculateSecret();
    calculateSecret.identityNullifier <== identityNullifier;
    calculateSecret.identityTrapdoor <== identityTrapdoor;

    signal secret;
    secret <== calculateSecret.out;

    component calculateIdentityCommitment = CalculateIdentityCommitment();
    calculateIdentityCommitment.secret <== secret;

    component calculateNullifierHash = CalculateNullifierHash();
    calculateNullifierHash.externalNullifier <== externalNullifier;
    calculateNullifierHash.identityNullifier <== identityNullifier;

    component inclusionProof = MerkleTreeInclusionProof(nLevels);
    inclusionProof.leaf <== calculateIdentityCommitment.out;

    for (var i = 0; i < nLevels; i++) {
        inclusionProof.siblings[i] <== treeSiblings[i];
        inclusionProof.pathIndices[i] <== treePathIndices[i];
    }

    root <== inclusionProof.root;

    // Dummy square to prevent tampering signalHash.
    signal signalHashSquared;
    signalHashSquared <== signalHash * signalHash;

    nullifierHash <== calculateNullifierHash.out;
}

component main {public [signalHash, externalNullifier]} = Semaphore(20);
```

#### packages/contracts

These contracts are closely related to the protocol. You can use them in your contract or you can use [**Semaphore.sol**](https://semaphore.appliedzkp.org/docs/technical-reference/contracts#semaphoresol), which integrates them for you.

> While some DApps may use on-chain groups, others may prefer to use off-chain groups, saving only their tree roots in the contract.

Group

##### 1 | Semaphore.sol

Below are the main `functions` and `features` of the file.

**Modifiers:**
- `onlyGroupAdmin`: Ensures the function can only be called by a specific group's admin.
- `onlySupportedMerkleTreeDepth`: Ensures the Merkle tree depth is within a certain range (between 16 and 32 inclusive).

**State Variables:**
- `verifier`: Holds the address of the Semaphore verifier used to verify users' ZK proofs.
- `groups`: A mapping to store group parameters by their IDs.

**Functions:**
- `verifyProof`: It is used to validate a zero-knowledge proof on chain
- Group related:
	- `createGroup`: Allows creation of a new group with details like the group's ID, Merkle tree depth, admin, and optionally, the Merkle tree duration.
	- `updateGroupAdmin`: Lets an admin update the admin of a group.
	- `updateGroupMerkleTreeDuration`: Allows an admin to update the duration of the Merkle tree for a group.
	- `addMember`: Admin can add a new member to a group.
	- `addMembers`: Admin can add multiple members to a group at once.
	- `updateMember`: Admin can update a group member's identity.
	- `removeMember`: Admin can remove a member from a group.

Next, we will focus primarily on `verifyProof`
1. fetches the Merkle tree depth for the `groupId`. If the depth is 0, it reverts `GroupDoesNotExist`
2. fetches the current **Merkle tree** root for the `groupId`.
3. If the input `merkleTreeRoot` is different from the `currentMerkleTreeRoot`, it checks:
	1. If the `merkleTreeRoot` is a part of the group's history. If not, it reverts.
	2. If the `merkleTreeRoot` is expired (i.e., it's older than allowed). If it is expired, it reverts.
	3. otherwise, go on.
4. So, There's a question in Step 3 : even if `merkleTreeRoot != currentMerkleTreeRoot` , we can continue verify.
	1. merkleTreeRoot may have been a valid root in the past. There are several possible reasons for this (某个特定的 merkleTreeRoot 可能是过去的有效根。这有几个可能的原因):
		1. Historical compatibility(历史兼容性): In some cases, especially when using the Merkle proof system, users may use a slightly older Merkle root to create their proofs. If the system only allows the latest Merkle root, then any attestations generated shortly after the root update will become invalid immediately, which could lead to a poor user experience. (在某些情况下，特别是在使用Merkle证明系统的时候，用户可能会使用一个稍微旧的 Merkle根来创建他们的证明。如果系统只允许使用最新的Merkle根，那么在根更新之后不久生成的任何证明都将立即失效，这可能会导致不良的用户体验 )
		2. Data Propagation Delay(数据传播延迟): In a decentralized system, information updates (such as a new Merkle root) may take some time to be seen by all participants on the network. Thus, allowing slightly older Merkle roots provides a fault-tolerance mechanism such that users who may not have seen the latest data can still submit valid proofs. (在去中心化系统中，信息更新（例如新的Merkle根）可能需要一些时间才能被网络上的所有参与者看到。因此，允许稍旧的 Merkle根提供了一种容错机制，使得那些可能还没有看到最新数据的用户仍然可以提交有效的证明。)
		3. Security(安全性): There is also a check in the code to ensure that even if merkleTreeRoot is a valid old root, it cannot outlive its intended age. This ensures that even if old Merkle roots are accepted, they are only valid for a secure, predetermined window of time. (代码中还有一个检查，确保即使 merkleTreeRoot 是一个有效的旧根，它也不能超过其预定的使用期限。这确保了即使旧的Merkle根被接受，它们也只在安全的、预定的时间窗口内有效。)
	2. Official issue:   Adding a new member to a Semaphore group changes the root, **invalidating any currently-generated proofs** (从而使当前正在生成的 proof 都无效化了). This makes it so that a transaction submitted with a valid proof can become invalid if another transaction modifying a group is submitted first.
		1. Solution : Root history kind of solves this (by accepting the root as well as the proof, and ensuring the root was part of the group at some point)
5. If the nullifier hash has been used before (to prevent double-spends), it reverts with an error `Semaphore__YouAreUsingTheSameNillifierTwice`.
6. It then calls a `verifyProof` function on what seems to be another contract or module named `verifier`. This likely checks the zk-SNARK proof to ensure its validity.
7. Once verification succeeds, the nullifier hash is marked as used.

```ts
/// Saves the nullifier hash to avoid double signaling and emits an event
/// if the zk proof is valid.
function verifyProof(
	uint256 groupId,
	uint256 merkleTreeRoot,
	uint256 signal,
	uint256 nullifierHash,
	uint256 externalNullifier,
	uint256[8] calldata proof
) external override {

    uint256 merkleTreeDepth = getMerkleTreeDepth(groupId);
	if (merkleTreeDepth == 0) { revert Semaphore__GroupDoesNotExist(); }

	uint256 currentMerkleTreeRoot = getMerkleTreeRoot(groupId);

	// A proof could have used an old Merkle tree root.
	// https://github.com/semaphore-protocol/semaphore/issues/98
	if (merkleTreeRoot != currentMerkleTreeRoot) {
		uint256 merkleRootCreationDate = groups[groupId].merkleRootCreationDates[merkleTreeRoot];
		uint256 merkleTreeDuration = groups[groupId].merkleTreeDuration;

		if (merkleRootCreationDate == 0) {
			revert Semaphore__MerkleTreeRootIsNotPartOfTheGroup();
		}

		if (block.timestamp > merkleRootCreationDate + merkleTreeDuration) {
			revert Semaphore__MerkleTreeRootIsExpired();
		}
	}

	if (groups[groupId].nullifierHashes[nullifierHash]) {
		revert Semaphore__YouAreUsingTheSameNillifierTwice();
	}
    /// in SemaphoreVerifier.sol, a Pairing implementation.
	verifier.verifyProof(merkleTreeRoot, nullifierHash, signal, externalNullifier, proof, merkleTreeDepth);

	groups[groupId].nullifierHashes[nullifierHash] = true;

	emit ProofVerified(groupId, merkleTreeRoot, nullifierHash, externalNullifier, signal);
}
```

##### 2 | SemaphoreVerifier.sol

It is a modified version of the `Groth16` verifier template of SnarkJS
deal with `Pairings` ...

```ts
function verifyProof(
        uint256 merkleTreeRoot,
        uint256 nullifierHash,
        uint256 signal,
        uint256 externalNullifier,
        uint256[8] calldata proof,
        uint256 merkleTreeDepth
    ) external view override {
        signal = _hash(signal);
        externalNullifier = _hash(externalNullifier);

        Proof memory p;

        p.A = Pairing.G1Point(proof[0], proof[1]);
        p.B = Pairing.G2Point([proof[2], proof[3]], [proof[4], proof[5]]);
        p.C = Pairing.G1Point(proof[6], proof[7]);

        /// ....
```

##### 3 | SemaphoreGroups.sol

**_createGroup**:
    - It initializes a new group with an associated Merkle tree.
    - The zeroValue is a unique placeholder value created using the `groupId`, which minimizes the risk of potential Merkle tree vulnerabilities.
**_addMember**:
    - It allows for adding an identity commitment (a member) to an existing group's Merkle tree.
**_updateMember**:
    - It updates an existing identity commitment in a group.
    - This requires a proof of membership (proofSiblings and proofPathIndices) to ensure that the identity being updated is already part of the Merkle tree.
**_removeMember**:
    - It removes an identity commitment from a group.
    - Like the `_updateMember` function, this also requires a proof of membership.

`IncrementalBinaryTree.sol` : 
  - `@zk-kit` 是 `PSE` 开发的一套 zk 工具
  - `IncrementalBinaryTree` Solidity script is a library for managing an Incremental Binary Merkle Tree using the Poseidon hash function. **This type of tree allows to compute the root hash every time a leaf is inserted**, maintaining the integrity of the tree. It also supports leaf updates and removals.
  - 所以 `merkleTrees` 每次向某个 Group 插入一个 `identityCommitment` 都会更新其 root

```rust
// semaphore/packages/contracts/contracts/base/SemaphoreGroups.sol
import "@zk-kit/incremental-merkle-tree.sol/IncrementalBinaryTree.sol";
// Associate type, 关联类型, 可直接调用其函数
using IncrementalBinaryTree for IncrementalTreeData; 

mapping(uint256 => IncrementalTreeData) internal merkleTrees;

function _addMember(uint256 groupId, uint256 identityCommitment) internal virtual {
	if (getMerkleTreeDepth(groupId) == 0) {
		revert Semaphore__GroupDoesNotExist();
	}

	merkleTrees[groupId].insert(identityCommitment);

	uint256 merkleTreeRoot = getMerkleTreeRoot(groupId);
	uint256 index = getNumberOfMerkleTreeLeaves(groupId) - 1;

	emit MemberAdded(groupId, index, identityCommitment, merkleTreeRoot);
}

// @dev See {ISemaphore-addMembers}.
// 批量向 Group 中添加用户
function addMembers(
	uint256 groupId,
	uint256[] calldata identityCommitments
) external override onlyGroupAdmin(groupId) {
	for (uint256 i = 0; i < identityCommitments.length; ) {
		_addMember(groupId, identityCommitments[i]);

		unchecked {
			++i;
		}
	}

	uint256 merkleTreeRoot = getMerkleTreeRoot(groupId);

	groups[groupId].merkleRootCreationDates[merkleTreeRoot] = block.timestamp;
}
```

`Identity` 所对应的 `commitment` 会添加到一个 `merkle` 树上，merkle root 更新, 同时新的 `merkle root` 会记录在 root_history 的 mapping 中

#####  4 | ./test/Semaphore.ts

These're test in Semaphore test :
1. Verify the proof
2. **a proof can't be verified more than once(on chain)** 
	1. like the money you can't withdraw **twice** !
```ts
before(async () => {
	await semaphoreContract.addMembers(groupId, [members[1], members[2]])

	fullProof = await generateProof(identity, group, group.root, signal, {
		wasmFilePath,
		zkeyFilePath
	})
})

it("Should verify a proof for an onchain group correctly", async () => {
	const transaction = semaphoreContract.verifyProof(
		groupId,
		fullProof.merkleTreeRoot,
		fullProof.signal,
		fullProof.nullifierHash,
		fullProof.merkleTreeRoot,
		fullProof.proof
	)

    // This assertion checks if calling the `verifyProof` method emits a `ProofVerified` event from the `semaphoreContract` with the expected arguments.
	await expect(transaction)
		.to.emit(semaphoreContract, "ProofVerified")
		.withArgs(
			groupId,
			fullProof.merkleTreeRoot,
			fullProof.nullifierHash,
			fullProof.merkleTreeRoot,
			fullProof.signal
		)
})

it("Should not verify the same proof for an onchain group twice", async () => {
	const transaction = semaphoreContract.verifyProof(
		groupId,
		fullProof.merkleTreeRoot,
		fullProof.signal,
		fullProof.nullifierHash,
		fullProof.merkleTreeRoot,
		fullProof.proof
	)

	await expect(transaction).to.be.revertedWithCustomError(
		semaphoreContract,
		"Semaphore__YouAreUsingTheSameNillifierTwice"
	)
})
```

### **How to get involved? Build together**

Semaphore is a project by and for the Ethereum community, and we welcome all kinds of [contributions](https://github.com/semaphore-protocol#ways-to-contribute). You can find guidelines for contributing code on [this page](https://github.com/semaphore-protocol/semaphore/blob/main/CONTRIBUTING.md).

If you want to experiment with Semaphore, the [Quick Setup guide](https://semaphore.appliedzkp.org/docs/quick-setup) and [Semaphore Boilerplate](https://github.com/semaphore-protocol/boilerplate) are great places to start. Feel free to [get in touch](https://t.me/joinchat/B-PQx1U3GtAh--Z4Fwo56A) with any questions or suggestions, or just to tell us about your experience!

We would also love to hear from developers who are interested in integrating Semaphore into new or existing dapps. Let us know what you’re working on by [opening an issue](https://github.com/semaphore-protocol/semaphore/issues/new?assignees=&labels=documentation++%F0%9F%93%96&template=----project.md&title=), or get in touch through the Semaphore [Telegram group](https://t.me/joinchat/B-PQx1U3GtAh--Z4Fwo56A) or the [PSE Discord](https://discord.com/invite/g5YTV7HHbh).
