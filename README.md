# EVM (Cheat Sheet)

> [!NOTE]  
> This is a cheat sheet on EVM-specific concepts (mostly notes-to-self). They are incomplete by
> default.

## Transaction types

### Dynamic fee transaction (`2`) or "EIP1559"

#### Transaction definition

This transaction is defined as follows:

```txt
0x02 || RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, value, data, accessList, signatureYParity, signatureR, signatureS])
```

Source:
[github.com/celo-org/txtypes](https://github.com/celo-org/txtypes#-dynamic-fee-transaction-2)

It was introduced on Ethereum during the Ethereum London hard fork
on [Aug, 5 2021](https://ethereum.org/en/history/#london) as specified
in [EIP-1559: Fee market change for ETH 1.0 chain](https://eips.ethereum.org/EIPS/eip-1559).

#### Transaction fees

There are a few parameters to consider when estimating transaction fees:

| Tx Parameter                                                                                                                                       | Description                                                                                                                                                                                                                                                                                                                                                                                                                            | Unit                         |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| [`gasLimit`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1914C13-L1915C93)              | The maximum amount of gas units that the transaction is allowed to consume.                                                                                                                                                                                                                                                                                                                                                            | units of gas (#)             |
| `baseFeePerGas`[^1]                                                                                                                                | Base fee (or minimum) per unit of gas paid for the transaction to be included in block. Burnt by the network.                                                                                                                                                                                                                                                                                                                          | ETH per unit of gas (in wei) |
| [`maxPriorityFeePerGas`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1890C13-L1891C125) | Incentive fee (or tip) per unit of gas paid to validators for the transaction to be included in a block. Minimum 1 wei, in practice 2 gwei-ish on Ethereum (see [live gas estimator](https://www.blocknative.com/gas-estimator)).                                                                                                                                                                                                      | ETH per unit of gas (in wei) |
| [`maxFeePerGas`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1878C13-L1879C111)         | Maximum fee per unit of gas the transaction is allowed to pay. This serves as an upper-bound, so if less is paid because the base fee is lower than expected, then at most the sum of $\text{base fee} + \text{incentive fee}$ is paid per unit of gas. This is a way a to protect users from paying too much because of inflated base fees. If this is lower than the current base fee, then the transaction can become unmarketable. | ETH per unit of gas (in wei) |

[^1]: `baseFeePerGas` is not a parameter specified by the user. It's a dynamic parameter given by the
network that can change from block to block.

There are useful RPC methods to help estimate transaction fees:

| RPC method                                                                                   | Helps with                                                                                                                                         | Parameters and response                                                                                                                                                                                               |
| -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [eth_estimateGas](https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_estimategas)    | [`gasLimit`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1914C13-L1915C93)              | Given a transaction object, returns how much gas is necessary to complete the specified transaction. The estimation is deterministic, because the EVM specifies the units of gas required by each computational step. |
| `block.baseFeePerGas`[^2]                                                                    | `baseFeePerGas`                                                                                                                                    | Base fee of the previous block. Since the base fee is dynamic, the base fee of the next block depends on the density of the previous block.                                                                           |
| [eth_maxPriorityFeePerGas](https://www.quicknode.com/docs/ethereum/eth_maxPriorityFeePerGas) | [`maxPriorityFeePerGas`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1890C13-L1891C125) | Incentive fee per unit of gas paid on the network, effectively $\text{total fee} - \text{base fee}$ or `eth_gasPrice` - `block.baseFeePerGas`.                                                                        |
| [eth_gasPrice](https://www.quicknode.com/docs/ethereum/eth_gasPrice)                         | [`maxFeePerGas`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1878C13-L1879C111)         | Total fee (base fee + incentive fee) per unit of gas paid on the network at the moment. Useful to find out what incentive fee per unit of gas the market is currently paying.                                         |

[^2]: `block.baseFeePerGas` is not an RPC method. It's a field available on every past block (after
the London hard fork).

Once a transaction has been mined, a transaction receipt can be requested to inspect the actual
transaction fees paid. A transaction receipt contains useful information such as:

| Tx Parameter                                                                                                                          | Description                                                                                                                                                | Unit                         |
| ------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| [`gasUsed`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1970C13-L1971C171) | The actual amount of gas used to process the transaction. If the transaction has not yet been mined, this field will be `null`.                            | units of gas (#)             |
| [`effectiveGasPrice`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1995)    | The actual fee per unit of gas paid as calculated by $\text{baseFeePerGas} + min(\text{maxFeePerGas} - \text{baseFeePerGas}, \text{maxPriorityFeePerGas})$ | ETH per unit of gas (in wei) |

The specific RPC method for this is:

| RPC method                                                                                          | Helps with                                                                                                                                                                                                                                                               | Parameters and response                                         |
| --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------- |
| [eth_getTransactionReceipt](https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_estimategas) | [`gasUsed`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1970C13-L1971C171) [`effectiveGasPrice`](https://github.com/ethereum/execution-apis/blob/7907424db935b93c2fe6a3c0faab943adebe8557/graphql.json#L1995) | Given a transaction hash, returns the receipt of a transaction. |

#### Historical context

The Ethereum JSON-RPC spec attempts to remain backwards-compatible with how transactions were
created in the past. For that reason, RPC methods sometimes serve both a present and a past purpose.
That can make method naming, request formats, and request responses a little odd at this. To help,
here is some historical context from a Geth maintainer ([@karalabe](https://github.com/karalabe)):

> Geth's gas price oracle internally calculates the priority fees actually paid by the transactions.
> For the `eth_gasPrice` call, it will return the estimated `priority fee` + 1x `base fee` (which
> will essentially be the "current" full `gasPrice` that it estimated pre-London hardfork). For
> `eth_maxPriorityFeePerGas`, it will return the estimated `priority fee`. The user needs to set
> the `maxFeePerGas` either manually based on the tip, or Geth defaults to the `priority fee` + 2x
> `baseFee`.

Source: [github.com/ethereum/pm](https://github.com/ethereum/pm/issues/328#issuecomment-853612573)
(modified for readability)

> Geth's implementation currently:
>
> - `eth_gasPrice` **before** London uses the current algorithm (deliberately won't say)
> - `eth_gasPrice` **after** London will return the exact same number based on the total fees paid
>   (tip + base)
> - `eth_maxPriorityFeePerGas` **after** London will effectively return `eth_gasPrice - baseFee`
>
> Auto-fill details:
>
> - If the user doesn't specify either `gasPrice` or 1559 gas fields, we default to 1559 style
>   transactions
> - If the user doesn't specify `maxPriorityFeePerGas`, we default to the above estimation
> - If the user doesn't specify `maxFeePerGas`, we default to `maxPriorityFeePerGas + 2*baseFee`
>
> The rationale's behind this thought:
>
> - Users continuing to rely on legacy transactions and `eth_gasPrice` should not observe any
>   behavioral change. (Hence why I didn't detail how the GPO works, it works the same way as until
>   now).
> - Users stuffing `eth_gasPrice` into a 1559 tx for the priorityFee or maxFee will be handled
>   gracefully as if they just did a legacy transaction, no shooting yourself in the foot.
> - Advanced users who know 1559 is different and how can rely on the new endpoint
>   `eth_maxPriorityFeePerGas` which returns a value for the tip only (thus it won't be mined if
>   used as is without a baseFee + slack on top).
>
> Edit: Geth's gas price oracle retrieves the cheapest 3 transactions from the past X blocks, and
> uses the 60th percentile as the suggestion for the price. The 60th percentile ensures we're aiming
> for inclusion in 2-3 blocks, whereas looking at the minimums ensures we're trying to push prices
> downward instead of up.
>
> Edit2: X = 20 for full nodes and 2 for light clients (they need to retrieve the entire blocks to
> suggest a price)

Source: [github.com/ethereum/pm](https://github.com/ethereum/pm/issues/328#issuecomment-853234014)

### Dynamic fee transaction v2 (`123`) or "CIP64"

> [!NOTE]  
>  This transaction is not compatible with Ethereum and has one Celo-specific
> parameter: `feeCurrency`.

#### Transaction definition

This transaction is defined as follows:

```
0x7b || RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, gasLimit, to, value, data, accessList, feeCurrency, v, r, s])

```

It was introduced on Celo during
the [Celo Gingerbread hard fork](https://github.com/celo-org/celo-proposals/blob/8260b49b2ec9a87ded6727fec7d9104586eb0752/CIPs/cip-0062.md) on [Sep 26, 2023](https://forum.celo.org/t/mainnet-alfajores-gingerbread-hard-fork-release-sep-26-17-00-utc/6499) as
specified
in [CIP-64: New Transaction Type: Celo Dynamic Fee v2](https://github.com/celo-org/celo-proposals/blob/master/CIPs/cip-0064.md)

#### Transaction fees

There are a few parameters to consider when estimating transaction fees:

| Tx Parameter           | Description                                                                                                                                                                                                                                                                                                                                                                                                                         | Unit                                  |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| `gasLimit`             | The maximum amount of gas units that the transaction is allowed to consume.                                                                                                                                                                                                                                                                                                                                                         | units of gas (#)                      |
| `baseFeePerGas`[^1]    | Base fee (or minimum) per unit of gas paid for the transaction to be included in block.                                                                                                                                                                                                                                                                                                                                             | Fee currency per unit of gas (in wei) |
| `maxPriorityFeePerGas` | Incentive fee (or tip) per unit of gas paid to validators for the transaction to be included in a block. Minimum 1 wei, in practice blocks are also mined without tip on Celo.                                                                                                                                                                                                                                                      | Fee currency per unit of gas (in wei) |
| `maxFeePerGas`         | Maximum fee per unit of gas the transaction is allowed to pay. This serves as an upper-bound, so if less is paid because the base fee is lower than expected, then at most the sum of $\text{base fee} + \text{incentive fee}$ is paid per unit of gas. This is a way a to protect users from paying too much because of inflated base fees. If this is lower than the current base fee, then the transaction becomes unmarketable. | Fee currency per unit of gas (in wei) |

[^1]`baseFeePerGas` is not a parameter specified by the user. It's a dynamic parameter given by the
network that can change from block to block.

There are useful RPC methods to help estimate transaction fees:

| RPC method                              | Helps with             | Parameters and response                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| --------------------------------------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `eth_estimateGas`                       | `gasLimit`             | Given a transaction object, returns how much gas is necessary to complete the specified transaction. The estimation is deterministic, because the EVM specifies the units of gas required by each computational step.                                                                                                                                                                                                                                      |
| `eth_gasPrice[feeCurrency]`             | `baseFeePerGas`        | Returns base fee for the given currency [multiplied](https://github.com/celo-org/celo-blockchain/blob/master/eth/gasprice/gasprice.go#L73) by a [constant](https://github.com/celo-org/celo-blockchain/blob/master/eth/api_backend.go#L351) by default [set](https://github.com/celo-org/celo-blockchain/blob/master/eth/ethconfig/config.go#L58) to `2` and doesn't include tips. This can be overwritten by passing the `--rpc.gaspricemultiplier` flag. |
| `eth_maxPriorityFeePerGas[feeCurrency]` | `maxPriorityFeePerGas` | Returns the incentive fee for the given currency per unit of gas paid on the network.                                                                                                                                                                                                                                                                                                                                                                      |

[^2]`block.baseFeePerGas` is not an RPC method. It's a field available on every past block (after
the London hard fork).
