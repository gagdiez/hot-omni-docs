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

### Using Omni SDK

```js
const solanaUsdtDeposit = async () => {
  const USDT = omni.token(TokenId.USDT);
  const input = await USDT.input(Network.Solana, 0.01);
  await omni.depositToken(input);
};

solanaUsdtDeposit();
```

### Using blockchains directly
#### Step One: Fungible Token Transfer

{% tabs %}
{% tab title="NEAR" %} 

In NEAR, you only need to transfer your Fungible Tokens to the smart contract deployed at `v1-1.omni.hot.tg`.

> [!Warning]
> Do not make a direct NEAR transfer, instead you will need to transfer the `Wrapped NEAR` (`wrap.near`) token to the Omni Chain contract.

<details>

<summary>Deposit from a backend</summary>

```js
import { baseEncode } from "@near-js/utils";
import { Buffer } from 'buffer';
import { getBytes, sha256 } from "ethers";

const TGAS = 1000000000000n;

const tokenAmount = 1;
const tokenDecimals = 6;

const args = {
  contractId: 'usdt.tether-token.near',
  methodName: "ft_transfer_call",
  args: {
    amount: String(BigInt(tokenAmount * 10**tokenDecimals)),
    receiver_id: 'v1-1.omni.hot.tg',
    msg: baseEncode(getBytes(sha256(Buffer.from(accountId, "utf8")))),
  },
  deposit: 1n,
  gas: 80n * TGAS,
};

// "account" here is an instance of Account interface from near-api-js
await account.functionCall(args);
```

</details>

<details>

<summary>Deposit from a NEAR Smart Contract </summary>

```rust
fn get_omni_address(account_id: String) -> String {
    return bs58::encode(Sha256::digest(account_id.as_bytes())).into_string();
}

#[near]
impl Contract {
  pub fn deposit(&mut self, amount: String) -> Promise {
      usdt_contract::ext("usdt.tether-token.near".parse().unwrap())
          .with_static_gas(Gas::from_tgas(80))
          .with_attached_deposit(NearToken::from_yoctonear(1))
          .ft_transfer_call(
              amount,
              HOT_OMNI_CONTRACT.to_string(),
              get_omni_address(env::current_account_id().to_string()),
          )
          .then(
              Self::ext(env::current_account_id())
                  .with_static_gas(Gas::from_tgas(5))
                  .deposit_callback(),
          )
  }

  #[private]
  pub fn deposit_callback(
      &mut self,
      #[callback_result] call_result: Result<String, PromiseError>,
  ) -> String {
      if call_result.is_err() {
          log!("There was an error depositing on HOT Omni Contract");
          return "".to_string();
      }

      let deposited_amount: String = call_result.unwrap();
      deposited_amount
  }
}
```

</details>

{% endtab %}

{% tab title="Solana" %}

