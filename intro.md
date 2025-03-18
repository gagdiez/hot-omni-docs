# HOT Omni Protocol - By Hot Wallet

HOT Omni Protocol is an abstraction layer over multiple chains that allows you to handle tokens and assets in a unified way.

In a nutshell, HOT Omni Protocol works by having a smart contract deployed on each of the chains it operates (TON, Solana, NEAR, Ethereum), where users can deposit their tokens and assets, and they are all controlled through a single smart contract deployed on NEAR.

This way, whenever a user deposits tokens on Solana, they are provided with its Omni representation on NEAR, which they can exchange for other Omni tokens, and withdraw on any other chain.

---

## Omni SDK

Omni SDK is a JavaScript library by HOT Wallet team to interact with HOT Omni Protocol from backend. The library provides methods to deposit, transfer and withdraw tokens on the HOT network.

To initialize the Omni SDK you should provide credentials to your wallets on different chains. Those wallets will be used to make deposits/withdrawals to/from HOT Omni Protocol.

```js
import {
  OmniService,
  NearSigner,
  TonSigner,
  SolanaSigner,
  EvmSigner,
} from "@hot-wallet/omni-sdk";

const omni = new OmniService({
  near: new NearSigner(process.env.NEAR_ACCOUNT_ID, process.env.NEAR_PRIVATE_KEY),
  ton: new TonSigner(process.env.TON_PRIVATE_KEY, process.env.TON_WALLET_TYPE, process.env.TON_API_KEY),
  solana: new SolanaSigner(process.env.SOLANA_PRIVATE_KEY, [process.env.SOLANA_RPC]),
  evm: new EvmSigner(process.env.EVM_PRIVATE_KEY),
});
```

---

## Deposit

To deposit tokens on HOT Omni Protocol, you need to send them to the smart contract deployed on the chain you are using.

For all chains other than NEAR, you will need to provide an attestation of the transfer to HOT Omni Protocol smart contract on NEAR.


### Step One: Fungible Token Transfer

{% tabs %}

{% tab title="Omni SDK" %} 

  ```js
  const solanaUsdtDeposit = async () => {
    const USDT = omni.token(TokenId.USDT);
    const input = await USDT.input(Network.Solana, 0.01);
    await omni.depositToken(input);
  };

  solanaUsdtDeposit();
  ```

{% endtab %}

{% tab title="NEAR" %} 

In NEAR, you only need to transfer your Fungible Tokens to the smart contract deployed at `v1-1.omni.hot.tg`.

> [!Warning]
> Do not make a direct NEAR transfer, instead you will need to transfer the `Wrapped NEAR` (`wrap.near`) token to the Omni Chain contract.

<details>

<summary>Deposit from a backend</summary>

```js
import { connect, keyStores, KeyPair } from "near-api-js";
import { baseEncode } from "@near-js/utils";
import { Buffer } from 'buffer';
import { getBytes, sha256 } from "ethers";

const TGAS = 1000000000000n;

const privateKey = process.env.NEAR_PRIVATE_KEY;
const accountId = process.env.NEAR_ACCOUNT_ID;

const myKeyStore = new keyStores.InMemoryKeyStore();
const keyPair = KeyPair.fromString(privateKey);

const connectionConfig = {
  networkId: "mainnet",
  keyStore: myKeyStore,
  nodeUrl: "https://rpc.mainnet.near.org",
};
const nearConnection = await connect(connectionConfig);
const account = await nearConnection.account(accountId);

const getOmniAddress = (accountId) => {
  return baseEncode(getBytes(sha256(Buffer.from(accountId, "utf8"))));
}

const tokenAmount = 0.01;
const tokenDecimals = 6;

const args = {
  contractId: 'usdt.tether-token.near',
  methodName: "ft_transfer_call",
  args: {
    amount: String(BigInt(tokenAmount * 10**tokenDecimals)),
    receiver_id: 'v1-1.omni.hot.tg',
    msg: getOmniAddress(accountId) },
  deposit: 1n,
  gas: 80n * TGAS,
};

await account.functionCall(args);
```

</details>

<details>

