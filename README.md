# Omni Chain - By Hot Wallet

The Omni Chain is an abstraction layer over multiple chains that allows you to handle tokens and assets in a unified way.

In a nutshell, the Omni Chain works by having a smart contract deployed on each of the chains it operates (TON, Solana, NEAR, Ethereum), where users can deposit their tokens and assets, and they are all controlled through a single smart contract deployed on NEAR.

This way, whenever a user deposits tokens on Solana, they are provided with its Omni representation on NEAR, which they can exchange for other Omni tokens, and withdraw on any other chain.

---

## Deposit

To deposit tokens on the Omni Chain, you need to send them to the smart contract deployed on the chain you are using.

| Chain    | Address |
|----------|---------|
| TON      | `0:`    |
| Solana   | `0:`    |
| NEAR     | `0:`    |
| Ethereum | `0:`    |

<details>
<summary>Deposit using the Omni SDK</summary>

```js

```

</details>

<details>
<summary>Deposit from a NEAR Smart Contract </summary>

```rust

```

</details>


---

## Transfer

To transfer tokens on the Omni Chain, you need to interact with the smart contract deployed on NEAR.

<details>
<summary>Deposit using the Omni SDK</summary>

```js

```

</details>

<details>
<summary>Deposit from a NEAR Smart Contract </summary>

```rust

```

</details>


---

## Withdraw

To withdraw tokens from the Omni Chain, you first need to interact with the Omni Smart Contract on NEAR, and then interact with the smart contract on the chain you want to withdraw from.


### Step One: Ask to Withdraw on NEAR

<summary>Withdraw using the Omni SDK</summary>

```js

```

</details>

<summary>Withdraw from a NEAR Smart Contract</summary>

```js

```

</details>

> [!NOTE]
> If you want to withdraw your tokens directly on NEAR, you can simply call `withdraw_on_near` on the Omni smart contract, and you can then skip the next steps.


### Step Two: Withdraw on the Chain

Once you have the Omni token on the chain you want to withdraw from, you can interact with the smart contract to withdraw the tokens.

> [!NOTE]
> If you want to withdraw your tokens directly on NEAR, you can simply call `withdraw_on_near` on the Omni smart contract, and you can then skip the next steps.


#### Solana

In Solana, you will need to interact with the program deployed on `...` to withdraw the tokens.

```js

```

#### TON

Same

#### Ethereum

Same
