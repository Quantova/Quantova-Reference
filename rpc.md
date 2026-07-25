# The gateway RPC

The gateway is the client facing surface of a Quantova node. Every method is an HTTP POST to `/v1/<method>` with a flat JSON body, and the reply is a JSON object. The methods and their shapes below are taken from the gateway source in Quantova-Chain under `crates/qtv-gateway/src`. The field values in the examples are illustrative and only the shapes are normative.

## Transport

A request is a POST to a method path under the version prefix, for example `POST /v1/node_info`. The body is a flat JSON object, and an empty body is read as an empty object so a method with no fields needs no body. The reply carries `Content-Type application/json`. The gateway answers an OPTIONS preflight with 204 and sets permissive CORS headers, and it rejects any verb other than POST with 405. The request head is capped at 16 KiB and the body at 2 MiB, a connection over the cap is refused, and a slow request times out at fifteen seconds. A local devnet node serves the gateway on `127.0.0.1:8645`.

An error reply is the object `{"error":"<code>","message":"<text>"}`. The codes a client sees are `bad_request` and `bad_address` at 400, `not_found` at 404, `unknown_method` at 404, `method_not_allowed` at 405, `too_large` at 413, `head_too_large` at 431, and `busy` or `unavailable` at 503.

The methods are `node_info`, `head`, `validators`, `chain_params`, `staking_state`, `get_account`, `get_transaction`, `submit_transaction`, `get_block`, `pending`, `supply`, `get_container`, `get_storage`, and `get_events`.

## node_info

Returns the chain identity, the head height, the asset and its denomination, the current fee, and the node version.

**Path** `POST /v1/node_info`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| chain_id | string | the chain identifier |
| genesis_hash | string | the genesis hash in hex |
| head_height | integer | the height of the finalized head |
| asset | string | the native asset symbol, TQTOV on the testnet |
| denomination | string | the base unit name, Quon |
| fee | object | the current fee, see below |
| version | string | the node version |

The fee object carries `transfer_micro_usd`, `rate_micro_usd_per_qtov`, `quon_per_qtov`, and `transfer_quon`, each a decimal string. The fee is capped in the native asset and targeted in United States dollars, held to a tenth of a cent by the governance set rate.

```
POST /v1/node_info
{}
```
```
{"chain_id":"Q-test-net-1","genesis_hash":"9f2c...","head_height":10428,"asset":"TQTOV","denomination":"Quon","fee":{"transfer_micro_usd":"500","rate_micro_usd_per_qtov":"1000000","quon_per_qtov":"1000000","transfer_quon":"500"},"version":"0.1.0"}
```

## head

Returns the current finalized head.

**Path** `POST /v1/head`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| height | integer | the height of the head |
| block | string or null | the block id as a QBK identifier, null before the first finalized block |
| state_root | string | the state root as a QST identifier |

```
POST /v1/head
{}
```
```
{"height":10428,"block":"QBK1QW8P...","state_root":"QST1M4RD..."}
```

## validators

Returns the active validator set with each validator's staked weight.

**Path** `POST /v1/validators`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| count | integer | the number of validators |
| validators | array | one entry per validator |

Each validator entry carries `address`, a Q1 string, and `stake`, an integer weight in Quon.

```
POST /v1/validators
{}
```
```
{"count":1,"validators":[{"address":"Q1QW8P...","stake":10000000000}]}
```

## chain_params

Returns the staking and governance parameters the chain runs under.

**Path** `POST /v1/chain_params`

The request has no fields.

The response has a `staking` object and a `governance` object. The staking object carries `native_unit`, `min_stake`, `staking_pool`, `session_days`, `high_session_tx`, `low_session_bps`, `high_session_bps`, `reward_cap_micro_usd_per_session`, `mainnet_blackout_days`, `bond_lock_days`, `unbonding_days`, `vest_cliff_days`, `vest_tranche_days`, and `vest_tranches`, each an integer. The governance object carries `conviction_max_x10`, an integer, and `tracks`, an array where each track carries `code`, `deposit`, `approval_bps`, `support_bps`, and `period_seconds`. There are seven governance tracks and the example below shows the first.

