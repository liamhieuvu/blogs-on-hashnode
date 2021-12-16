## Algorand Starter (Part 2) - Smart Contract

This is the second part of Algorand Starter series. Smart contract is an indispensable part of any blockchains. It enables developing various dApps and extends the blockchain ecosystem. On Algorand chain, smart contracts (ASC1) are also known as stateful contract. This blog will introduce contract architecture and a simple example for demonstration.

Algorand Starter series:
+ Part 1: Client side
+ **Part 2: Stateful contract (Smart contract)**
+ Part 3: Stateless contract (Smart signature)

## Contract Architecture
### Model
![Screen Shot 2021-12-14 at 4.08.19 PM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1639472926288/EqQpLToL2.png)
Algorand smart contracts (ASC1) are stateful contracts, but they do not contain any variables. The variables will be stored at creator and user accounts, and ASC1 will read and write these variables using TEAL opcodes.

Local storage values are stored in the user account's balance record. An account can have its local storage modified by the smart contract as long as the account has opted into the smart contract. Global storage are stored in creators and can also be modified by the smart contract code.

### Application methods
![Screen Shot 2021-12-14 at 5.13.58 PM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1639476854446/T0b5erIsV.png)

Smart contracts are implemented using two programs:
+ **ApprovalProgram**: Responsible for most of the logic of an application. This program will succeed only if one nonzero value is returned
+ **ClearStateProgram**: Handle accounts using the clear call to remove the smart contract from their balance record

We can call to smart contracts using With ApplicationCall transactions. There are 6 types of transactions:
+ **NoOps**: Generic application calls to execute the ApprovalProgram
+ **OptIn**: Accounts use this transaction to opt into the smart contract to participate (local storage usage).
+ **DeleteApplication**: Transaction to delete the application.
+ **UpdateApplication**: Transaction to update TEAL Programs for a contract.
+ **CloseOut**: Accounts use this transaction to close out their participation in the contract. This call can fail based on the TEAL logic, preventing the account from removing the contract from its balance record.
+ **ClearState**: Similar to CloseOut, but the transaction will always clear a contract from the account’s balance record whether the program succeeds or fails.

### Runtime execution
![Screen Shot 2021-12-14 at 5.56.49 PM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1639479741290/xKTcEmO6P.png)

A set of arrays can be passed with any application transaction, which instructs the protocol to load additional data for use in the contract. These arrays are:
+ **applications array**: used to read state for the specific contracts
+ **accounts array**: allows additional accounts to be passed to the contract for balance information and local storage
+ **assets array**: used to retrieve configuration and asset balance information
+ **arguments array**: used as method's input parameters.

On runtime, TEAL program will load necessary variables on its stack. The applications array, accounts array, and assets array are read-only, while global and local state are readable and writable. The program can also use temporary variables by requesting memory from scratch memory.

## Pyteal example
The following sample builds a simple counter smart contract that either adds or deducts one from a global counter based on how the contract is called.

### Setup development environment
Install pyteal
```sh
pip3 install pyteal
```

Install sandbox
```sh
git clone https://github.com/algorand/sandbox.git
cd sandbox
./sandbox up testnet
```
### Build contract
First, we need to handle 5 types of transactions for approval program. In this example, we don't handle `OptIn`, `CloseOut` (does not require local storage from users); `UpdateApplication`, and `DeleteApplication`, so just return 0 for errors.

```python
# counter_contract.py
def clear_state_program():
  program = Return(Int(1))
  compileTeal(program, Mode.Application, version=5)

def approval_program():
  # ...
  program = Cond(
    [Txn.application_id() == Int(0), on_creation],
    [Txn.on_completion() == OnComplete.OptIn, Return(Int(0))],
    [Txn.on_completion() == OnComplete.CloseOut, Return(Int(0))],
    [Txn.on_completion() == OnComplete.UpdateApplication, Return(Int(0))],
    [Txn.on_completion() == OnComplete.DeleteApplication, Return(Int(0))],
    [Txn.on_completion() == OnComplete.NoOp, handle_noop]
  )
  compileTeal(program, Mode.Application, version=5)
```

