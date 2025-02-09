# Getting Started

libraries:

- [ethers](#ethers)  
- [web3.js](#web3)  
- [truffle](#truffle)

some examples:

- [React using ethers and web3 providers](https://github.com/RetricSu/polyjuice-provider-example)
- [depoly contract to polyjuice using Truffle](https://github.com/RetricSu/simple-storage-v2)

<a name="ethers"/>

## ethers

### Install

```sh
yarn add @polyjuice-provider/ethers 
```

or

```sh
npm install --save @polyjuice-provider/ethers
```

### Usage

once you install this module, then you got two main tools to run with `ethers` for compatibility:

- PolyjuiceJsonRpcProvider (compatible version of [JsonRpcProvider](https://github.com/ethers-io/ethers.js/blob/master/packages/providers/src.ts/json-rpc-provider.ts))
- PolyjuiceWallet (compatible version of [Wallet](https://github.com/ethers-io/ethers.js/tree/master/packages/wallet))

replacement:

```ts
new providers.JsonRpcProvider(..) ---> new PolyjuiceJsonRpcProvider(...)
new Wallet(..) --> new PolyjuiceWallet(...)
```

#### Example: Deploy contract

```ts
import { ContractFactory } from "ethers";
import { PolyjuiceWallet, PolyjuiceConfig, PolyjuiceJsonRpcProvider } from "@polyjuice-provider/ethers";

const polyjuice_config: PolyjuiceConfig = {
  rollupTypeHash: 'godwoken rollup type hash', 
  ethAccountLockCodeHash: 'godwoken eth account lock code hash', 
  abiItems: ['your abi items array'], // this is optional
  web3Url: 'godwoken web3 rpc url', 
};
const rpc = new PolyjuiceJsonRpcProvider(polyjuice_config, PolyjuiceConfig.web3Url); 
const deployer = new PolyjuiceWallet('<your deployer private key>', polyjuice_config, rpc);
const implementationFactory = new ContractFactory(
  contract.abi,
  contract.bytecode,
  deployer,
);
const tx = implementationFactory.getDeployTransaction();
tx.gasPrice = 0;
tx.gasLimit = 1_000_000;
deployer.sendTransaction(tx);
```

a more complete and real example can be found [here](https://github.com/RetricSu/godwoken-polyjuice-compatibility-examples/commit/90ccce0288cc44f0c5ba3d338c142922518867d2#diff-86f1dc0bf3c5524626be0d195ed3872e309c3175c4cd71305b7ffcc7c1444164)

### Migrate dapp

if your dapp are using `ethers` and `metamask`, you can simply change it like following:

```ts
import { providers, ContractFactory, Signer } from "ethers";
import { PolyjuiceHttpProvider } from "@polyjuice-provider/web3";

const polyjuiceConfig: PolyjuiceConfig = {
  rollupTypeHash: 'godwoken rollup type hash', 
  ethAccountLockCodeHash: 'godwoken eth account lock code hash', 
  abiItems: ['your abi items array'], // this is optional
  web3Url: 'godwoken web3 rpc url', 
};

export async function createEthersSignerWithMetamask(): Promise<
  Signer | undefined
> {
  if ((window as any).ethereum) {
    const provider = new providers.Web3Provider(
      new PolyjuiceHttpProvider(polyjuiceConfig.web3Url!, polyjuiceConfig)
    );
    let signer;

    try {
      await (window as any).ethereum.enable();
      signer = provider.getSigner((window as any).ethereum.selectedAddress);
    } catch (error) {
      // User denied account access...
      throw error;
    }

    return signer;
  }

  console.error(
    "Non-Ethereum browser detected. You should consider trying MetaMask!"
  );
  return undefined;
}

const signer = await createEthersSignerWithMetamask();
const contract = new ethers.Contract(
  'your contract address',
  'your contract abi',
  signer
);
let overrides = {
  gasLimit: 0x54d30,
  gasPrice: 0x0,
  value: 0x0,
};
const txResponse = await contract.METHOD_NAME(..args, overrides);
console.log(txResponse);
```

known issue:

if you want to use `Metamask` and `ethers.Contract` to deploy contract, then do not use the contract.address as your deployed contract address:

```ts
const deployContractWithEtherContractFactory = async () => {
  const signer = await createEthersSignerWithMetamask();
  
  const contractDeployer = new ContractFactory(
    'your contract abi',
    'your contract compiled bytecode',
    signer
  );
  let overrides = {
    gasLimit: 0x54d30,
    gasPrice: 0x0,
    value: 0x0,
  };
  const contract = await contractDeployer.deploy(overrides);
  await contract.deployed();
  // ! please do not use `contract.address` as contractAddress here. 
  // due to an known issue, it is wrong eth address in polyjuice. 
  // instead, you should get address through txReceipt.
  const txReceipt = await rpc.eth_getTransactionReceipt(contract.deployTransaction.hash);
  console.log(`contract address: ${txReceipt.contractAddress)}`);
}
```

<a name="web3"/>

## web3.js

### Install

```sh
yarn add @polyjuice-provider/web3 
```

or

```sh
npm install --save @polyjuice-provider/web3 
```

### Usage

once you install this module, then you got three main tools to run with `web3.js` for compatibility:

- PolyjuiceHttpProvider (compatible version of [web3-providers-http](https://github.com/ChainSafe/web3.js/tree/1.x/packages/web3-providers-http))
- PolyjuiceAccounts (compatible version of [web3.eth.accounts](https://github.com/ChainSafe/web3.js/tree/1.x/packages/web3-eth-accounts))
- PolyjuiceHttpProviderCli (old version solution for signing tx with web3.js in nodejs environment, now recommend use PolyjuiceAccounts instead of this module)

replacement:

```sh
new Web3HttpProvider(..) --> new PolyjuiceHttpProvider(...)
new Web3EthAccounts(..) ---> new PolyjuiceAccounts(...)
```

#### Example: Deploy contract

```js
const Web3 = require("web3");
const { PolyjuiceHttpProvider, PolyjuiceAccounts } = require("@polyjuice-provider/web3");

const polyjuice_config: PolyjuiceConfig = {
  rollupTypeHash: 'godwoken rollup type hash', 
  ethAccountLockCodeHash: 'godwoken eth account lock code hash', 
  abiItems: ['your abi items array'], // this is optional
  web3Url: 'godwoken web3 rpc url', 
};

provider = new PolyjuiceHttpProvider(
  godwoken_rpc_url,
  provider_config,
);
polyjuiceAccounts = new PolyjuiceAccounts(polyjuice_config);

const web3 = new Web3(provider);
web3.eth.accounts = polyjuiceAccounts;
web3.eth.Contract._accounts = web3.eth.accounts;
web3.eth.accounts.wallet.add(`your private key`);

const myContract = await web3.eth.Contract(`your contract's abi`);
const contractInstance = myContract
    .deploy({
      data: `contract bin`,
      arguments: [],
    })
    .send({
      gas: "0x30d40", 
      gasPrice: "0x00"
    });
const contract_deployed_address = contractInstance._address;
```

### Migrate dapp

if your dapp are using `web3.js` and `metamask`, you can simply change it like following:

### init web3

Before:

```js
import Web3 from 'web3';

var web3 = new Web3(new Web3.providers.HttpProvider('http://localhost:8024'));
```

Now:

```js
import Web3 from 'web3';
import { PolyjuiceHttpProvider } from '@polyjuice-provider/web3';

var web3 = new Web3(new PolyjuiceHttpProvider('http://localhost:8024', polyjuiceConfig));
```

for ```polyjuiceConfig```: see above.

### init contract instance

Before:

```js
import Web3 from 'web3';

var provider = new Web3.providers.HttpProvider('http://localhost:8024');
var web3 = new Web3(provider);

var contract = web3.eth.Contract(abi, contract_address);
```

Now:

```js
import Web3 from 'web3';
import { PolyjuiceHttpProvider } from '@polyjuice-provider/web3';

var provider = new PolyjuiceHttpProvider('http://localhost:8024', polyjuiceConfig);
var web3 = new Web3(provider);

var contract = web3.eth.Contract(abi, contract_address);
```

<a name="truffle"/>

## truffle

### Install

```sh
yarn add @polyjuice-provider/truffle 
```

or

```sh
npm install --save @polyjuice-provider/truffle 
```

### Usage

once you install this module, then you got the following tool to run inside truffle project for compatibility:

- PolyjuiceHDWalletProvider (compatible version of [@truffle/hdwallet-provider](https://github.com/trufflesuite/truffle/tree/develop/packages/hdwallet-provider))

replacement:

```sh
new HDWalletProvider(..) --> new PolyjuiceHDWalletProvider(...)
```

#### Deploy contract With Truffle

use `PolyjuiceHDWalletProvider` in truffle-config.js for your truffle project.

```sh
const { PolyjuiceHDWalletProvider } = require("@polyjuice-provider/truffle");
const { PolyjuiceHttpProvider } = require("@polyjuice-provider/web3");

const root = require("path").join.bind(this, __dirname, ".");
require("dotenv").config({ path: root(".env") });

const rpc_url = new URL(process.env.WEB3_JSON_RPC);

const godwoken_rpc_url = process.env.WEB3_JSON_RPC;
const polyjuice_config = {
  rollupTypeHash: process.env.ROLLUP_TYPE_HASH,
  ethAccountLockCodeHash: process.env.ETH_ACCOUNT_LOCK_CODE_HASH,
  web3Url: godwoken_rpc_url,
};

const polyjuiceHttpProvider = new PolyjuiceHttpProvider(
  polyjuice_config.web3Url,
  polyjuice_config
);
const polyjuiceTruffleProvider = new PolyjuiceHDWalletProvider(
  [
    {
      privateKeys: [process.env.PRIVATE_KEY],
      providerOrUrl: polyjuiceHttpProvider,
    },
  ],
  polyjuice_config
);

module.exports = {
  networks: {
    development: {
      host: rpc_url.hostname, // Localhost (default: none)
      port: rpc_url.port, // Standard Ethereum port (default: none)
      gasPrice: "0", // important for dryRun mode. 0 must be string type.
      network_id: "*", // Any network (default: none)
      provider: () => polyjuiceTruffleProvider,
    }
};
```

checkout this [example](https://github.com/RetricSu/simple-storage-v2)
