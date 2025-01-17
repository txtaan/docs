---
title: Aztec Sandbox
---

import Image from "@theme/IdealImage";
import GithubCode from '../../../src/components/GithubCode';

## Introduction

The Aztec Sandbox aims to provide a local development system against which you can build and test Noir contracts in a fast, safe, and free environment.

Here we will walkthrough the process of retrieving the Sandbox, installing the client libraries and using it to deploy and use a fully private token contract on the Aztec network.

## What do you need?

- Node.Js >= v18
- Docker and Docker Compose (Docker Desktop under WSL2 on windows)

That's it...

## Ok, so how do I try it out?

Well, you can find instructions [at the website](https://up.aztec.network).

Or you can just curl the site instead like this:

`/bin/bash -c "$(curl -fsSL 'https://up.aztec.network')"`

It will download and execute a script invoking docker compose with 2 containers:

- Anvil
- Aztec Sandbox

It will need to create servers on localhost ports 8545 (Anvil) and 8080 (Sandbox) so you will need to ensure nothing conflicts with this.

Within a few seconds the Sandbox should be ready for use!

<Image img={require("/img/sandbox.png")} />

## Great, but I want to know more about it

Aztec's Layer 2 network is a fully programmable combined private/public ZK rollup. To achieve this, the network contains the following primary components:

- Aztec Node - Aggregates all of the 'backend' services necessary for the building and publishing of rollups.
- Aztec RPC Server - Normally residing with the end client, this decrypts and stores a client's private state, executes simulations and submits transactions to the Aztec Node.
- Aztec.js - Aztec's client library for interacting with the Aztec RPC Server (think Ethers.js).

All of this is included in the Sandbox, with the exception of Aztec.js which you can use to interact with it.

With the help of Aztec.js you will be able to:

- Create an account
- Deploy a contract
- Call view methods on contracts
- Simulate the calling of contract functions
- Send transactions to the network
- Be notified when transactions settle
- Retrieve and view unencrypted logs emitted by contracts
- Query chain state such as chain id, block number etc.

## I have the Sandbox running, show me how to use it!

Let's have a complete walkthrough from start to finish. I'm using WSL2 Ubuntu under Windows but the following should work under regular Linux or MacOS. We will deploy and use a private token contract on our Sandbox. Writing the contract itself is out of scope for this tutorial, we will use the Private Token Contract supplied as one of the example contracts.

Let's create an empty project called `private-token`. If you are familiar with setting up Typescript projects then you can fast-forward the next couple of steps.

Also, although I am using `yarn`, vanilla `npm` would work just as well.

```
~$ node -v
v18.8.0
~$ mkdir private-token
~$ cd private-token
~/private-token$ yarn init
yarn init v1.22.19
question name (private-token):
question version (1.0.0):
question description: My first private token contract
question entry point (index.js):
question repository url:
question author: Phil
question license (MIT):
question private:
success Saved package.json
Done in 23.60s.
~/private-token$ mkdir src
```

We use Typescript here at Aztec, so lets add this to the project.

```
~/private-token yarn add typescript @types/node --dev
```

Add a `tsconfig.json` file into the project root, here is an example:

```
{
  "compilerOptions": {
    "outDir": "dest",
    "rootDir": "src",
    "target": "es2020",
    "lib": ["dom", "esnext", "es2017.object"],
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "declaration": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "downlevelIteration": true,
    "inlineSourceMap": true,
    "declarationMap": true,
    "importHelpers": true,
    "resolveJsonModule": true,
    "composite": true,
    "skipLibCheck": true
  },
  "references": [],
  "include": ["src"]
}
```

Add a `scripts` section to `package.json` and set `"type": "module"`:

```
{
	"name": "private-token",
	"version": "1.0.0",
	"description": "My first private token contract",
	"main": "index.js",
	"author": "Phil",
	"license": "MIT",
	"type": "module",
	"scripts": {
		"build": "yarn clean && tsc -b",
		"build:dev": "tsc -b --watch",
		"clean": "rm -rf ./dest tsconfig.tsbuildinfo",
		"start": "yarn build && export DEBUG='private-token' && node ./dest/index.js"
	},
	"devDependencies": {
		"@types/node": "^20.4.9",
		"typescript": "^5.1.6"
	}
}
```

Now we want to install 2 Aztec packages from npm:

```
yarn add @aztec/aztec.js @aztec/noir-contracts
```

Create an `index.ts` under the `src` directory:

#include_code index /docs/src/code_examples/sandbox_example.ts typescript

Running `yarn start` should give:

```
  private-token Aztec Sandbox Info  { version: 1, chainId: 31337 } +0ms
```

Great!. The Sandbox is running and we are able to interact with it.

## Account Creation/Deployment

The next step is to create some accounts. I won't go into detail about accounts as that is covered [here](../../concepts/foundation/accounts/main.md). But creating an account on the Sandbox does 2 things:

1. Deploys an account contract reprepresenting you allowing you to perform actions on the network (deploy contracts, call functions etc).
2. Adds your encryption keys to the RPC Server allowing it to decrypt and manage your private state.

Continue with adding the following to our example:

#include_code Accounts /docs/src/code_examples/sandbox_example.ts typescript

Running `yarn start` should now output:

```
  private-token Aztec Sandbox Info  { version: 1, chainId: 31337 } +0ms
  private-token Creating accounts using schnorr signers... +2ms
  private-token Created Alice's account at 0x054d89d0...f17e +23s
  private-token Created Bob's account at 0x0a8410a1...7c48 +1ms
```

That might seem like a lot to digest but it can be broken down into the following steps:

1. We create 2 `Account` objects in Typescript. This object heavily abstracts away the mechanics of configuring and deploying an account contract and setting up a 'wallet' for signing transactions. If you aren't interested in building new types of account contracts or wallets then you don't need to be too concerned with it. In this example we have constructed account contracts and corresposing wallets that sign/verify transactions using schnorr signatures.
2. We wait for the deployment of the 2 account contracts to complete.
3. We retrieve the expected account addresses from the `Account` objects and ensure that they are present in the set of account addresses registered on the Sandbox.

Note, we use the `getAccounts` api to verify that the addresses computed as part of the
account contract deployment have been successfully added to the Sandbox.

If you were looking at your terminal that is running the Sandbox you should hopefully have seen a lot of activity. This is because the Sandbox will have simulated the deployment of both contracts, executed the private kernel circuit for each before submitted 2 transactions to the pool. The sequencer will have picked them up and inserted them into a rollup and executed the recursive rollup circuits before publising the rollup to Anvil. Once this has completed, the rollup is retrieved and pulled down to the internal RPC Server so that any new account state can be decrypted.

## Token Contract Deployment

Now that we have our accounts setup, let's move on to deploy our private token contract. Add this to `index.ts`:

#include_code Deployment /docs/src/code_examples/sandbox_example.ts typescript

`yarn start` will now give the following output:

```
  private-token Aztec Sandbox Info  { version: 1, chainId: 31337 } +0ms
  private-token Creating accounts using schnorr signers... +2ms
  private-token Created Alice's account at 0x054d89d0...f17e +23s
  private-token Created Bob's account at 0x0a8410a1...7c48 +1ms
  private-token Deploying private token contract minting an initial 1000000 tokens to Alice... +0ms
  private-token Transaction status is mined +8s
  private-token Contract successfully deployed at address 0x143e0af4...11b6 +7ms
```

We can break this down as follows:

1. We create and send a contract deployment transaction to the network.
2. We wait for it to be successfully mined.
3. We retrieve the transaction receipt containing the transaction status and contract address.
4. We use the `getContractInfo()` api on the RPC Server to retrieve information about the reported contract address.
5. The fact that this api returns a valid object tells us that the contract was successfully deployed in a prior block.

The Private Token Contract emits an unencrypted log message during construction:

<GithubCode owner="AztecProtocol" language="rust" repo="aztec-packages" branch="master" filePath="yarn-project/noir-contracts/src/contracts/private_token_contract/src/main.nr" startLine={25} endLine={45} />

We can retrieve this emitted log using the `getUnencryptedLogs()` api:

#include_code Logs /docs/src/code_examples/sandbox_example.ts typescript

Our output will now be:

```
  private-token Aztec Sandbox Info  { version: 1, chainId: 31337 } +0ms
  private-token Creating accounts using schnorr signers... +2ms
  private-token Created Alice's account at 0x054d89d0...f17e +23s
  private-token Created Bob's account at 0x0a8410a1...7c48 +1ms
  private-token Deploying private token contract minting an initial 1000000 tokens to Alice... +0ms
  private-token Transaction status is mined +8s
  private-token Contract successfully deployed at address 0x143e0af4...11b6 +7ms
  private-token Retrieving unencrypted logs for block 3 +4ms
  private-token Emitted logs:  [ 'Balance set in constructor' ] +5ms
```

Note how we used the `getBlockNum()` api to retrieve the number of the last mined block. This is the block for which we want to retrieve logs as it is the last mined block number.

## Viewing the balance of an account

A token contract wouldn't be very useful if you aren't able to query the balance of an account. As part of the deployment, tokens were minted to Alice. We can now call the contract's `getBalance()` function to retrieve the balances of the accounts.

<GithubCode owner="AztecProtocol" language="rust" repo="aztec-packages" branch="master" filePath="yarn-project/noir-contracts/src/contracts/private_token_contract/src/main.nr" startLine={96} endLine={106} />

Call this function using the following code:

#include_code Balance /docs/src/code_examples/sandbox_example.ts typescript

Running now should yield output:

```
  private-token Aztec Sandbox Info  { version: 1, chainId: 31337 } +0ms
  private-token Creating accounts using schnorr signers... +2ms
  private-token Created Alice's account at 0x054d89d0...f17e +23s
  private-token Created Bob's account at 0x0a8410a1...7c48 +1ms
  private-token Deploying private token contract minting an initial 1000000 tokens to Alice... +0ms
  private-token Transaction status is mined +8s
  private-token Contract successfully deployed at address 0x143e0af4...11b6 +7ms
  private-token Retrieving unencrypted logs for block 3 +4ms
  private-token Emitted logs:  [ 'Balance set in constructor' ] +5ms
  private-token Alice's balance 1000000 +4s
  private-token Bob's balance 0 +3s
```

In this section, we first created 2 instances of the `PrivateTokenContract` contract abstraction. One for each of our deployed accounts. This contract abstraction offers a Typescript interface reflecting the abi of the contract. We then call `getBalance()` as a `view` method. View methods can be thought as read-only. No transaction is submitted as a result but a user's state can be queried.

We can see that each account has the expected balance of tokens.

## Creating and submitting transactions

Now lets transfer some funds from Alice to Bob by calling the `transfer` function on the contract. This function takes 3 arguments:

1. The quantity of tokens to transfer.
2. The sender.
3. The recipient.

<GithubCode owner="AztecProtocol" language="rust" repo="aztec-packages" branch="master" filePath="yarn-project/noir-contracts/src/contracts/private_token_contract/src/main.nr" startLine={69} endLine={93} />

We will again view the unencrypted logs emitted by the function and check the balances after the transfer:

#include_code Transfer /docs/src/code_examples/sandbox_example.ts typescript

Our output should now look like this:

```
  private-token Aztec Sandbox Info  { version: 1, chainId: 31337 } +0ms
  private-token Creating accounts using schnorr signers... +2ms
  private-token Created Alice's account at 0x054d89d0...f17e +23s
  private-token Created Bob's account at 0x0a8410a1...7c48 +1ms
  private-token Deploying private token contract minting an initial 1000000 tokens to Alice... +0ms
  private-token Transaction status is mined +8s
  private-token Contract successfully deployed at address 0x143e0af4...11b6 +7ms
  private-token Retrieving unencrypted logs for block 3 +4ms
  private-token Emitted logs:  [ 'Balance set in constructor' ] +5ms
  private-token Alice's balance 1000000 +4s
  private-token Bob's balance 0 +3s
  private-token Transferring 543 tokens from Alice to Bob... +0ms
  private-token Retrieving unencrypted logs for block 4 +20s
  private-token Emitted logs:  [ 'Coins transferred' ] +13ms
  private-token Alice's balance 999457 +4s
  private-token Bob's balance 543 +3s
```

Here, we used the same contract abstraction as was previously used for reading Alice's balance. But this time we called `send()` generating and sending a transaction to the network. After waiting for the transaction to settle we were able to retrieve the newly emitted unencrypted logs and check the new balance values.

Finally, the contract has a `mint` function that can be used to generate new tokens for an account. This takes 2 arguments:

1. The quantity of tokens to be minted.
2. The recipient of the new tokens.

<GithubCode owner="AztecProtocol" language="rust" repo="aztec-packages" branch="master" filePath="yarn-project/noir-contracts/src/contracts/private_token_contract/src/main.nr" startLine={48} endLine={66} />

Let's mint some tokens to Bob's account:

#include_code Mint /docs/src/code_examples/sandbox_example.ts typescript

Our complete output should now be:

```
  private-token Aztec Sandbox Info  { version: 1, chainId: 31337 } +0ms
  private-token Creating accounts using schnorr signers... +2ms
  private-token Created Alice's account at 0x054d89d0...f17e +23s
  private-token Created Bob's account at 0x0a8410a1...7c48 +1ms
  private-token Deploying private token contract minting an initial 1000000 tokens to Alice... +0ms
  private-token Transaction status is mined +8s
  private-token Contract successfully deployed at address 0x143e0af4...11b6 +7ms
  private-token Retrieving unencrypted logs for block 3 +4ms
  private-token Emitted logs:  [ 'Balance set in constructor' ] +5ms
  private-token Alice's balance 1000000 +4s
  private-token Bob's balance 0 +3s
  private-token Transferring 543 tokens from Alice to Bob... +0ms
  private-token Retrieving unencrypted logs for block 4 +20s
  private-token Emitted logs:  [ 'Coins transferred' ] +13ms
  private-token Alice's balance 999457 +4s
  private-token Bob's balance 543 +3s
  private-token Minting 10000 tokens to Bob... +1ms
  private-token Retrieving unencrypted logs for block 5 +13s
  private-token Emitted logs:  [ 'Coins minted' ] +13ms
  private-token Alice's balance 999457 +4s
  private-token Bob's balance 10543 +4s
```

That's it! We have successfully deployed a private token contract to an instance of the Aztec network and mined private state-transitioning transactions. We have also queried the resulting state all via the interfaces provided by the contract.
