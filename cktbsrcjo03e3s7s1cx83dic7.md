## BSC - Polygon Centralized Token Bridge

Blockchain bridges are connections that enable interoperability between vastly different networks, such as BSC and Polygon. These chains may have different protocols and consensus mechanisms, but the bridges provide a compatible way to transfer token and/or data on all sides. There are many different designs for bridges, and this post relies on the centralized approach, which requires trust on the bridge's authority.

# Why we need bridges?
Suppose we want to move our fund from Ethereum to Polygon network for some reasons, e.g. lower gas fees, investing on projects on Polygon, etc. As Polygon is layer 2 of Ethereum, can we just send tokens from Ethereum wallet to the address of Polygon wallet? The answer is **NO**. They are separate networks, and what happened on Ethereum only affects to Ethereum (see the below image).

![Screen Shot 2021-09-09 at 2.02.30 AM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1631127785582/wOSkVgrG7.png)

# Centralized bridge design

Assume that we are working on a token that can be minted and burned. We will need 2 bridge contracts for both sides and a backend server. If there is a need of bridging tokens, clients will call the bridge contract on the current chain, for example BSC, to burn their tokens in exchange of receiving the corresponding amount of tokens on the other chain, e.g. Polygon. Backend server will listen for BSC bridge contract. If there is a valid request, backend server will call Polygon bridge contract to mint tokens for the clients. Flow diagram is described in the below image.

![Screen Shot 2021-09-09 at 2.43.51 AM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1631130265985/0sTgKxqsz.png)

# Smart contracts