<summary>Deposit from a NEAR Smart Contract </summary>

```rust
TODO
```

</details>

{% endtab %}

{% tab title="Solana" %}

<details>

<summary>Deposit from a backend</summary>

We don't provide the entire code on how to prepare and send deposit instructions to Solana from the server side because of its complexity. You can see how it's implemented in Omni SDK [here](https://github.com/hot-dao/omni-sdk/blob/main/src/omni-chain/solana/index.ts#L132).

But there are some key points:

1. You need to get a last deposit nonce from HOT metawallet on Solana
2. You need to send instructions on Solana metawallet to generate a new nonce
3. You need to wait untill a new deposit nonce is generated
4. You need to send tokens on Solana metawallet.

</details>


{% endtab %}

{% tab title="TON" %}

You need to ...


{% endtab %}

{% endtabs %}


### Step Two: Attestation

Once you have transferred your tokens to one of the protocol wallets, you will need to provide an attestation of the transfer to the HOT Omni Protocol smart contract on NEAR.

> [!NOTE]
> If you transferred your tokens from the NEAR chain, you can skip this step.


{% tabs %}

{% tab title="Solana" %} 

After you sent tokens on HOT's solana wallet, you need to:

1. Wait for receipt of the sent transaction
2. Check on NEAR smart contract if the deposit was already used.
3. If it's not, you need to get a deposit signature from HOT Omni API
4. Use this signature to finish your deposit by calling `deposit` method on NEAR smart contract.

{% endtab %}

{% endtabs %}


---

## Transfer

To transfer tokens on the Omni Chain, you need to interact with the smart contract deployed on NEAR.

### Transfer to user
To transfer assets between users within HOT network you just need to call the `omni_transfer` method on NEAR smart contact and pass following arguments:

- amount
- token_id
- account_id
- receiver_id

### Transfer to smart contract and call a method
Similar to transferring fungible tokens to smart contract on NEAR you can transfer tokens using HOT network. Call `omni_transfer_call` method and pass the arguments:

- amount
- chain_id
- token_id
- account_id
- receiver_id
- callback_account_id


---

## Withdraw

To withdraw tokens from the Omni Chain, you first need to interact with the Omni Smart Contract on NEAR, and then interact with the smart contract on the chain you want to withdraw from.

{% tabs %}
{% tab title="Omni SDK" %} 

  ```js
  const solanaUsdtWithdraw = async () => {
    const USDT = omni.token(TokenId.USDT);
    const input = await USDT.output(Network.Solana, 0.01);
    await omni.withdrawToken(output);
  };

  const finishWithdrawals = async () => {
    const pendings = await omni.getActiveWithdrawals();
    for (let pending of pendings) {
      await omni.finishWithdrawal(pending.nonce);
    }
  };

  solanaUsdtWithdraw();
  ```

{% endtab %}
{% endtabs %}

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

To withdraw tokens on Solana you need:

1. Call `withdraw` method on NEAR smart contract. It will create a withdrawal request and decreased your Omni balance.
2. Get a nonce and amount from near transaction receipts.
3. Check on NEAR smart contract if nonce is expired or already used.
4. If it's not, get a signature for that withdrawal request from HOT Omni API
5. Use signature to send instructions on Solana liquidity contract and withdraw funds on your wallet.

If the withdrawal request is expired:
1. Wait until the required time limit has passed and the refund becomes available.
2. Get a signature from HOT Omni API to refund your Omni balance.
3. Call `refund` method on NEAR smart contract to finish refund in restore your asset's balance.

> [!Warning]
> Any actions on your withdrawal request are available only for the last nonce. It means once you created a withdrawal request you have to finish or refund it before you can create a new one. This is due to the specifics of how smart contracts work in Solana.

<!-- In Solana, you will need to interact with the program deployed on `...` to withdraw the tokens. -->

```js

```

#### TON

> [!Warning]
> Any actions on your withdrawal request are available only for the last nonce. It means once you created a withdrawal request you have to finish or refund it before you can create a new one. This is due to the specifics of how smart contracts work in Ton.

#### Ethereum

Same
