## Algorand Starter (Part 1) - Client Side

This is the first part of Algorand Starter series. Algorand is a new raising blockchain and it aims to be scalable. However, most developers are familiar with Ethereum-like blockchains currently. To boost the progress of learning and working on Algorand, this series will provide some basic knowledge:
+ **Part 1: Client side**
+ Part 2: Stateful contract (Smart contract)
+ Part 3: Stateless contract (Smart signature)
+ Part 4: Test scripts

The content of the first part will includes algod, indexer and Golang SDK.
## Algod
![Screen Shot 2021-12-13 at 4.16.39 PM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1639387050757/svKXK-VSn.png)

Algod, like `geth` on Ethereum, is the main Algorand process for handling the blockchain. It processes messages between nodes, executes the protocol steps, and write the blockchain data to disk. The `algod` process also exposes REST APIs like EVM RPC that clients can use to communicate with the node and the network. There are 2 main types of actions that `algod` can carry on: submitting TX and searching data. The speed of searching on `algod` may be slow because it uses the data directory for storage.

One of the public `algod` is https://algoexplorerapi.io. Beside that, we can use paid services to achieve full query range and faster speed. Personally, I prefer https://purestake.com to https://ankr.com when choosing paid services for `algod`. We can also setup a local node which contains `algod` with  [sanbox](https://github.com/algorand/sandbox). Sandbox is a light node (store up to 1000 blocks) and can be installed as below:
```sh
git clone https://github.com/algorand/sandbox.git
cd sandbox
./sandbox up testnet
```
|         | Public                     | Sandbox                                                          |
|---------|----------------------------|------------------------------------------------------------------|
| Address | https://algoexplorerapi.io | http://localhost:4001                                            |
| Token   | "" (None)                  | aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa |

## Indexer
![Screen Shot 2021-12-13 at 4.47.36 PM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1639388887947/z-FMrIvu_.png)

Unlike `algod`, indexer is focus only on searching operation and cannot process transaction requests. To archive better querying speed, the Indexer REST APIs retrieve the blockchain data from a database instead of file system. This database is populated by an indexer which must connect to the `algod` of an archival node to read all possible blockchain data. When the database is populated, multiple indexer instances can be connected to this database to enable higher query performance. We can make a load balancer, e.g. NGINX, for a cluster of indexers.

One of the public indexer is https://algoexplorerapi.io/idx2. We can also use paid services, e.g. https://purestake.com or local sandbox instance.

## Golang SDK
This section is focus only common operations on testnet, and contract write interaction will be covered in next parts of the series.

![Screen Shot 2021-12-13 at 5.29.02 PM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1639391364314/MJGqf4wi5.png)

#### Instantiate clients
Algod:
```go
const algodAddress = "https://testnet.algoexplorerapi.io"
const algodToken = ""
algodClient, err := algod.MakeClient(algodAddress, algodToken)
```
Indexer:
```go
const indexerAddress = "https://testnet.algoexplorerapi.io/idx2"
const indexerToken = ""
indexerClient, err := indexer.MakeClient(indexerAddress, indexerToken)
```

#### Check account balances
```go
addr := "27GYZK5KAHA2A7URIBQ7ENGH3WQMKWKHQTOC4XUB56XFTNPWRMNI6A6HLY"
_, accInfo, _ := indexerClient.LookupAccountByID(addr).Do(context.Background())
// accInfo, _ := algodClient.AccountInformation(account).Do(context.Background())
fmt.Println("microAlgos:", accInfo.Amount)
fmt.Printf("assets: %+v\n", accInfo.Assets)
```
We can use both algod and indexer to query. Comment indexer and uncomment algod line to try with algod. `accInfo.Amount` shows the ALGO (native asset) balance, and `accInfo.Assets` shows all balances of [ASAs](https://developer.algorand.org/docs/get-details/asa)  (Algorand Standard Assets). ASAs almost like tokens on Ethereum. They can be created as smart contracts, but the their ID is integers (not opaque string).

### Lookup info of an ASA (Token)
```go
assetID := uint64(408947)
_, assetInfo, _ := indexerClient.LookupAssetByID(assetID).Do(context.Background())
// assetInfo, _ := algodClient.GetAssetByID(assetID).Do(context.Background())
fmt.Printf("assetParams: %+v\n", assetInfo.Params)
```
We can get all info, e.g. name, decimals, etc.  of an ASA at once. On Ethereum chain, we must use multicall to group multiple function call to an token (smart contract).

#### Read smart contract variables
```go
appID := uint64(43178587)
appInfo, _ := indexerClient.LookupApplicationByID(appID).Do(context.Background())

globalState := appInfo.Application.Params.GlobalState
globalStateMap := make(map[string]interface{})
for i := range globalState {
	key, _ := base64.StdEncoding.DecodeString(globalState[i].Key)
	switch globalState[i].Value.Type {
	case 1:
		globalStateMap[string(key)] = globalState[i].Value.Bytes
	case 2:
		globalStateMap[string(key)] = globalState[i].Value.Uint
	}
}
fmt.Printf("globalState: %+v\n", globalStateMap)
```
There are 2 types of contract state: global and local state. The global state of a contract are used to contain configurations or general data. They are stored on the creator account, but we can get them by calling APIs to smart contract. The algod or indexer will aggregate data for us. The local state are data per user and stored on user accounts. We can get them by using `indexerClient.LookupAccountByID`.

#### Transfer assets
There are 6 type of transactions. One of them is payment transaction, and we can use this type of transaction to transfer assets. We must use algod to submit TX.
```go
txParams, err := algodClient.SuggestedParams().Do(context.Background())
if err != nil {
    fmt.Printf("Error getting suggested tx params: %s\n", err)
    return
}
fromAddr := myAddress
toAddr := yourAddress
var amount uint64 = 9000000
var minFee uint64 = 1000
note := []byte("Hello World")
genID := txParams.GenesisID
genHash := txParams.GenesisHash
firstValidRound := uint64(txParams.FirstRoundValid)
lastValidRound := uint64(txParams.LastRoundValid)
txn, err := transaction.MakePaymentTxnWithFlatFee(fromAddr, toAddr, minFee, amount, firstValidRound, lastValidRound, note, "", genID, genHash)
if err != nil {
    fmt.Printf("Error creating transaction: %s\n", err)
    return
}
```
The about example will transfer 9 ALGO from my address to your address. Note that the suggested parameters provide the default values that are required to submit a transaction, such as the expected fee (we use flat fee for this example), first and last valid rounds (blocks) for the transaction.

Thank you for reading my blog. Stay tune for the next parts.

Github: https://github.com/liamhieuvu/algorand-first-txns.git

References: https://developer.algorand.org/docs