**Github**: https://github.com/liamhieuvu/centralized-bridge-contract.
This project uses OpenZeppelin with Hardhat, and [here](https://docs.openzeppelin.com/learn)  is a good start if you are beginners.

## Token contracts
`TokenBase.sol`
```
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

contract TokenBase is ERC20, AccessControl {
    bytes32 public constant MINTER_BURNER_ROLE = keccak256("MINTER_BURNER_ROLE");

    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _setupRole(MINTER_BURNER_ROLE, msg.sender);
    }

    function mint(address to, uint256 amount) external onlyRole(MINTER_BURNER_ROLE) {
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) external onlyRole(MINTER_BURNER_ROLE) {
        _burn(from, amount);
    }
}
```

The `TokenBase` inherits `ERC20` and `AccessControl` from OpenZeppelin library. By default, the deployer has admin role, which is used to grant other addresses minting/burning permission. Now, we can create token on BSC and Polygon by inheriting `TokenBase` with different constructor parameters:

`Token.sol`
```
pragma solidity ^0.8.0;

import "./TokenBase.sol";

contract Token is TokenBase {
    constructor() TokenBase("BSC Bee Token", "tbBee") {}
    // Polygon: constructor() TokenBase("Polygon Bee Token", "tpBee") {}
}
```

## Bridge contract
```
contract Bridge is Ownable {
    IMintableBurnableToken public token;
    mapping(uint256 => mapping(bytes32 => bool)) processedChainTxns;

    event Transfer(address from, address to, uint256 amount, uint256 toChainID, uint256 timestamp);
    event Mint(address from, address to, uint256 amount, uint256 fromChainID, bytes32 fromChainTxn, uint256 timestamp);

    constructor(address _token) {
        token = IMintableBurnableToken(_token);
    }

    function transferTo(address to, uint256 amount, uint256 toChainID) external {
        require(toChainID != block.chainid, "cannot bridge to the same chain");
        token.burn(msg.sender, amount);
        emit Transfer(msg.sender, to, amount, toChainID, block.timestamp);
    }

    function mint(address to, uint256 amount, uint256 fromChainID, bytes32 fromChainTxn) external onlyOwner {
        require(processedChainTxns[fromChainID][fromChainTxn] == false, "transfer already processed");
        processedChainTxns[fromChainID][fromChainTxn] = true;
        token.mint(to, amount);
        emit Mint(msg.sender, to, amount, fromChainID, fromChainTxn, block.timestamp);
    }
}
```
When deploy `Bridge` contract, we need to grant it permission to mint and burn ( [IMintableBurnableToken](https://github.com/liamhieuvu/centralized-bridge-contract/blob/master/contracts/IMintableBurnableToken.sol)) the token. The `transferTo` method is used for initializing bridge requests. It requires 3 params: `to` for receiver's address, `amount` for token amount, and `toChainID` for the destination network. The `mint` method is used for generating token to receivers. It requires `fromChainID`, and `fromChainTxn` to prevent double minting.

There is an important event - `Transfer`. Backend server will listen to this event and call `mint` method on the network having `toChainID` with the specific amount and receiver's address.

# Backend server
**Github**: https://github.com/liamhieuvu/centralized-bridge-be.
This project uses NodeJS with Loopback 3 framework. [Here](https://loopback.io/doc/en/lb3)  is the full document for learning Loopback 3.

![Untitled drawing (1) (2) (1).png](https://cdn.hashnode.com/res/hashnode/image/upload/v1631132713982/DLJglr4kS.png)

For each chain, there are 3 threads running simultaneously: `syncTx` for listening to `Transfer` events, `confirmTx` for preventing network fork, `processTx` for minting token on the other side of bridge. The following code blocks are the simplified. For the full version, visit [here](https://github.com/liamhieuvu/centralized-bridge-be/blob/master/common/models/transaction.js).

`syncTx` reads the last synced block from database to continue syncing from this block. This prevents missing events when the server is restarted or redeployed. Then, `syncTx` loads bridge contract with given ABI and address. Finally, it gets events with `getPastEvents` and save them to database in an infinity loop.
```js
Transaction.syncTx = async function (cfg) {
  let web3 = getWeb3(cfg.rpcs);
  let currentBlock = await web3.eth.getBlockNumber();
  let syncingBlock = await Transaction.app.models.Status.getSyncingBlock(cfg);
  let toBlock = syncingBlock + cfg.step - 1 < currentBlock ? syncingBlock + cfg.step - 1 : currentBlock;

  const bridge = new web3.eth.Contract(BridgeABI, cfg.address.bridge);

  while (true) {
    let events = await bridge.getPastEvents('Transfer', {fromBlock: syncingBlock, toBlock: toBlock});
    for (const e of events) {
      let data = {
        fromChainID: cfg.id,
        fromTxHash: e.transactionHash,
        fromAddress: e.returnValues.from,
        toChainID: e.returnValues.toChainID,
        toTxHash: '',
        ...
      }
      await Transaction.findOrCreate({where: {fromChainID: cfg.id, fromTxHash: e.transactionHash}}, data);
    }

    syncingBlock = toBlock + 1;
    await Transaction.app.models.Status.upsert({'key': cfg.name + 'SyncingBlock', value: syncingBlock});
    if (toBlock === currentBlock) {
      await sleep(constants.TASK_SLEEP_MS);
      currentBlock = await web3.eth.getBlockNumber();
    }
    toBlock = syncingBlock + cfg.step - 1 < currentBlock ? syncingBlock + cfg.step - 1 : currentBlock;
  }
}
```

`confirmTx` thread is simply waiting for enough confirmation for the transaction from network. It will update status of transaction to `transfering` when there are at least 12 confirmations.
```js
Transaction.confirmTx = async function (cfg) {
  while (true) {
    let txns = await Transaction.find({where: {fromChainID: cfg.id, status: constants.STATUS_CONFIRMING}});
    if (txns.length > 0) {
      let web3 = getWeb3(cfg.rpcs);
      let currentBlock = await web3.eth.getBlockNumber();
      for (let txn of txns) {
        let rawTx = await web3.eth.getTransaction(txn.fromTxHash);
        if (rawTx.blockNumber + 12 > currentBlock) {
          break;
        }
        txn.status = constants.STATUS_TRANSFERRING;
        await Transaction.upsert(txn);
      }
    }
    await sleep(constants.TASK_SLEEP_MS);
  }
}
```

`processTx` only call `mint` method on bridge contract when transactions are marked as `transferring` status. It also retries for a limited number of times when there are errors on calling `mint` method.
```js
Transaction.processTx = async function (cfg) {
  while (true) {
    let txns = await Transaction.find({where: {fromChainID: cfg.id, status: constants.STATUS_TRANSFERRING, retry: {lt: constants.MAX_RETRY}}, limit: 5, order: 'id asc'});

    for (let txn of txns) {
      try {
        let web3 = getWeb3(config.network[txn.toChainID].rpcs);
        const account = web3.eth.accounts.privateKeyToAccount(secrets.network[txn.toChainID].privateKey);
        web3.eth.accounts.wallet.add(account);
        web3.eth.defaultAccount = account.address;
        const bridge = new web3.eth.Contract(BridgeABI, config.network[txn.toChainID].address.bridge);

        let method = bridge.methods.mint(txn.toAddress, txn.amount, txn.fromChainID, txn.fromTxHash);
        const estimatedGas = await method.estimateGas({from: account.address});
        const tx = await method.send({from: account.address, gas: estimatedGas});

        txn.status = constants.STATUS_SUCCESS;
        txn.toTxHash = tx.transactionHash;
      } catch (e) {
        txn.retry += 1;
        if (txn.retry >= constants.MAX_RETRY) txn.status = constants.STATUS_FAILED;
      }
      await Transaction.upsert(txn);
    }
    await sleep(constants.TASK_SLEEP_MS);
  }
}
```

# Demo
## Deploy contracts
Open shell `1`:
```shell
git clone https://github.com/liamhieuvu/centralized-bridge-contract.git
cd centralized-bridge-contract
npm install
```
Edit the mnemonic, which is used to generate the deployer address, in `secrets.json`, then:
```shell
npm run deploy:testnet:bsc
npm run deploy:testnet:polygon
```
After deploying, remember the addresses of token and bridge on 2 chains.

## Run backend server
Open new shell `2`:
```shell
git clone https://github.com/liamhieuvu/centralized-bridge-be.git
cd centralized-bridge-be
npm install
```
Update the deployed addresses to `common/models/config.json` and add deployer's secret key to `common/models/secrets.json`. Then, run `node .`

## Interact with contracts
On the shell `1`:
Minting some first tokens to demo:
```shell
node scripts/mint-token.js <receiver-address>
```

Then, call `transferTo` method of bridge contract and check the fund is moved from one chain to another chain:
```shell
node scripts/transfer-bsc-polygon.js <sender-private-key>
node scripts/transfer-polygon-bsc.js <sender-private-key>
```