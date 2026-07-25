# The primitives

This file documents every Q primitive the Quanta language and the QVM share, the address scheme, the identifier families, the native asset and its units, and the cryptographic schemes with their FIPS numbers. It is drawn from the normative specifications in Quantova-Specs and from the identifier code in Quantova-Chain under `crates/qtv-idfmt`. The stack is at the testnet and pre audit stage.

## The Q primitive catalog

The typed primitives are the language surface over the machine instructions. Each type is defined so a value of it can only be produced in the sanctioned way, and the code generator lowers each to its instruction and never re implements a cryptographic operation in the language.

### Q_Address

An address names an account or a contract. It is the SHA-3 256 hash of the scheme identifier byte followed by the public key, rendered in Bech32m as a Q1 string and never as hexadecimal. It is compared by value. See the address scheme below.

### Q_Sig of T

A signature is a scheme agnostic fact that binds a signed message to the party that signed it. A value of this type is produced only by the machine verifying a lattice signature over exactly those bytes, so a contract cannot forge one and an unchecked authority cannot be expressed. It lowers to the verify instruction chosen by the one byte scheme identifier, ML-DSA to the module lattice verify, SLH-DSA to the hash based verify, and FN-DSA to its verify behind the fn dsa feature flag. A contract never names a scheme.

### Q_Asset

An asset value carries an amount of a declared asset kind and is linear, so it must be used exactly once and copying it or dropping it is a compile error, which is what makes a double spend and a lost balance impossible to write. An asset can be split, merged, and sent. Its conservation is proven at type check and preserved through lowering with no runtime substitute. Amounts are 128 bit. Some asset kinds are origin tagged bridged kinds identified by the pair of origin chain and origin asset, and an origin tagged asset is never valid as validator stake, so only the native asset secures consensus.

### Q_Commit of T

A commitment binds a hidden value to a public digest using SHA-3, so a party can commit now and reveal later without being able to change the value. It lowers to the SHA-3 hash instruction.

### Q_Rand

A randomness value is a verified output of the verifiable random function. It is produced only by the machine verifying a random output and its proof, so a contract cannot forge randomness. It lowers to the random function verify instruction.

### Q_Sealed of T

A sealed value is confidential in the mempool. It is carried under key encapsulation and is opened only at execution, which gives protection against front running without breaking any conservation or audit. A competitive or pooled order that is not sealed does not compile. The decapsulation and authenticated decryption that open a sealed value land in the sequencing wave, and until then the compiler refuses to lower a sealed opening rather than read sealed bytes in the clear, so the guarantee stays a compile time fact.

### Q_Key

A key value holds a public key together with its scheme identifier byte, so the machine knows which scheme to use when it verifies. The secret half is never a value inside a contract, and the code generator can emit no path that reads a private seed.

## The address scheme

An address is the Bech32m encoding of a payload under the human readable part Q, where the payload is the SHA-3 256 hash of the scheme identifier byte followed by the public key. Because the character after the human readable part is the natural Bech32 separator, every address reads as Q1 by construction. The payload is 32 bytes and the string is about sixty characters, checksummed and error detecting, and there is one width and no shorter tier, because every payload is sized by security at 256 bits for a 128 bit quantum preimage margin. The whole public key is bound into the address, nothing is truncated, and there is no key recovery from a signature, so one address names exactly one post quantum key. The address does not reveal the scheme, since the scheme byte is hashed into the payload.

The stored secret is a 32 byte seed, exported under the human readable part Q2 and also expressible as a 24 word mnemonic using Quantova derivation over SHAKE256. The full lattice key is expanded at signing time and is never stored or exported as the user key. The Q2 namespace is the secret namespace and is never a public identifier, so a string that reads Q2 is always a secret and never a hash.

## The scheme identifier registry

A one byte scheme identifier prefixes every key and every signature, and verification dispatches on it. The value 1 is ML-DSA-65 and is the default. The value 2 is SLH-DSA. The value 3 is FN-DSA, which is pre final and stays behind the fn dsa feature flag, off in every default build until the standard is final and audited. A contiguous range above these is reserved for other parameter sets of the same schemes so a future set never renumbers an existing one. A user account may sign with any registered scheme, but validator attestations and the QORUS finality certificate use ML-DSA only.

## The identifier families

Every hash and key the chain surfaces is rendered in the Bech32m identifier format, never in hexadecimal. Each family has a fixed human readable part and a fixed width. An address and a secret hold a 32 byte key floor, and a transaction, a block, a state root, a contract interface digest, and a proof each hold a fixed 32 byte digest.

| prefix | names | width | reads as |
| --- | --- | --- | --- |
| Q | an account or contract address | 32 byte payload, 32 byte floor | Q1 followed by Bech32m |
| Q2 | a secret seed | 32 byte seed | Q2 followed by Bech32m, never published |
| QTX | a transaction id | 32 byte digest | QTX1 followed by Bech32m |
| QBK | a block hash | 32 byte digest | QBK1 followed by Bech32m |
| QST | a state root | 32 byte digest | QST1 followed by Bech32m |
| QCID | a contract interface digest | 32 byte digest | QCID1 followed by Bech32m |
| QPF | a proof digest | 32 byte digest | QPF1 followed by Bech32m |

A selector, the handle a caller names an entry or an event by, is the leading four bytes of the SHA-3 hash of the canonical signature string, which is the name followed by the parenthesized parameter types written with the language type names.

## The asset and its units

The native asset is QTOV. Its base unit is Quon, and one QTOV is one million Quon. On the testnet the asset carries the symbol TQTOV. An amount is a 128 bit value and it always crosses the gateway wire as a decimal string, so a client number never rounds a balance, and a balance of one TQTOV is the string `1000000` in Quon. The fee is capped in the native asset and targeted in United States dollars, held to a tenth of a cent by the governance set rate. The dollar figure is a target rather than a hard cap, since a hard dollar cap would need a price feed read on the hot path.

## The cryptographic schemes

Only NIST standardized post quantum primitives secure the stack.

| scheme | standard | role |
| --- | --- | --- |
| ML-DSA-65 | FIPS 204 | the module lattice signature, the default account scheme at identifier 1, and the only scheme in consensus attestations and the finality certificate |
| SLH-DSA | FIPS 205 | the hash based signature, account scheme at identifier 2 |
| FN-DSA | pre final | the compact signature at identifier 3, behind the fn dsa feature flag and off in every default build until the standard is final |
| ML-KEM-768 | FIPS 203 | key encapsulation, the mechanism that seals a value in the mempool |
| SHA-3 and SHAKE | FIPS 202 | hashing, address derivation, commitments, selectors, and the SHAKE256 seed derivation and mnemonic |

The 256 bit symmetric primitives ChaCha20-Poly1305 and AES-256 provide authenticated symmetric encryption, and succinct proofs use hash based STARKs with no trusted setup.
