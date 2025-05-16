# Omni SDK

Omni SDK is a JavaScript library to interact with HOT Bridge - a protocol for minting omni-assets within the HOT OMNI Balance smart contract.

See how HOT Bridge works [here](https://hot-labs.gitbook.io/hot-protocol/vqtWI3SVupbKsdCP1xgA/omni-tokens).

Omni SDK key features include:

- Deposit tokens (from NEAR and EVM)
- Withdraw tokens (to NEAR and EVM)
- Handle pending withdrawals
- View HOT Bridge token balances

Coming soon:

- Support for Solana, TON, and Stellar — connect, deposit, and withdraw.

---

## Install

The sdk requires `nodejs` version 20.18.0 or higher.

```bash
npm install @hot-labs/omni-sdk
```

---

## Initialize

To initialize the Omni SDK and interact with HotBridge from your frontend, create a new instance of the HotBridge class:

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

We are instantiating a `HotBridge`, and defining a dictionary of `chain-id` to `rpc-url` mappings. This is later used by the bridge to interact with different chains.

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

Notice that we are setting the `executeNearTransaction` function on the bridge instance. In our example, the function is in turn calling an instance of the `walletSelector` to sign and send the transaction on NEAR.

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

---

## Deposit

Lets take a look at how to deposit tokens into the Omni SDK from different chains.

### NEAR

To deposit tokens from the NEAR chain, you need to call the `bridge.near.depositToken` method,passing the following parameters:

- `getAddress`: function that returns user's NEAR accountId
- `getIntentAccount`: function that returns user's NEAR Intent accountId
- `sendTransaction`: function that sends a transaction on NEAR (see [the near react hook](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/hooks/near.ts#L82))
- `amount`: amount of tokens (without denomanation), e.g. 1000 for 0.001 USDT (usdt.tether-token.near)
- `token`: token contract, eg. `usdt.tether-token.near`.

```js
const nearSigner = useNearWallet();

// ...

const handleDeposit = async (e: any) => {
  // ...

  await bridge.near.depositToken({
    getAddress: async () => nearSigner.accountId!,
    getIntentAccount: async () => nearSigner.intentAccount!,
    sendTransaction: nearSigner.sendTransaction,
    amount: BigInt(amount),
    token: token,
  });

  // ...
}
```

>_See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/DepositComponent.tsx#L46)._

<details>

<summary> Where are the tokens being deposited? </summary>

When you deposit tokens from the NEAR chain, the tokens are actually being deposited into the NEAR Intents smart contract.

It is important to note that all tokens will be deposited as native `NEP141`, including native NEAR which will be converted to its wrapped representation (`wrap.near`)

</details>

### EVM chains
 
When depositing tokens process from EVM chains we need to do two things, the first is depositing the tokens on what we call a `locker` contract on the native chain, and the second is to let know the Omni contract on NEAR that a deposit has been made.

All the complexity is hidden behind the SDK, so we just need to call `bridge.evm.deposit` method, followed by `bridge.finishDeposit`.

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

All complexity of the deposit is handled by the `bridge.evm.deposit`, including handling token approvals in the EVM chain if necessary.

The same way, `bridge.finishDeposit` gathers all the deposit data from the EVM chain, and presents a proof of deposit to the HOT Bridge, and then confirms that the deposit was credited by checking the new HOT OMNI Balance.

>_See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/DepositComponent.tsx#L46)._

<details>

<summary> How are tokens represented on NEAR?</summary>

On NEAR, the `v2_1.omni.hot.tg` contract follows what is known as the [MultiToken standard](https://github.com/shipsgold/NEPs/blob/master/specs/Standards/MultiToken/Core.md).

When we deposit tokens from an EVM chain, and present a proof of deposit, the Omni contract proceeds to mint a new token on the NEAR chain, which represent the deposited token. Particularly, tokens follow the form: `nep245:v2_1.omni.hot.tg:<chain_id>_<base58_encoded_address>` - e.g. `nep245:v2_1.omni.hot.tg:1_43mFVfjVcGGEbHurXm8UFVkHey6W` represents the USDT token on Ethereum.

Immediately after the tokens are minted, they are deposited into NEAR Intents on behalf of the user, who retains full ownership and control of these assets.

</details>

---

## Withdraw

### NEAR

To withdraw tokens on NEAR network you need just to call `withdrawToken` method of Omni SDK. For NEAR chain that method just asks user to sign withdrawing intent message and calls withdrawing transaction directly on NEAR Intents. It requires passing the following parameters:

- `getIntentAccount`: function that returns user's NEAR Intent accountId
- `signIntent`: function that leverages Wallet Selector's signMessage method to sign intent's payload (see [the near react hook](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/hooks/near.ts#L60))
- `receiver`: a wallet address to withdraw assets to
- `amount`: amount of tokens (without denomanation), e.g. 1000 for 0.001 USDT (usdt.tether-token.near)
- `chain`: chain id
- `token`: token contract, eg. `usdt.tether-token.near`.

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

You will just make a call to `withdraw` the token directly to your account

</details>

### EVM chains

To withdraw on Evm chain you should also call `withdrawToken` method of Omni SDK and pass the same parameters as for NEAR chain. Under the hood the method:

1. Check if there is unfinished withdrawing process. User can have only one unfinished withdrawal process
2. Ask user to sign withdrawing intent message and calls withdrawing transaction for HOT Omni Token representaion on NEAR Intents to Hot Balance smart contract
3. Checks if withdrawal was created and has not been claimed yet
4. Signs withdrawal data using HOT Bridge MPC service.

But to finish withrawing you need to call additionally `bridge.evm.withdraw` method. That method withdraws funds from corresponding locker contract.

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

Since tokens are actually deposited on NEAR Intents, the Omni SDK will call `withdaw_frunciton` to recover the MultiTokens... and then burn them? ... and then something? ... , which will end up in ... until you get the tokens.

</details>

---

## Swap

At the moment Omni SDK doesn't support exchanging functionality. To swap one token to another it's recommended to use [Near Intents](https://docs.near.org/tutorials/intents/swap) directly.

---

## Omni Contracts

| Chain    | Address                                            |
| -------- | -------------------------------------------------- |
| NEAR     | `v2_1.omni.hot.tg`                                 |
| Evm      | `0x233c5370CCfb3cD7409d9A3fb98ab94dE94Cb4Cd`       |
| Solana   | `hCofXYTiYHwCPpgVpLvd3VgpapmhqAeNU26bWZANmS8`      |
| TON      | `EQAbCbnq3QDZCN2qi3wu6pM6e1xrSHkCdtLLSqJnWDYRGhPV` |