We don't provide the entire code on how to prepare and send deposit instructions to Solana from the server side because of its complexity. You can see how it's implemented in Omni SDK [here](https://github.com/hot-dao/omni-sdk/blob/main/src/omni-chain/solana/index.ts#L132).

But there are some key points:

1. You need to send instructions on Solana metawallet to generate a new nonce.
2. You need to wait untill a new deposit nonce is generated.
3. You need call a `nativeDeposit` or `tokenDeposit` method.

{% endtab %}

{% tab title="TON" %}

We don't provide the entire code on how to prepare and send deposit instructions to Ton from the server side because of its complexity. You can see how it's implemented in Omni SDK [here](https://github.com/hot-dao/omni-sdk/blob/main/src/omni-chain/ton/index.ts#L154).

But there are some key points:

1. You need to send deposit on Ton metawallet
2. Wait for a `transaction:success` event
3. Parse deposit data from the deposit transaction

{% endtab %}

{% tab title="Ethereum" %}

We don't provide the entire code on how to prepare and send deposit transaction to Ethereum from the server side because of its complexity. You can see how it's implemented in Omni SDK [here](https://github.com/hot-dao/omni-sdk/blob/main/src/omni-chain/evm/index.ts#L56).

But the key point here is sending a deposit transaction calling `deposit` method on Ethereum smart contract. 

{% endtab %}


{% endtabs %}


#### Step Two: Attestation

Once you have transferred your tokens to one of the protocol wallets, you will need to provide an attestation of the transfer to the HOT Omni Protocol smart contract on NEAR.

> [!NOTE]
> If you transferred your tokens from the NEAR chain, you can skip this step.


{% tabs %}

{% tab title="Solana" %} 

After you sent tokens on HOT's solana wallet, you need to:

1. Wait for receipt of the sent transaction and get `nonce`, `amount`, `receiver` and `token` parameters from receipt's logs.
2. Check on NEAR smart contract if the deposit was already used.
3. If it's not, you need to get a deposit signature from HOT Omni API
4. Use this signature to finish your deposit by calling `deposit` method on NEAR smart contract.
5. After finishing your deposit or if the deposit was already used, you need to call `clearDepositInfo` on Solana's metawallet.

{% endtab %}

{% tab title="Ton" %} 

After you sent tokens on HOT's ton wallet, you need to:

1. Check on NEAR smart contract if the deposit was already used.
2. If it's not, you need to get a deposit signature from HOT Omni API
3. Use this signature to finish your deposit by calling `deposit` method on NEAR smart contract.
4. After finishing your deposit or if the deposit was already used, you need to call `selfDestruct` on Ton's metawallet.

{% endtab %}

{% tab title="Ethereum" %} 

After you sent tokens on HOT's ethereum wallet, you need to:

1. Wait for receipt of the sent transaction
2. Get a deposit data from the receipt
3. Check on NEAR smart contract if the deposit was already used.
4. If it's not, you need to get a deposit signature from HOT Omni API
5. Use this signature to finish your deposit by calling `deposit` method on NEAR smart contract.

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

### Using Omni SDK

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

### Using blockchains directly

#### Step One: Ask to Withdraw on NEAR

<summary>Withdraw from a backend</summary>

```js
import { baseEncode } from "@near-js/utils";
import { Buffer } from 'buffer';
import { getBytes, sha256 } from "ethers";

const TGAS = 1000000000000n;

const tokenAmount = 1;
const tokenDecimals = 6;

const provider = new providers.JsonRpcProvider({ url });

const viewCallResult = await provider.query({
  request_type: "call_function",
  account_id: "usdt.tether-token.near",
  method_name: "storage_balance_of",
  args_base64: Buffer.from(JSON.stringify({ account_id: this.signers.near.accountId })).toString("base64"),,
  finality: finality,
});

const needReg = JSON.parse(Buffer.from(viewCallResult.result).toString());

const args = {
  contractId: "v1-1.omni.hot.tg",
  methodName: "withdraw_on_near",
  args: {
    account_id: baseEncode(getBytes(sha256(Buffer.from(accountId, "utf8")))),
    token_id: 9,
    amount: String(BigInt(tokenAmount * 10**tokenDecimals)),
  },
  deposit: needReg == null ? 5000000000000000000000n : 1n,
  gas: 80n * TGAS,
};

// "account" here is an instance of Account interface from near-api-js
await account.functionCall(args);
```

</details>

<summary>Withdraw from a NEAR Smart Contract</summary>

```rust
fn get_omni_address(account_id: String) -> String {
    return bs58::encode(Sha256::digest(account_id.as_bytes())).into_string();
}

#[near]
impl Contract {
  pub fn withdraw(&self, amount: U128, token_id: u8) -> Promise {
      let account_id = env::current_account_id();
      hot_omni_contract::ext("v1-1.omni.hot.tg".parse().unwrap())
          .with_static_gas(Gas::from_tgas(80))
          .with_attached_deposit(NearToken::from_yoctonear(1))
          .withdraw_on_near(
              get_omni_address(account_id.to_string()),
              token_id,
              amount.0.to_string(),
          )
          .then(
              Self::ext(env::current_account_id())
                  .with_static_gas(Gas::from_tgas(5))
                  .withdraw_callback(),
          )
  }

  #[private]
  pub fn withdraw_callback(
      &self,
      #[callback_result] call_result: Result<(), PromiseError>,
  ) -> bool {
      if call_result.is_err() {
          log!("There was an error withdrawing from HOT Omni Contract");
          false
      } else {
          env::log_str("withdrawing was successful!");
          true
      }
  }
}
```

</details>

> [!NOTE]
> If you want to withdraw your tokens directly on NEAR, you can simply call `withdraw_on_near` on the Omni smart contract, and you can then skip the next steps.


#### Step Two: Withdraw on the Chain

Once you have the Omni token on the chain you want to withdraw from, you can interact with the smart contract to withdraw the tokens.

> [!NOTE]
> If you want to withdraw your tokens directly on NEAR, you can simply call `withdraw_on_near` on the Omni smart contract, and you can then skip the next steps.


{% tabs %}
{% tab title="Solana" %} 
  To withdraw tokens on Solana you need:

  1. Initiate withdrawing process by calling `withdraw` method on NEAR smart contract. It will create a withdrawal request and decreased your Omni balance.
  2. Get a nonce and amount from near transaction receipts.
  3. Check on NEAR smart contract if nonce is expired or already used.
  4. If it's not, get a signature for that withdrawal request from HOT Omni API
  5. Check if there is unfinished withdrawal request and finish it first.
  6. If there's no unfinished withdrawal use signature to send instructions on Solana liquidity contract and withdraw funds on your wallet.

  If the withdrawal request was not claimed:
  1. Get a withdrawal request data from NEAR smart contract.
  2. Wait until the required time limit has passed and the refund becomes available.
  2. Get a signature from HOT Omni API to refund your Omni balance.
  3. Call `refund` method on NEAR smart contract to finish refund in restore your asset's balance.

  > [!Warning]
  > Any actions on your withdrawal request are available only for the last nonce. It means once you created a withdrawal request you have to finish or refund it before you can create a new one. This is due to the specifics of how smart contracts work in Solana.

{% endtab %}

{% tab title="TON" %}

> [!Warning]
> Any actions on your withdrawal request are available only for the last nonce. It means once you created a withdrawal request you have to finish or refund it before you can create a new one. This is due to the specifics of how smart contracts work in Ton.

To withdraw tokens on Ton you need:

1. Create TON bridge account
2. Initiate withdrawing process by calling `withdraw` method on NEAR smart contract. It will create a withdrawal request and decreased your Omni balance.
3. Get a nonce and amount from near transaction receipts.
4. Check on NEAR smart contract if nonce is expired or already used.
5. If it's not, get a signature for that withdrawal request from HOT Omni API
6. Check if there is unfinished withdrawal request and finish it first.
7. If there's no unfinished withdrawal use signature to send transaction on Ton liquidity contract and withdraw funds on your wallet.

{% endtab %}

{% tab title="Ethereum" %}
{% endtab %}

{% endtabs %}

---

## Metawallets

Metawallet is a smart contract deployed in the relevant network and acting as a liquidity provider on that chain.

| Chain    | Address                                            |
| -------- | -------------------------------------------------- |
| NEAR     | `v1-1.omni.hot.tg`                                 |
| Solana   | `5bG1Kru6ifRmkWMigYaGRKbBKp3WrgcmB6ARNKsV2y2v`     |
| TON      | `EQCuVv07tBHuJrgFrMcDJFHESoE6TpLLoNTuqdL2LkXi7JGM` |
| Ethereum | `0x42351e68420D16613BBE5A7d8cB337A9969980b4`       |