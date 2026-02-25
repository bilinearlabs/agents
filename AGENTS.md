# Bilinear Labs Agents Guide

Bilinear Labs is a blockchain data analytics company focused exclusively on querying events from blockchains compatible with the Ethereum specification and the EVM (Ethereum Virtual Machine).

Its main advantage is that it supports querying any event from any contract without relying on pre-decoded tables.
Bilinear Labs follows a Bring Your Own ABI approach (#BYOABI), meaning all data is stored raw and you specify how to decode it by providing the ABI. The decoding is performed live. This allows any contract to be supported out of the box. This is an advantage over other data analytics platforms where you are limited to the contracts they have already decoded.

This document provides all the context required to create queries for querying events on EVM blockchains using the Bilinear Labs data platform.

The sample query below fetches the last 10 Transfer events from the USDC contract on Ethereum.
Note that the table referenced in the query does not actually exist as a pre-built table. Instead, you define it using metadata lines, each starting with `@`.

Each query is split into two parts:
* **Query metadata**: Lines starting with `@` that define the contracts and events.
* **Query body**: The SQL query itself.

```sql

@ethereum::usdc(0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48)
@usdc::Transfer(address indexed from, address indexed to, uint256 value)

SELECT * FROM ethereum.usdc.Transfer
ORDER BY block_num DESC, log_idx DESC
LIMIT 10;
```

## Query Metadata

The metadata is defined using the following formats:

* `@<network>::<contract_alias>(0x<contract_address>)`: Creates an alias for a contract on a given network. Note that you cannot query `<network>.<contract_alias>` directly, you must also define at least one event.
* `@<contract_alias>::<Event>(signature)`: Defines an event for an already-declared contract. Multiple events can be defined for the same contract. The signature must follow the Ethereum ABI specification. Once defined, you can query it with `FROM <network>.<contract_alias>.<Event>`.
* `@<network>::<contract_alias>::<Event>(params)`: Same as above but explicitly includes the network.
* `@<network>::<Event>(params)`: Defines an event across the entire network, useful when you want to query events emitted by multiple contracts.


<network> can take the following values:
* ethereum
* arbitrum
* base
* polygon
* op
* unichain
* linea
* mantle
* monad
* scroll
* plasma
* gnosis

Event signatures follow standard Ethereum ABI format. Examples of valid `<Event>(params)` signatures include, but are not limited to:
* Transfer(address indexed from, address indexed to, uint256 value)
* Upgraded(address implementation)
* NameRegistered(string name, bytes32 indexed label, address indexed owner, uint256 baseCost, uint256 premium, uint256 expires)
* ModifyLiquidity(bytes32 indexed id, address indexed sender, int24 tickLower, int24 tickUpper, int256 liquidityDelta, bytes32 salt)
* SupportedInterfacesRegistered(address indexed operator_, bytes4[] interfaceIds_)

`<contract_alias>` must be a lowercase identifier that starts with a letter and can contain letters, digits, and underscores. Examples:
* `usdc`: valid
* `some_contract_name`: valid
* `contract1`: valid
* `some-contract`: invalid — hyphens are not allowed.
* `1usdc`: invalid — cannot start with a digit.

## Table Schema

The underlying database is ClickHouse. Every table includes the following common fields:

* block_num: UInt64
* block_timestamp: DateTime64(0, 'UTC')
* block_hash: FixedString(32)
* log_idx: UInt64
* tx_idx: UInt64
* tx_hash: FixedString(32)

In addition to these common fields, the table includes one column for each parameter in the event signature, using the same name.

Solidity types are mapped to ClickHouse types as follows:
* `bool` -> `Bool`
* `address` -> `String`
* `uint256` -> `UInt256`
* All `uint` types (`uint8` through `uint256`) -> `UInt256`
* `bytes` -> `String`

## Query Format

The query format follows standard ClickHouse SQL syntax, with one extension: you can compare `address` and `bytes` fields directly against hex literals without casting. See the examples below.

```
@ethereum::usdc(0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48)
@usdc::Transfer(address indexed from, address indexed to, uint256 value)

SELECT * FROM ethereum.usdc.Transfer
WHERE to = 0x852f57dd17edbb0bedae8c55dd4b20feb3133089
LIMIT 10;
```

```
@ethereum::polygon_agglayer_unified_bridge(0x2a3dd3eb832af982ec71669e178424b10dca2ede)
@polygon_agglayer_unified_bridge::NewWrappedToken(uint32 originNetwork, address originTokenAddress, address wrappedTokenAddress, bytes metadata)

SELECT * FROM ethereum.polygon_agglayer_unified_bridge.NewWrappedToken
WHERE metadata = 0x000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000000120000000000000000000000000000000000000000000000000000000000000006415354455354000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000034153540000000000000000000000000000000000000000000000000000000000
LIMIT 10;
```


## Query Examples


Daily circulating supply of PyUSD on Arbitrum. The supply is calculated by summing the `value` of `SupplyIncreased` events and subtracting the `value` of `SupplyDecreased` events. This token uses 6 decimals.
```
@arbitrum::pyusd(0x46850ad61c2b7d64d08c9c754f45254596696984)
@pyusd::SupplyIncreased(address indexed to, uint256 value)
@pyusd::SupplyDecreased(address indexed from, uint256 value)

SELECT
  i.day,
  sum(sum(i.net_increase) - sum(coalesce(d.net_decrease, 0))) OVER (ORDER BY i.day ASC)  / 1e6 AS circulating_supply
FROM (
  SELECT
    date_trunc('day', block_timestamp) AS day,
    sum(value) AS net_increase
   FROM arbitrum.pyusd.SupplyIncreased
   GROUP BY date_trunc('day', block_timestamp)
  ) i
  LEFT JOIN (
  SELECT
    date_trunc('day', block_timestamp) AS day,
    sum(value) AS net_decrease
    FROM arbitrum.pyusd.SupplyDecreased
    GROUP BY date_trunc('day', block_timestamp)
  ) d ON i.day = d.day
GROUP BY i.day
ORDER BY i.day ASC;
```

Most popular Uniswap V4 pools ranked by number of swaps.
```
@ethereum::uniswapv4(0x000000000004444c5dc75cb358380d2e3de08a90)
@uniswapv4::Swap(bytes32 indexed id, address indexed sender, int128 amount0, int128 amount1, uint160 sqrtPriceX96, uint128 liquidity, int24 tick, uint24 fee)

SELECT
    id as pool_id,
    count(*) AS swap_count
  FROM ethereum.uniswapv4.Swap
  GROUP BY id
  ORDER BY swap_count DESC
  LIMIT 10;
```

Current USDC balance on Ethereum for a given address.
```
@ethereum::usdc(0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48)
@usdc::Transfer(address indexed from, address indexed to, uint256 value)

SELECT
  ( sumIf(value, `to`   = 0xeb2629a2734e272bcc07bda959863f316f4bd4cf)
  - sumIf(value, `from` = 0xeb2629a2734e272bcc07bda959863f316f4bd4cf)
  ) / 1e6 AS usdc_balance
FROM ethereum.usdc.Transfer
WHERE
  to = 0xeb2629a2734e272bcc07bda959863f316f4bd4cf OR
  from = 0xeb2629a2734e272bcc07bda959863f316f4bd4cf
```

USDC balance over time on Ethereum for a given address.
```
@ethereum::usdc(0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48)
@usdc::Transfer(address indexed from, address indexed to, uint256 value)

SELECT
  block_timestamp,
  delta_usdc,
  sum(delta_usdc) OVER (ORDER BY block_num, log_idx) AS balance_usdc
FROM
(
  SELECT
    block_num,
    log_idx,
    block_timestamp,
    (toFloat64(
        if(`to`   = 0xeb2629a2734e272bcc07bda959863f316f4bd4cf,  value, 0)
      - if(`from` = 0xeb2629a2734e272bcc07bda959863f316f4bd4cf,  value, 0)
    ) / 1e6) AS delta_usdc
  FROM ethereum.usdc.Transfer
  WHERE
    to   = 0xeb2629a2734e272bcc07bda959863f316f4bd4cf OR
    from = 0xeb2629a2734e272bcc07bda959863f316f4bd4cf)
ORDER BY block_num, log_idx;
```


Daily count of `Upgraded` events across the entire Ethereum network for the full history.
```
@ethereum::Upgraded(address implementation)

SELECT 
  toDate(block_timestamp) AS time,
  count() AS daily_upgrades
FROM ethereum.Upgraded
GROUP BY toDate(block_timestamp)
ORDER BY max(block_num) ASC;
```


## API Usage

The API accepts a JSON body with two fields:
* **signatures**: An array of metadata lines (the `@` declarations).
* **query**: The SQL query string.

```
curl --location 'https://api.bilinearlabs.io/api/query' \
  --header 'Content-Type: application/json' \
  --data '{
    "signatures": ["@ethereum::usdc(0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48)", "@usdc::Transfer(address indexed from, address indexed to, uint256 value)"],
    "query": "SELECT * FROM ethereum.usdc.Transfer ORDER BY block_num DESC, log_idx DESC LIMIT 2"
  }'
```

Example response:
```
{
  "result": {
    "data": [
      {
        "block_num": "24533439",
        "block_timestamp": "2026-02-25 10:49:59",
        "contract": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
        "from": "0xd1941733908c5bd5c1e83b54ce0f18ce88342a62",
        "log_idx": "433",
        "to": "0xe2244d0623a11763500604487f591ec930aa6133",
        "tx_hash": "0x2132e41e71ca89f8fb155a07aafaf96d556fff22d0fe2c4863cadd73b0ec5526",
        "value": "126"
      },
      {
        "block_num": "24533439",
        "block_timestamp": "2026-02-25 10:49:59",
        "contract": "0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
        "from": "0x1c8039d7cf30dbab0980bc369bf3bbfe179e7887",
        "log_idx": "429",
        "to": "0xd055eca292d4e7450aa1379ca83b565fa77ba403",
        "tx_hash": "0x2143d4f9339d1f66aed02711db3337cb2f3f160d223de504e66411dbbe600ec9",
        "value": "235"
      }
    ],
    "meta": [
      {
        "name": "tx_hash",
        "type": "String"
      },
      {
        "name": "log_idx",
        "type": "UInt64"
      },
      {
        "name": "contract",
        "type": "String"
      },
      {
        "name": "block_num",
        "type": "UInt64"
      },
      {
        "name": "block_timestamp",
        "type": "DateTime64(0, 'UTC')"
      },
      {
        "name": "from",
        "type": "Nullable(String)"
      },
      {
        "name": "to",
        "type": "Nullable(String)"
      },
      {
        "name": "value",
        "type": "UInt256"
      }
    ],
    "rows": 2,
    "rows_before_limit_at_least": 1458,
    "statistics": {
      "bytes_read": 69237582,
      "elapsed": 0.090577,
      "rows_read": 909312
    }
  },
  "rate_limit": {
    "limit": 0,
    "used": 0,
    "remaining": 0,
    "resets_at": ""
  }
}
```


<!-- TODO: This section is unfinished.
Paths to browse in the navigator:
/curated/chain/tag
/curated/chain/0x

TODO: Review types. Missing tuples, structs, and others.
-->