```
POST /v1/chain_params
{}
```
```
{"staking":{"native_unit":1000000,"min_stake":2000000000,"staking_pool":685714000000,"session_days":182,"high_session_tx":50000000000,"low_session_bps":100,"high_session_bps":175,"reward_cap_micro_usd_per_session":4000000000,"mainnet_blackout_days":365,"bond_lock_days":90,"unbonding_days":21,"vest_cliff_days":365,"vest_tranche_days":120,"vest_tranches":4},"governance":{"conviction_max_x10":25,"tracks":[{"code":1,"deposit":600000,"approval_bps":8000,"support_bps":4000,"period_seconds":1209600}]}}
```

## staking_state

Returns the live staking pools and the price the reward cap is measured against.

**Path** `POST /v1/staking_state`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| reward_pool | integer | the remaining reward pool in Quon |
| treasury | integer | the treasury balance in Quon |
| price_micro_usd_per_qtov | string | the price in micro dollars per QTOV |
| mainnet_started | boolean | whether the mainnet start point has passed |
| governance_locked | string | the total stake locked in governance |

```
POST /v1/staking_state
{}
```
```
{"reward_pool":685714000000,"treasury":0,"price_micro_usd_per_qtov":"1000000","mainnet_started":false,"governance_locked":"0"}
```

## get_account

Returns the account record at a Q1 address.

**Path** `POST /v1/get_account`

| request field | type | meaning |
| --- | --- | --- |
| address | string | the Q1 address to read |

| response field | type | meaning |
| --- | --- | --- |
| address | string | the address that was read |
| nonce | integer | the next expected nonce |
| balance | string | the balance in Quon as a decimal string |
| scheme | integer | the signature scheme identifier of the account key |
| has_key | boolean | whether the account has a registered public key |

An address that is not a Q1 Bech32m address returns `bad_address` at 400.

```
POST /v1/get_account
{"address":"Q1QW8P..."}
```
```
{"address":"Q1QW8P...","nonce":3,"balance":"8000000000","scheme":1,"has_key":true}
```

## get_transaction

Returns the status and, when known, the fields of a transaction by its id.

**Path** `POST /v1/get_transaction`

| request field | type | meaning |
| --- | --- | --- |
| tx_id | string | the QTX transaction id |

The response always carries `tx_id` and `status`, where status is `finalised`, `pending`, or `unknown`. A finalised transaction also carries `height` and `block`. A finalised or pending transaction also carries the transaction fields `from`, `to`, `value`, `fee`, `nonce`, `meter_limit`, `scheme`, `signature`, and `raw`. The `value`, `fee`, `from`, `to`, `signature`, and `raw` fields are strings, where `signature` and `raw` are hex, and `nonce`, `meter_limit`, and `scheme` are integers.

```
POST /v1/get_transaction
{"tx_id":"QTX1A9F0..."}
```
```
{"tx_id":"QTX1A9F0...","status":"finalised","height":10420,"block":"QBK1QW8P...","from":"Q1QW8P...","to":"Q1M4RD...","value":"1000","fee":"500","nonce":2,"meter_limit":100000,"scheme":1,"signature":"a1b2...","raw":"00ff..."}
```

## submit_transaction

Submits a signed transaction to the mempool. The transaction is the canonical wrapper bytes in hex, as produced by QCore.

**Path** `POST /v1/submit_transaction`

| request field | type | meaning |
| --- | --- | --- |
| tx | string | the canonical signed transaction in hex |

On acceptance the reply is `verdict` `accepted`, `state` either `fresh` or `known`, and `tx_id`. On rejection the reply is `verdict` `rejected` and `reason`. The reason is one of `malformed`, `unknown_sender`, `unsupported_scheme`, `bad_signature`, `bad_nonce`, `bad_call`, `self_transfer`, `meter_limit_too_low`, `fee_too_low`, or `insufficient_funds`. A `bad_nonce` rejection also carries `expected` and `got`.

