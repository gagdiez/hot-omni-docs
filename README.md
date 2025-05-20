# Omni SDK - Frontend Integration

Omni SDK is a JavaScript library to interact with HOT Bridge - a protocol for minting and handling omni-assets. Find how HOT Bridge works [here](https://hot-labs.gitbook.io/hot-protocol/vqtWI3SVupbKsdCP1xgA/omni-tokens).

Omni SDK key features include:

- Deposit tokens (from NEAR and EVM)
- Withdraw tokens (to NEAR and EVM)
- Handle pending withdrawals
- View HOT Bridge token balances

## Supported Chains

Here is a non exhaustive list of Omni contracts:

| Chain Name       | Chain Id         |
|------------------|------------------|
| NEAR             | 1010             |
| Ethereum         | 1                |
| Other EVM Chains | Official ChainId |
| Solana           | 1001             |
| TON              | 1111             |
| Aurora           | 131316554        |

Coming soon:

- Support for Stellar — connect, deposit, and withdraw.

---

## Install

The sdk requires `nodejs` version 20.18.0 or higher.

```bash
npm install @hot-labs/omni-sdk
```

---

## Initialize

To initialize the Omni SDK and interact with the HotBridge from your frontend, we need to create a new instance of the HotBridge class:

```js
import { useEffect } from "react";
import { mainnet, base, arbitrum, optimism, polygon, bsc, avalanche } from "viem/chains";
import { HotBridge } from "@hot-labs/omni-sdk";
import { useNearWallet } from "./near";

export const bridge = new HotBridge({
  evmRpc: {
    8453: base.rpcUrls.default.http as any,
    42161: arbitrum.rpcUrls.default.http as any,
    10: optimism.rpcUrls.default.http as any,
    137: polygon.rpcUrls.default.http as any,
    56: bsc.rpcUrls.default.http as any,
    43114: avalanche.rpcUrls.default.http as any,
    1: mainnet.rpcUrls.default.http as any,
  },

  executeNearTransaction: async () => {
    throw "executor not implemented";
  },
});

// React hook to use the bridge instance
export const useBridge = () => {
  const nearWallet = useNearWallet();

  useEffect(() => {
    bridge.executeNearTransaction = async (tx) => {
      const hash = await nearWallet.sendTransaction(tx);
      return { sender: nearWallet.accountId!, hash };
    };
  }, [nearWallet.accountId]);

  return { bridge };
};
```
> _See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/hooks/bridge.ts)._

Let's break down the code so we can understand what is happening.

### Instantiating the Bridge

We are instantiating a `HotBridge`, and defining a dictionary of `chain-id` to `rpc-url` with the networks that we want to use:

```js
export const bridge = new HotBridge({
  evmRpc: {
    8453: base.rpcUrls.default.http as any,
    ...
    1: mainnet.rpcUrls.default.http as any,
  },

  executeNearTransaction: async () => { throw "..."; },
});
```

### The Hook

In order for other components to use the bridge, we are creating a react hook.

```js
// Reach hooks to use the bridge instance
export const useBridge = () => {
  const nearWallet = useNearWallet();

  useEffect(() => {
    bridge.executeNearTransaction = async (tx) => {
      const hash = await nearWallet.sendTransaction(tx);
      return { sender: nearWallet.accountId!, hash };
    };
  }, [nearWallet.accountId]);

  return { bridge };
};
```

Notice that we are setting the `executeNearTransaction` function on the bridge instance. This function tells the SDK how to make a transaction on NEAR. 

Since we are working on a frontend, our example executes the functions by calling an instance of the `NEAR Wallet Selector`.

---

## Deposit

The SDK makes it very simply to deposit tokens into the Hot Bridge, both from NEAR and EVM chains. Lets take a look at how it works for each chain.

### NEAR

To deposit tokens from the NEAR chain, we need to call the `bridge.near.depositToken` method, passing the following parameters:

