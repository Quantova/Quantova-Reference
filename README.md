# Quantova Reference

Quantova is a post quantum Layer 1 blockchain. Its cryptography, virtual machine, contract language, and consensus are built on the NIST post quantum standards, so the keys, signatures, finality, and randomness that secure the network are designed to withstand both classical and quantum attack. The stack is written to its own specification. It is at the testnet and pre audit stage, and this reference states what is built today, drawn from the source and the normative specifications.

This repository is the developer map. It gives the layout of the stack, every gateway RPC method in one place in [rpc.md](rpc.md), and every Q primitive, identifier family, and cryptographic scheme in one place in [primitives.md](primitives.md). Everything here is drawn from the source and the normative specifications, not invented.

## The layout

| repository | what it is |
| --- | --- |
| Quantova-Chain | the node, the mempool, the state trie, execution, and the gateway that clients talk to |
| QVM | the register machine that runs compiled containers, with the post quantum operations wired in as instructions |
| QRC-CONSENSUS | QORUS, the committee based consensus that records finality as a single aggregated certificate |
| Quanta-Smart-Contract-language | the Quanta contract language and its compiler |
| Q-Crypto | the from scratch post quantum primitives, ML-DSA, ML-KEM, SLH-DSA, SHA-3, and SHAKE |
| QCore.rs, QCore.js, QCore.py | the one client core and its per language bindings |
| QNS | the Quantova Name Service |
| QVRF | the verifiable random function used for sampling |
| Quantova-Specs | the normative specifications every component is built from |
| Quantova-Conformance | the frozen test vectors |

## The QVM

The QVM is the execution layer. It is a deterministic register machine that runs a compiled bytecode container. It has sixteen general registers of 64 bit words, a linear scratch memory that is zeroed at entry and sized to hold a full post quantum key or signature or proof, and a bounded operand and call stack. Arithmetic is checked so an overflow faults rather than wraps, and on any fault the machine rolls back and records nothing. The post quantum operations are first class instructions, so a contract verifies a post quantum signature, hashes over SHA-3, checks a Merkle proof, or verifies a random output through native instructions rather than rolling its own cryptography. Read the instruction set in Quantova-Specs under SPEC-isa.md and the container format under SPEC-container.md.

## QORUS consensus

QORUS is Quantova's consensus. A committee is sampled each round from the stake ranked active set by the verifiable random function, a leader proposes a block, the committee attests with the module lattice signature, and a supermajority of two thirds plus one aggregates into a single STARK backed certificate that finalizes the block. Finality is recorded as that one aggregated certificate rather than a list of votes, so it does not consume block space, and it is reached in under a second. The committee is bounded, so the work to finalize a block does not grow with the validator set. Validation is permissionless and Sybil resistance comes from stake alone, and every validator re executes every block, so one honest re executor protects validity. QORUS reaches agreement so long as the dishonest stake stays a minority. Read SPEC-consensus-qorus.md in Quantova-Specs and the implementation in QRC-CONSENSUS.

## The Quanta contract language

Quanta is the Quantova contract language, compiled to QVM containers. Assets are linear resources and authority is a capability, so a contract declares what it may do and the compiler holds it to that. The language is designed so that whole classes of vulnerability, among them reentrancy, unchecked arithmetic, and value created or destroyed without accounting or authority, are caught at compile time. Source files carry the extension qs and compile to a bytecode container with an embedded interface descriptor. Authority comes from a `signed by` parameter that desugars to a native post quantum signature verify, asset conservation is proven by a linear flow analysis over the `conserves` clause, and a competitive order that is not `sealed` does not compile. Read SPEC-quanta-language.md and SPEC-lowering.md in Quantova-Specs.

## Q-Crypto

Q-Crypto is Quantova's implementation of the NIST post quantum standards, written from scratch against the published FIPS documents. It provides the module lattice signature ML-DSA-65 under FIPS 204, the hash based signature SLH-DSA under FIPS 205, key encapsulation with ML-KEM-768 under FIPS 203, and hashing with SHA-3 and SHAKE under FIPS 202, and it is validated against the official NIST vectors. Every operation maps to an exact FIPS reduction recorded in the specifications. The node and the client build on this same code, so a client and the chain agree byte for byte.

## The QCore SDK family

QCore is the client core, one implementation of the four things a client has to get exactly right, key derivation, transaction building, signing, and speaking the gateway wire. It is written once in Rust as QCore.rs, and QCore.js and QCore.py are generated over that same core so a signature is never rewritten in another language. QCore.js is published on npm as `@quantovainc/qcore` and carries a compiled core build for the browser and a native build for Node. QCore.py is the Python binding. All three derive the same Q1 address from the same seed and sign the same transaction body to the same bytes. The client passes the highest fee it will accept and the core refuses to sign a fee the gateway reports above it, so a gateway cannot inflate the fee and drain the account. Read the QCore.rs README for the full surface.

## QNS

The Quantova Name Service is the human layer of identity. A name ends in the suffix q, reads as a label then a dot then q, and resolves forward to a Q1 address while an address resolves back to its primary name. A name is a pointer in an on chain registry rather than a key, so a short name resolves safely to a full length address. Every record is signed with the module lattice signature, and the whole resolution path is post quantum. A memorable name gives a reader something to recognise where a raw address does not. Read SPEC-qns.md in Quantova-Specs.

## The token standards QAsset and QCollectible

QAsset is the fungible token standard and QCollectible is the collectible standard, both written directly in the Quanta primitives so their guarantees are proven by the compiler rather than hand rolled. QAsset gives a regulated issuer the controls the law requires under an issuer quorum of guardian keys with a threshold, freeze and unfreeze of a named account, an evidence bound clawback, quorum gated mint and burn, and an emergency pause, each gated by the quorum and each emitting a permanent event. Optional compliance hooks such as an allow or deny registry and a sanctions freeze are opt ins the chain never imposes. QCollectible tracks ownership of a collectible by its identifier with a per token owner and a signed mint. These standards use the module lattice or the hash based signature for every authority check. The reference contracts live in the Quanta standard library and are brought up as the compiler reaches its milestones. Read SPEC-token-standard.md in Quantova-Specs and the QCollectible and issuer token examples in the language repository.

## Where to read more

The two companion files in this repository are [rpc.md](rpc.md), which documents every gateway method with its path, request fields, response fields, and a small example, and [primitives.md](primitives.md), which documents every Q primitive, the address scheme, the identifier families, the asset and its units, and the cryptographic schemes with their FIPS numbers. The full normative detail lives in Quantova-Specs.
