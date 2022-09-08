## Cryptography

My review of cryptography and data structures used in Cosmos/Tendermint.
Work done for the Polkadot Blockchain Academy event at Cambdrige in July 2022.
This review is not exhaustive, and can be enhanced.

---

### Asymmetric cryptography

Elliptic Curve Cryptography protocol is used in Cosmos SDK.

Cosmos SDK provides 3 digital signature algorithms for creating digital signatures:

- ECDSA on secp256k1: use by accounts
- ECDSA on secp256r1: use by accounts
  (for supporting signing algorithm on secure enclave in macOS/iOS/watchOS and Android Hardware-backed Keystore)
- EdDSA on Ed25519: Only used for Consensus validation on Tendermint side

We can see that user accounts and consensus validator accounts don't use the same digital signature algorithm.

---

### Public key compressed format for secp256k1 & secp256r1 I

secp256k1 elliptic curve with formula: `y^2 = x^3 + 7`.

<img src="./secp256k1.png" style="width: 50%" alt="secp256k1 curve" />

---

### Public key compressed format for secp256k1 & secp256r1 II

A public key is just a point on the elliptic curve secp256k1.

The `G point` is added to himself a secret number of times (the `private key`) to finally land on a point on the curve which is the `public key`.

As an y-coordinate can be calculated easily from an x-coordinate following the formula `y^2 = x^3 + 7`, we can just keep the x-coordinate + a prefix to identify if the y-coordinate associated is the largest one or the smaller one.

Cosmos follows best practice by compressing the public key to 33 bytes (1 byte prefix + 32 bytes x-coordinate):

- `0x02` prefix concatenated to the x-coordinate if the y-coordinate associated is the largest one.
- `0x03` prefix concatenated to the x-coordinate if the y-coordinate associated is the smaller one.

---

### Signature

Processes involving signing with private keys:

- application transactions and common usage: token transfers, smart contract interactions, etc...
- governance: submitting and voting for proposals.
- consensus: proposing and voting for new blocks.

