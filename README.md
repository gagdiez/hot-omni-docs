# Omni SDK

Omni SDK is a JavaScript library to interact with HOT Bridge - a protocol for minting omni-assets within the HOT OMNI Balance smart contract.

See how HOT Bridge works [here](https://hot-labs.gitbook.io/hot-protocol/vqtWI3SVupbKsdCP1xgA/omni-tokens).

Omni SDK key features include:

- Connect NEAR, EVM (currently deposit only to Intent account binded to NEAR wallet)
- Deposit token widget (from NEAR, EVM)
- Withdraw token widget (to NEAR, EVM)
- Find pending withdrawals and finish them
- View HOT Bridge tokens balances on Intents.

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

To initialize the Omni SDK and interact with HotBridge from your frontend you should create a new instance of HotBridge class:

```js
import { mainnet, base, arbitrum, optimism, polygon, bsc, avalanche } from "viem/chains";
import { HotBridge } from "@hot-labs/omni-sdk";

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
```

When you have your HotBridge initialized you can use Reach hooks to interact with it from React components:

```js
import { useEffect } from "react";
import { useNearWallet } from "./near";

// Initializing HotBridge instance here, see the code above

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

See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/hooks/bridge.ts).

---

## Deposit

### NEAR

Depositing tokens from NEAR requires just calling `bridge.near.depositToken` method. Under the hood the method is making a deposit NEAR tokens to NEAR Intents account.

```js
const nearSigner = useNearWallet();

// ...

const handleDeposit = async (e: any) => {
  // ...

  await bridge.near.depositToken({
    getAddress: async () => nearSigner.accountId!, // User's accountId
    getIntentAccount: async () => nearSigner.intentAccount!, // User's accountId on NEAR Intents (on NEAR it's the same as accountId)
    sendTransaction: nearSigner.sendTransaction, // method sendTransaction from to actually make a transaction on NEAR (see near react hook)
    amount: BigInt(amount), // Amount of tokens (without denomanation), e.g. 1000 for 0.001 USDT (usdt.tether-token.near)
    token: token, // Token contract, eg. usdt.tether-token.near
  });

  // ...
}
```

See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/DepositComponent.tsx#L46).

### EVM chains

Depositing tokens from EVM chains requires calling `bridge.evm.deposit` method to make a deposit on NEAR Intents through the corresponding omni bridge locker contract (including token contract approving if it's neccessary). When deposit transaction is sent it's neccessary to finish deposit calling `bridge.finishDeposit` method. That method signs deposit data using HOT Bridge MPC service and confirms that deposit was made successfully by calling HOT OMNI Balance smart contract.

```js
const evmSigner = useEvmWallet();

// ...

const handleDeposit = async (e: any) => {
  // ...

  const deposit = await bridge.evm.deposit({
    getAddress: async () => evmSigner.address!, // User's EVM address
    getIntentAccount: async () => nearSigner.intentAccount!, // User's NEAR Intent account
    sendTransaction: evmSigner.sendTransaction, // method sendTransaction to actually make a transaction on EVM (see evm react hook)
    amount: BigInt(amount),
    chain: network,
    token: token,
  });

  await bridge.finishDeposit(deposit);

  // ...
}
```

See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/DepositComponent.tsx#L46).

---

## Withdraw

### NEAR

To withdraw tokens on NEAR network you need just to call `withdrawToken` method of Omni SDK. For NEAR chain that method just asks user to sign withdrawing intent message and calls withdrawing transaction directly on NEAR Intents.

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

See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/WithdrawComponent.tsx#L47).

### EVM chains

To withdraw on Evm chain you should also call `withdrawToken` method of Omni SDK. Under the hood the method:

1. Check if there is unfinished withdrawing process. User can have only one unfinished withdrawal process
2. Ask user to sign withdrawing intent message and calls withdrawing transaction for HOT Omni Token on NEAR Intents to Hot Balance smart contract
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

See the full code [here](https://github.com/hot-dao/omni-sdk/blob/main/demo-ui/src/components/WithdrawComponent.tsx#L47).

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