# Omni Chain - By Hot Wallet

The Omni Chain is an abstraction layer over multiple chains that allows you to handle tokens and assets in a unified way.

In a nutshell, the Omni Chain works by having a smart contract deployed on each of the chains it operates (TON, Solana, NEAR, Ethereum), where users can deposit their tokens and assets, and they are all controlled through a single smart contract deployed on NEAR.

This way, whenever a user deposits tokens on Solana, they are provided with its Omni representation on NEAR, which they can exchange for other Omni tokens, and withdraw on any other chain.

---

## Deposit

To deposit tokens on the Omni Chain, you need to send them to the smart contract deployed on the chain you are using.

For all chains other than NEAR, you will need to provide an attestation of the transfer to the Omni Chain smart contract on NEAR.


### Step 1: Fungible Token Transfer
{% tabs %}

{% tab title="Omni SDK" %} 

{% endtab %}

{% tab title="NEAR" %} 

In NEAR, you only need to transfer your Fungible Tokens to the smart Omni Chain contract deployed at `...`


> [!Warning]
> Do not make a direct NEAR transfer, instead you will need to transfer the `Wrapped NEAR` (`wrap.near`) token to the Omni Chain contract.

<details>
<summary>Deposit from a NEAR Smart Contract </summary>

</details>

<details>

<summary>Deposit from a NEAR Smart Contract </summary>

```rust

```

</details>

{% endtab %}

{% tab title="Solana" %}



{% endtab %}

{% tab title="TON" %}

You need to ...

```js

````

{% endtab %}

{% endtabs %}


## Step 2: Attestation

Once you have transferred your tokens to the Omni Chain, you will need to provide an attestation of the transfer to the Omni Chain smart contract on NEAR.

> [!NOTE]
> If you transferred your tokens from the NEAR chain, you can skip this step.





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