The compileTeal function compiles the program as defined by the program variable. The compileTeal method also sets the Mode.Application to let PyTeal know this is for a smart contract and not a smart signature. The version parameter instructs PyTeal on which version of TEAL to produce when compiling. We don't store any user data in both global and local state, so the clear program just returns 1 for successful method call.

In approval program, `Cond` expression allows several conditions to be chained. The first is condition, and the second is the condition body. If none of the conditions are true the smart contract will return an err and fail. When the contract is first created, the contract’s ID will be equal to 0. After that, all smart contracts will have a unique ID and a unique Algorand address. The first condition checks if this is the first execution of the contract. Next, we will define `on_create`:

```python
# counter_contract.py
def approval_program():
# ...
on_creation = Seq([
  App.globalPut(Bytes("Count"), Int(0)),
  Return(Int(1))
])
# ...
```

The `Seq` is used to provide a sequence of expressions. When this smart contract is first deployed it will store a global variable named `Count` with a value of 0 and immediately return success. Next, we will define `handle_noop`

```python
# counter_contract.py
def approval_program():
# ...
scratchCount = ScratchVar(TealType.uint64)

add = Seq([
  scratchCount.store(App.globalGet(Bytes("Count"))),
  App.globalPut(Bytes("Count"), scratchCount.load() + Int(1)),
  Return(Int(1))
])

deduct = Seq([
  scratchCount.store(App.globalGet(Bytes("Count"))),
  If(scratchCount.load() > Int(0),
    App.globalPut(Bytes("Count"), scratchCount.load() - Int(1)),
  ),
  Return(Int(1))
])

handle_noop = Cond(
  [And(
    Global.group_size() == Int(1),
    Txn.application_args[0] == Bytes("Add")
  ), add],
  [And(
    Global.group_size() == Int(1),
    Txn.application_args[0] == Bytes("Deduct")
  ), deduct],
)
# ...
```

We request scratch memory with size of `uint64` to store data from the global `Count` variable, and we can read and write global state with `App.globalGet` and `App.globalPut`. The condition `Global.group_size() == Int(1)` is used to guarantee that this transaction is not submitted with other transactions in a group. We use the first application argument as function name - `Add` and `Deduct`.

## Deploying and Calling Contract
First, we define algod endpoint and the account for signer. Usually, we use a wallet to sign transactions, and our private keys will be kept secretly in wallet. In this example, we define mnemonic directly in code for practicing, and this shall never be used in production.
```python
creator_mnemoic = "YOUR MNEMONIC"
algod_address = "http://localhost:4001"
algod_token = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
```