Tendermint adopted [zip215](https://zips.z.cash/zip-0215) for verification of ed25519 signatures.

Multisig transactions are a built-in feature in the Cosmos SDK.

---

### Cryptographic hash function

The cryptographic hash function used in Cosmos is SHA-256.

It is used everywhere a hash function is needed:

- compute addresses from public keys.
- Merkle tree hashing function.

_RIPEMD160_ cryptographic hash function has been [fully removed](https://github.com/cosmos/cosmos-sdk/pull/2308) from all components of Cosmos/Tendermint.

---

### Symmetric encryption

_XSalsa20_ Symmetric Cipher is used to store on disk encrypted private keys protected with a passphrase.

---

### Exotic primitives

There are initiatives for supporting ([Zero Knowledge Validator](https://medium.com/zero-knowledge-validator/cosmos-privacy-zkp-showcase-recap-8ad8def58573)) and for using ([Penumbra Blockchain](https://penumbra.zone)) ZKP in cosmos ecosystem for privacy.

But for Cosmos developers/builders (and Go developers in general) ZKP is a bleeding edge technology.
As tools for ZK and privacy are missing, mainly due to the fact that the CosmosSDK is written in Go, and many of the ZK related libraries are written in Rust and the ZK community is primarily focused on building with that language.

---

### Dependency on cryptography libraries I

CosmosSDK and Tendermint are written in Go, so libraries used for cryptography are imported from various Go packages and projects.

- secp256k1: depends on the implementation of the [decred project](https://github.com/decred/dcrd/tree/master/dcrec/secp256k1). This is also the library used in [btcd](https://github.com/btcsuite/btcd)(a bitcoin node written in go).
- secp256r1: depends on [Go core crypto package](https://pkg.go.dev/crypto/elliptic#P256)
- ed25519: depends on [Go core crypto package](https://pkg.go.dev/crypto/ed25519)
- SHA256: depends on [Go core crypto package](https://pkg.go.dev/crypto/sha256)
- XSalsa20: depends on [secretbox](https://pkg.go.dev/golang.org/x/crypto/nacl/secretbox). It is also a Go core crypto package.

---

### Dependency on cryptography libraries II

- Don’t Roll Your Own Crypto, particularly on a controversial elliptic curve.
- Writing cryptography software isn’t like writing regular software. Crypto is Hard.
- All the security of user accounts in Cosmos depend on a secp256k1 Go module written by the [decred project](https://github.com/decred/dcrd/tree/master/dcrec/secp256k1) from scratch (with their own maths).
- Yes a module in a project, not a library!!!
- Even the `go-ethereum`, which is one of the biggest project written in Go using secp256k1, and probably the more mature, preferred to wrap the [bitcoin secp256k1 C library](https://pkg.go.dev/github.com/ethereum/go-ethereum/crypto/secp256k1).

---

### Accounts

At the core of every Cosmos account, there is a seed, which takes the form of a 12 or 24-words mnemonic. From this mnemonic, it is possible to create any number of Cosmos accounts, i.e. pairs of private key/public key

---

### Accounts

#### Hierarchical Deterministic Key Derivation

Cosmos use hard derivation for deriving multiple cryptographic keypairs from a single secret following [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki), [BIP43](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) and [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) specifications and principally the [confio specifications](https://github.com/confio/cosmos-hd-key-derivation-spec) for HD key derivation standarization on the Cosmos ecosystem.

An "HD path" is an instruction as to how to derive a keypair from a root secret. [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) specifies a schema for such paths as follows:

`m / 44' / coin_type' / account' / change / address_index`

---

### Accounts

#### Hierarchical Deterministic Key Derivation

The Cosmos Hub HD path is:

`m / 44' / 118' / 0' / 0 / address_index`

- 44 is the BIP44
- 118 is the coin type for ATOM as defined in [SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md)
- address_index is used for multiple accounts of the same user.

---

### Accounts

#### Hierarchical Deterministic Key Derivation

```text
     Account 0                         Account 1                         Account 2
+------------------+              +------------------+               +------------------+
|    Address 0     |              |    Address 1     |               |    Address 2     |
|        ^         |              |        ^         |               |        ^         |
|        |         |              |        |         |               |        |         |
|  Public key 0    |              |  Public key 1    |               |  Public key 2    |
|        ^         |              |        ^         |               |        ^         |
|        |         |              |        |         |               |        |         |
|  Private key 0   |              |  Private key 1   |               |  Private key 2   |
+------------------+              +------------------+               +------------------+
         |                                 |                                  |
         +--------------------------------------------------------------------+
                                           |
                                 +---------+---------+
                                 |  Mnemonic (Seed)  |
                                 +-------------------+
```

---

### Accounts

#### Addresses in Cosmos

- Addresses are calculated by hashing the public key using SHA-256 and truncating it to only use the first 20 bytes of the slice.

  ```rust
  // An hypothetical rust code
  let addr = SHA265(pub_key)[0..20];
  ```

For user facing representation/interaction, addresses are formatted using [Bech32](https://en.bitcoin.it/wiki/Bech32) with a prefix which depend on the type of the address:

- account addresses are prefixed with `cosmos`
- validator operator addresses are prefixed with `cosmosvaloper`
- consensus node addresses are prefixed with `cosmosvalcons`

---

### Data structures

Cosmos Applications and Tendermint uses a lot of data structures for storing important data:
Block, Transaction, Vote, Events, Application's state.
The notable one are tree-like data structure:

- Simple Merkle tree: used for storing transaction hashs. The Merkle root is stored in the block header.
- IAVL+ Tree: an implementation of a key-value pair based storage for the application's state.

Because Tendermint only uses a Simple Merkle Tree, application developers are expect to use their own Merkle tree in their applications.
For example, the IAVL+ Tree is an immutable self-balancing binary tree for persisting application state is defined in the [Cosmos SDK](https://github.com/cosmos/cosmos-sdk/blob/ae77f0080a724b159233bd9b289b2e91c0de21b5/docs/interfaces/lite/specification.md).

Any data structure can be used as a store, as long as it implements de [BasicKVStore](https://github.com/cosmos/cosmos-sdk/blob/main/store/types/store.go#L197) and [KVStore](https://github.com/cosmos/cosmos-sdk/blob/main/store/types/store.go#L212).

---

### Data structures

#### Merkle tree

Tendermint uses RFC 6962 specification of a merkle tree, with SHA-256 as the hash function.
Merkle trees are used throughout Tendermint to compute a cryptographic digest of a data structure.

2 Particularities for RFC 6962 compliant merkle trees:

- leaf nodes and inner nodes have different hashes. This is for "second pre-image resistance", to prevent the proof to an inner node being valid as the proof of a leaf. The leaf nodes are SHA256(0x00 || leaf_data), and inner nodes are SHA256(0x01 || left_hash || right_hash). `||` means concatenation.

- When the number of items isn't a power of two, the left half of the tree is as big as it could be. (The largest power of two less than the number of items) This allows new leaves to be added with less recomputation.

---

To compute a Merkle proof, the number of aunts is limited to 100 (MaxAunts) to protect the node against DOS attacks. This limits the tree size to 2^100 leaves. In Cosmos aunts are hashes from leaf's sibling to a root's child.
https://github.com/tendermint/tendermint/blob/master/crypto/merkle/proof.go

---

### Data structures

#### Merkle tree with 6 items

```text
            root
             / \
           /     \
         /         \
       /             \
    h0123            h45
     / \             / \
    /   \           /   \
   /     \         /     \
  h01    h23      h4     h5
 / \     / \      |      |
h0  h1  h2 h3     |      |
|   |   |  |      |      |
d0  d1  d2 d3     d4     d5

h0 = SHA256(0x00 || d0); h3 = SHA256(0x00 || d3);
h1 = SHA256(0x00 || d1); h4 = SHA256(0x00 || d4);
h2 = SHA256(0x00 || d2); h5 = SHA256(0x00 || d5);
h01 = SHA256(0x01 || h0 || h1);
h23 = SHA256(0x01 || h2 || h3);
h45 = SHA256(0x01 || h4 || h5);
h0123 = SHA256(0x01 || h01 || h23);
root = SHA256(0x01 || h0123 || h45);
```

---

### Data structures

#### Merkle tree with 7 items

Adding a 7th element to the merkle tree:

```text
              *
             / \
           /     \
         /         \
       /             \
      *               *
     / \             / \
    /   \           /   \
   /     \         /     \
  *       *       *       h6
 / \     / \     / \
h0  h1  h2  h3  h4  h5
```

---

### Data structures

#### IAVL+ Tree

The purpose of this data structure is to provide persistent storage for key-value pairs, for example store account balances.

In Ethereum, the analog is Patricia tries.

Pro:

- Keys do not need to be hashed prior to insertion in IAVL+ trees, so this provides faster iteration in the key space which may benefit some applications.
- The logic is simpler to implement, requiring only two types of nodes -- inner nodes and leaf nodes

Cons:

- while IAVL+ trees provide a deterministic merkle root hash, it depends on the order of transactions.
- slow and inefficient

---

### What did we learn from Cosmos's cryptography

- They follow a lot of good conventions and specifications which are from Bitcoin era and are still valid and used today.
- Reliable cryptography on Tendermint side
- Usage of good cryptography libraries for hashing, encryption and Ed25519.
- The spec256k1 library they used is coming from a project which is not so reputable in cryptography development.
- While Bitcoin used spec256k1 in 2009, one could expect Cosmos to use a less controversial elliptic curve (NSA backdoor).
- Would like to see a better data structures for storing application's state like Sparse Merkle Tree (expected Q1 2023), MMR in the CosmosSDK
- Different Elliptic curves, digital signature algorithms, address types are used on application side and consensus side. It adds complexity.