- `getAddress`: a function that returns our NEAR account
- `getIntentAccount`: a function that returns the account that will receive the tokens on NEAR
- `sendTransaction`: function that sends a transaction on NEAR (see [the near react hook](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/hooks/near.ts#L82))
- `token`: the token we want to deposit, e.g. `usdt.tether-token.near`.
- `amount`: amount of tokens (in units), e.g. 1000 for 0.001 USDT

```js
const nearSigner = useNearWallet();

// ...

const handleDeposit = async (e: any) => {
  // ...

  await bridge.near.depositToken({
    getAddress: async () => nearSigner.accountId!,
    getIntentAccount: async () => nearSigner.intentAccount!,
    sendTransaction: nearSigner.sendTransaction,
    token: token,
    amount: BigInt(amount),
  });

  // ...
}
```

>_See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/DepositComponent.tsx#L46)._

<details>

<summary> Where are the tokens being deposited? </summary>

When depositing tokens on the NEAR chain, the Omni SDK will actually call the `ft_transfer_call` on the `token`, asking to deposit the tokens into the NEAR Intents smart contract.

Since the Intents contract only handles `NEP141` tokens, any NEAR token deposit will be converted to its wrapped representation (`wrap.near`).
</details>

### EVM chains

To deposit tokens from EVM chains, we need to call two methods:

- First, we call the `bridge.evm.deposit` method, which will deposit the tokens on the EVM chain and create a proof of deposit
- Second, we call the `bridge.finishDeposit` method, which will present the proof of deposit to the HOT Bridge and mint the tokens on NEAR 

Similar to the NEAR deposit, `bridge.evm.deposit` requires us to pass a couple of parameters:
- `getAddress`: function that returns user's EVM address
- `getIntentAccount`: function that returns user's NEAR Intent accountId, this is, the account that **will receive the tokens on NEAR**
- `sendTransaction`: function that sends a transaction on EVM (see the [evm react hook](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/hooks/evm.ts#L55))
- `amount`: amount of tokens (without denomanation)
- `chain`: chain id of the EVM chain
- `token`: token contract (e.g. `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48`)

```js
const evmSigner = useEvmWallet();

// ...

const handleDeposit = async (e: any) => {
  // ...

  const deposit = await bridge.evm.deposit({
    getAddress: async () => evmSigner.address!,
    getIntentAccount: async () => nearSigner.intentAccount!,
    sendTransaction: evmSigner.sendTransaction,
    amount: BigInt(amount),
    chain: network,
    token: token,
  });

  await bridge.finishDeposit(deposit);

  // ...
}
```

The`bridge.evm.deposit` will handle all the complexity of transferring the EVM tokens, including handling token approvals in the EVM chain if necessary.

The same way, `bridge.finishDeposit` will gather all the deposit data from the EVM chain, presents a proof of deposit to the HOT Bridge, and then confirm that the deposit was credited by checking the new HOT OMNI Balance.

>_See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/DepositComponent.tsx#L46)._

<details>

<summary> How are tokens represented on NEAR?</summary>

When depositing tokens from EVM chains two steps are performed.

The first is depositing the tokens on what we call a `locker` contract on the native chain, and the second is to let know the Omni contract on NEAR that a deposit has been made.

After the SDK presents the proof of deposit, the Omni contract proceeds to mint a new [MultiToken](https://github.com/shipsgold/NEPs/blob/master/specs/Standards/MultiToken/Core.md) on the NEAR chain, which represent the deposited token. Particularly, the minted tokens follow the form: `nep245:v2_1.omni.hot.tg:<chain_id>_<base58_encoded_address>` - e.g. `nep245:v2_1.omni.hot.tg:1_43mFVfjVcGGEbHurXm8UFVkHey6W` represents the USDT token on Ethereum.

Immediately after the tokens are minted, they are deposited into NEAR Intents on behalf of the user, who retains full ownership and control of these assets.

</details>

---

## Withdraw

The SDK makes it very simple to withdraw tokens from the Hot Bridge into any of the supported chains. The process differes slightly between NEAR and EVM chains, so let's take a look at each of them.

### NEAR

To withdraw tokens into the NEAR chain, we need to call the `bridge.withdrawToken` method, passing the following parameters:

- `getIntentAccount`: a function that returns our NEAR Intent account
- `signIntent`: a function that explains how to sign a message
- `chain`: The chain in which we want to withdraw the tokens
- `receiver`: a wallet address to withdraw assets to
- `token`: the token we want to withdraw, e.g. `usdt.tether-token.near`.
- `amount`: amount of tokens (in units), e.g. 1000 for 0.001 USDT

```js
// ...
const nearSigner = useNearWallet();

// ...

const handleWithdraw = async () => {
  // ...

  const result = await bridge.withdrawToken({
    getIntentAccount: async () => nearSigner.intentAccount!,
    signIntent: async (intent: any) => await nearSigner.signIntent(intent),
    receiver: receiver.trim(),
    amount: BigInt(amount),
    chain: network,
    token: token,
  });
  
  // ...
}
```

>_See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/WithdrawComponent.tsx#L47)._

<details>
<summary> Under the hood </summary>

When withdrawing tokens on the NEAR chain, the Omni SDK will follow the process for withdrawing tokens from NEAR Intents.

First, it will create an intent message that the user needs to sign. This message contains the information about the withdrawal, including the receiver address and the amount of tokens to withdraw.

Then, the Omni SDK will call the `execute_intent` method on the NEAR Intents smart contract, which will transfer the tokens from the user's NEAR Intent account to the receiver address.

</details>

### EVM chains

The withdrawing tokens process from EVM chains is a two-step process that basically reverses the depositing steps.

By calling `bridge.withdrawToken` method of the Omni SDK and pass the same parameters as for NEAR chain. Under the hood that method:

1. Checks if there is unfinished withdrawing process. User can have only one unfinished withdrawal process at a time.
2. Asks user to sign withdrawing intent message and calls withdrawing transaction for HOT Omni Token representation on NEAR Intents to Hot Balance smart contract
3. Checks if withdrawal was created and has not been fully claimed yet
4. Signs withdrawal data using HOT Bridge MPC service.

The second step is actually transferring assets from corresponding locker contract to receiver address. To do that you need to call the `bridge.evm.withdraw` method.

```js
const handleWithdraw = async () => {
  // ...

  await bridge.evm.withdraw({ sendTransaction: evmSigner.sendTransaction, ...result });

  // ...
}
```

>_See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/WithdrawComponent.tsx#L47)._

<details>

<summary> Under the hood </summary>

Since tokens are deposited on NEAR Intents as MultiTokens with format like `nep245:v2_1.omni.hot.tg:<chain_id>_<base58_encoded_address>`, the Omni SDK will firstly ask user to sign and execute `mt_withdraw` intent on Near Intents smart contract `intent.near` to withdraw those tokens to Hot Bridge smart contract.

Once that withdrawal is done, the Omni SDK gathers withdrawal information and signs it using Hot MPC service. Using that data and signature user eventually calls transaction that transfers tokens from corresponding locker contract to user's address by calling `bridge.evm.withdraw` method.
</details>


<!-- TODO ## Add Swap -->