Next, we define some helpful functions:
```python
def compile_program(client, source_code):
  compile_response = client.compile(source_code)
  return base64.b64decode(compile_response['result'])

def wait_for_confirmation(client, transaction_id, timeout):
  start_round = client.status()["last-round"] + 1
  current_round = start_round

  while current_round < start_round + timeout:
    try:
      pending_txn = client.pending_transaction_info(transaction_id)
    except Exception:
      return
    if pending_txn.get("confirmed-round", 0) > 0:
      return pending_txn
    elif pending_txn["pool-error"]:
      raise Exception('pool error: {}'.format(pending_txn["pool-error"]))
    client.status_after_block(current_round)
    current_round += 1
  raise Exception("pending tx not found in timeout rounds, timeout value = : {}".format(timeout))

def format_state(state):
  formatted = {}
  for item in state:
    key = item['key']
    value = item['value']
    formatted_key = base64.b64decode(key).decode('utf-8')
    if value['type'] == 1:
      if formatted_key == 'voted':
        formatted_value = base64.b64decode(value['bytes']).decode('utf-8')
      else:
        formatted_value = value['bytes']
      formatted[formatted_key] = formatted_value
    else:
      formatted[formatted_key] = value['uint']
  return formatted

def read_global_state(client, addr, app_id):
  results = client.account_info(addr)
  apps_created = results['created-apps']
  for app in apps_created:
    if app['id'] == app_id:
      return format_state(app['params']['global-state'])
  return {}

def create_app(client, private_key, approval_program, clear_program, global_schema, local_schema):
  sender = account.address_from_private_key(private_key)
  on_complete = transaction.OnComplete.NoOpOC.real
  params = client.suggested_params()
  txn = transaction.ApplicationCreateTxn(sender, params, on_complete, approval_program, clear_program, global_schema, local_schema)
  signed_txn = txn.sign(private_key)
  tx_id = signed_txn.transaction.get_txid()
  client.send_transactions([signed_txn])
  wait_for_confirmation(client, tx_id, 5)
  transaction_response = client.pending_transaction_info(tx_id)
  app_id = transaction_response['application-index']
  print("Created new app_id:", app_id)
  return app_id

def call_app(client, private_key, index, app_args):
  sender = account.address_from_private_key(private_key)
  params = client.suggested_params()
  txn = transaction.ApplicationNoOpTxn(sender, params, index, app_args)
  signed_txn = txn.sign(private_key)
  tx_id = signed_txn.transaction.get_txid()
  client.send_transactions([signed_txn])
  wait_for_confirmation(client, tx_id, 5)
  print("Application called")
```
The function `compile_program` will use algod api to compile from Teal to bytecode. `wait_for_confirmation` is a generic function to check if a transaction is confirmed on the blockchain. Each loop, it gets the TX status by `pending_transaction_info`, and the max number of loop is specified by the number of blocks counting from the current block. The `format_state` will parse key and value to prettier and readable format, and `read_global_state` will read the raw state of an application from the creator account.  The `create_app` and `call_app` can be used for any application. The `create_app` just uses `transaction.ApplicationCreateTxn` to build NoOp TX, next signs TX with private key recovered from mnemonic, then send TX and wait for confirmation, finally, it gets TX status and returns app ID. The `call_app` is similar to `create_app`, but it adds application arguments into NoOp transaction. Finally, we build `main` function for easier use:

```python
def main():
  algod_client = algod.AlgodClient(algod_token, algod_address)
  creator_private_key = mnemonic.to_private_key(creator_mnemoic)
  global_schema = transaction.StateSchema(num_uints=1, num_byte_slices=0)
  local_schema = transaction.StateSchema(num_uints=0, num_byte_slices=0)

  approval_program_compiled = compile_program(algod_client, approval_program())
  clear_state_program_compiled = compile_program(algod_client, clear_state_program())

  print("--------------------------------------------")
  print("Deploying Counter application...")
  app_id = create_app(algod_client, creator_private_key, approval_program_compiled, clear_state_program_compiled, global_schema, local_schema)
  print("Global state:", read_global_state(algod_client, account.address_from_private_key(creator_private_key), app_id))

  print("--------------------------------------------")
  print("Calling Counter application...")
  call_app(algod_client, creator_private_key, app_id, app_args=["Add"])
  print("Global state:", read_global_state(algod_client, account.address_from_private_key(creator_private_key), app_id))

main()
```

When creating application, we must specify how much memory we use for global and local state. For this smart contract, we use only 1 global uint. Run below command and enjoy result:

```text
$ python3 counter_contract.py
--------------------------------------------
Deploying Counter application...
Created new app_id: 51868031
Global state: {'Count': 0}
--------------------------------------------
Calling Counter application...
Application called
Global state: {'Count': 1}
```

Full code: https://github.com/hieutrgvu/algorand-first-contracts/blob/master/counter_contract.py
Reference: https://developer.algorand.org/docs

Thank you for reading my blog. Stay tune for the next parts.