```
POST /v1/submit_transaction
{"tx":"00ff1234..."}
```
```
{"verdict":"accepted","state":"fresh","tx_id":"QTX1A9F0..."}
```

## get_block

Returns a finalized block by height or by block id. Supply exactly one of the two.

**Path** `POST /v1/get_block`

| request field | type | meaning |
| --- | --- | --- |
| height | integer | the block height, optional |
| block | string | the QBK block id, optional |

| response field | type | meaning |
| --- | --- | --- |
| height | integer | the block height |
| block | string | the block id |
| parent | string | the parent block id as a QBK identifier |
| state_root | string | the state root as a QST identifier |
| proposer | string | the proposer address |
| time | integer | the block time |
| tx_count | integer | the number of transactions |
| extra_data | string | the header extra data in hex |
| tx_ids | array | the transaction ids in the block |

A missing height or block id returns `bad_request`, and a height or id with no finalized block returns `not_found` at 404.

```
POST /v1/get_block
{"height":10420}
```
```
{"height":10420,"block":"QBK1QW8P...","parent":"QBK1H3K2...","state_root":"QST1M4RD...","proposer":"Q1QW8P...","time":1700004200,"tx_count":1,"extra_data":"","tx_ids":["QTX1A9F0..."]}
```

## pending

Returns the transactions currently in the mempool.

**Path** `POST /v1/pending`

The request has no fields.

The reply carries `count` and `transactions`, an array where each entry carries `tx_id` and the same transaction fields as `get_transaction`.

```
POST /v1/pending
{}
```
```
{"count":1,"transactions":[{"tx_id":"QTX1A9F0...","from":"Q1QW8P...","to":"Q1M4RD...","value":"1000","fee":"500","nonce":2,"meter_limit":100000,"scheme":1,"signature":"a1b2...","raw":"00ff..."}]}
```

## supply

Returns the total native supply in Quon.

**Path** `POST /v1/supply`

The request has no fields.

| response field | type | meaning |
| --- | --- | --- |
| supply_quon | string | the total supply in Quon as a decimal string |

```
POST /v1/supply
{}
```
```
{"supply_quon":"4571429000000"}
```

## get_container

Returns the bytecode container held at a contract address.

**Path** `POST /v1/get_container`

| request field | type | meaning |
| --- | --- | --- |
| address | string | the Q1 contract address |

| response field | type | meaning |
| --- | --- | --- |
| address | string | the address that was read |
| container | string | the container bytecode in hex |
| size | integer | the container length in bytes |

An address that is not a Q1 address returns `bad_address`, and an address that holds no contract returns `not_found` at 404.

```
POST /v1/get_container
{"address":"Q1C0DE..."}
```
```
{"address":"Q1C0DE...","container":"00010203...","size":512}
```

## get_storage

Returns the contract storage slots at a contract address.

**Path** `POST /v1/get_storage`

| request field | type | meaning |
| --- | --- | --- |
| address | string | the Q1 contract address |

The reply carries `address` and `slots`, an array where each entry carries `slot`, the slot key in hex, and `value`, the slot value as a decimal string. An address that is not a Q1 address returns `bad_address`.

```
POST /v1/get_storage
{"address":"Q1C0DE..."}
```
```
{"address":"Q1C0DE...","slots":[{"slot":"0000...0001","value":"42"}]}
```

## get_events

Returns the events recorded at a block height.

**Path** `POST /v1/get_events`

| request field | type | meaning |
| --- | --- | --- |
| height | integer | the block height |

The reply carries `height`, `count`, and `events`, an array where each entry carries `contract`, the contract address, `selector`, the event selector in hex, and `data`, the event payload in hex. A missing height returns `bad_request`.

```
POST /v1/get_events
{"height":10420}
```
```
{"height":10420,"count":1,"events":[{"contract":"Q1C0DE...","selector":"1a2b3c4d","data":"00ff..."}]}
```
