## Solana for Beginners

Solana development tutorial covers core concepts (programming models and terminology), smart contract, rust language keywords, contract deployment, and contract interaction on Solana cluster.

# Content

![content.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1629880812858/7dh38vL7y.png)

Presentation file: https://docs.google.com/presentation/d/1EAQ300mamB1v8sRn0xpCcqug20LcLk1xtQTWPcZg5kQ/edit#slide=id.ge8e43e7bf9_0_503

# 1. Programing Model
### Reference
- https://docs.solana.com/developing/programming-model/overview
- https://docs.solana.com/terminology

### Basic terms
**Cluster**: A set of validators maintaining a single instance of the Solana blockchain, e.g. localhost, Testnet, Mainnet Beta, etc

**Accounts**: A file living on chain with address (pubkey) and must pay rent for space usage on chain

**Programs**: Accounts that are marked executable (smart contracts)

**Account Ownership**: Accounts are owned by programs which are indicated by a program id in the metadata `owner` field.

**Instructions**: The smallest unit of a program that a client can include in a transaction.

**Native Programs**:
- System Program:  Create accounts, assign account ownership.
- BPF Loader: For deployment, upgrades, instruction execution

# 2. Smart Contract
## Fundamental
- Solana smart contracts are called programs → read-only
- Stored on an account → account public key is program id
- Another account for storing app data


![smart-contract-fundamental (1).png](https://cdn.hashnode.com/res/hashnode/image/upload/v1629881738550/6EiTh2R76.png)

## Memory Management
- appAccount has fixed amount of memory for raw data
- Data structure is defined in program

![smart-contract-flow.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1629881654815/Kr9QHk-Nr.png)

## Programming Languages
- Can use Rust or C
- Quickly learn Rust with: https://doc.rust-lang.org/rust-by-example
- Rust keywords to focus: Primitives, custom type, flow of control, functions, modules, cargo, scoping rules, traits, error handling, testing

## Hello World Program
- Purpose: Count the greeting number of app account
- Reference: https://github.com/solana-labs/example-helloworld/tree/master/src/program-rust
- Project layout:

```
project
|- src           : source code
|- Cargo.toml    : package dependency
`- Xargo.toml    : sysroot manager
``` 

- `src/lib.rs`:

Include library for unpack/pack raw data &#8594; Include standard libraries &#8594; Define entry point:

```rust
use borsh::{BorshDeserialize, BorshSerialize};

use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};

entrypoint!(process_instruction);
```

Define data structure &#8594; Get app account in entry function:

```rust
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct GreetingAccount {
    pub counter: u32,
}

pub fn process_instruction(
    program_id: &Pubkey        // program acc pub key
    accounts: &[AccountInfo],  // app acc pub key
    _instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello World Rust program entrypoint");
    let accounts_iter = &mut accounts.iter();
    let account = next_account_info(accounts_iter)?;
    ...
```

Check ownership:

```rust
    if account.owner != program_id {
        return Err(ProgramError::IncorrectProgramId);
    }
```

Unpack data and increase counter by 1:

```rust
    let mut greeting_account = GreetingAccount::try_from_slice(&account.data.borrow())?;
    greeting_account.counter += 1;
```

Pack data and push to app account:

```rust
    greeting_account.serialize(&mut &mut account.data.borrow_mut()[..])?;
    msg!("Greeted {} time(s)!", greeting_account.counter);
    Ok(())
```

# 3. Contract Deployment
- Tool: Use script ([@solana/web3.js](https://solana-labs.github.io/solana-web3.js)  recommended) or command line
- Cluster: Start local cluster with https://github.com/solana-labs/example-helloworld
- Reference: https://github.com/solana-labs/example-helloworld/tree/master/src/client
- `deployer.js`:

Establish connection to RPC &#8594; Init payer account &#8594; Request 1 SOL airdrop to pay the fee &#8594; Init program account:

```js
const connection = new solanaWeb3.Connection('http://localhost:8899');

const payerAccount = new solanaWeb3.Account();
const res = await connection.requestAirdrop(payerAccount.publicKey, 1000000000);

const programAccount = new solanaWeb3.Account();
```

Read compiled contract &#8594; Load code on chain:

```js
const data = await fs.readFile('./directory/file.so');

await solanaWeb3.BpfLoader.load(
  connection,
  payerAccount,
  programAccount,
  data,
  solanaWeb3.BPF_LOADER_PROGRAM_ID,
);

const programId = programAccount.publicKey;
```

Setup app account with: required space, lamports to rent space, and program id for ownership

```js
const appAccount = new solanaWeb3.Account();

const transaction = new solanaWeb3.Transaction().add(
  solanaWeb3.SystemProgram.createAccount({
    fromPubkey: payerAccount.publicKey,
    newAccountPubkey: appAccount.publicKey,
    lamports: 5000000000,
    space: DATA_STRUCT_SIZE,
    programId,
  }),
);

await solanaWeb3.sendAndConfirmTransaction(
  connection, transaction,
  [payerAccount, appAccount],
);
```

# 4. Contract Interaction

Create an instruction with no data and send it the program

- `client.js`:

```js
const instruction = new solanaWeb3.TransactionInstruction({
  keys: [{pubkey: appAccount.publicKey, isSigner: false, isWritable: true}],
  programId: app.programId,
  data: Buffer.alloc(0),
});

const confirmation = await solanaWeb3.sendAndConfirmTransaction(
  connection,
  new solanaWeb3.Transaction().add(instruction),
  [payerAccount],
);
```



