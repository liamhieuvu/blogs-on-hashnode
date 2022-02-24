## Algorand Starter (Part 4) - Test Scripts

This is the fourth part of Algorand Starter series. In terms of blockchain, safety goes first, so test scripts shall be prepared for all production-ready contracts. This blog will demonstrate how to write test suites with `pytest`.

Algorand Starter series:
+ Part 1: Client side
+ Part 2: Stateful contract (Smart contract)
+ Part 3: Stateless contract (Smart signature)
+ **Part 4: Test scripts**

# Setup Algorand Sandnet using Sandbox
If Sandnet is already setup, skip this step.

```
cd ~
mkdir workspace
cd workspace
git clone https://github.com/algorand/sandbox.git
cd sandbox
```

Start Sandbox as Sandnet (not testnet):

```
./sandbox down
./sandbox clean
./sandbox up -v
```

After some minutes, Sandnet will be ready.

```text
...
algod - goal node status
Last committed block: 0
Time since last block: 0.0s
Sync Time: 2.9s
Last consensus protocol: https://github.com/algorandfoundation/specs/tree/bc36005dbd776e6d1eaf0c560619bb183215645c
Next consensus protocol: https://github.com/algorandfoundation/specs/tree/bc36005dbd776e6d1eaf0c560619bb183215645c
Round for next consensus protocol: 1
Next consensus protocol supported: true
Last Catchpoint:
Genesis ID: sandnet-v1
Genesis hash: 8PaE7Wx/QveyS5Bf21e19rk105L8REwdPD7hpM5LBsA=
```

Please note that Sandbox is not compatible with Docker Compose V2 at the time blog written. You can disable it in Docker configuration (if using Docker Desktop):
![Screen Shot 2022-02-23 at 5.04.50 PM-min.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1645610759497/5Z2aFSVv8.png)

# Setup Python virtual environment
This tutorial will build test scripts for the counter contract described in [Algorand Starter Part 2: Smart Contract](https://liamhieuvu.com/algorand-starter-part-2-smart-contract).

```
cd ~/workspace
git clone https://github.com/liamhieuvu/algorand-first-contracts.git
cd algorand-first-contracts
python3 -m venv venv
source venv/bin/activate
export PYTHONPATH=$(pwd)
export SANDBOX_DIR=~/workspace/sandbox
pip install -r requirements.txt
```

# Project structure
The test modules should be named `test_*. py` or `*_test.py`. The pytest discovery mechanism can find tests anywhere, but we will put all of test files in `tests` directory.

```text
algorand-first-contracts
├── counter_contract.py
├── donation_smart_sig.py
├── requirements.txt
└── tests
    ├── helpers.py
    └── test_counter_contract.py
```

# Structure of a testing module
Pytest allows running a special function named `setup_module` before any tests from the current module is run. In our testing module, we use it to run the Sandbox daemon. Note that we can remove this step if Sandnet is already run.

```python
from helpers import call_sandbox_command

def setup_module(module):
    """Ensure Algorand Sandbox is up prior to running tests from this module."""
    call_sandbox_command("up")
```

A test suite for the counter contract is created and the `setup_class` is run once before all tests (`test_add_deduct`, `test_deduct_below_zero`, `test_two_adds`). We use this to create the needed accounts. If you want a fresh start before each test in the suite, use `setup_method`.

```python
from helpers import fund_accounts

class TestCounterContract:
    """Class for testing the counter contract."""

    def setup_class(self):
        self.deployer = add_standalone_account()
        self.users = [add_standalone_account() for i in range(2)]

        print()
        print("init fund for deployer, users")
        fund_accounts(
            [self.deployer] + self.users,
            [5_000_000] * (1 + len(self.users)),
        )

    def test_add_deduct(self):
        print("deployer creates app")
        # ...

    def test_deduct_below_zero(self):
        print("deployer creates app")
        # ...

    def test_two_adds(self):
        print("deployer creates app")
        # ...
```

# Prepare utilities for app calls

Before going to test suites, we will prepare creation and app call functions. These functions will be used multiple times to fit the logic in test.

```python
from algosdk.future import transaction

from helpers import (
    compile_teal_source,
    suggested_params,
    send_transactions,
)

from counter_contract import approval_program, clear_state_program

class TestCounterContract:
    # ...
    def _create(self):
        txn = transaction.ApplicationCreateTxn(
            sender=self.deployer.get("address"),
            on_complete=transaction.OnComplete.NoOpOC,
            approval_program=compile_teal_source(approval_program()),
            clear_program=compile_teal_source(clear_state_program()),
            global_schema=transaction.StateSchema(num_uints=1, num_byte_slices=0),
            local_schema=transaction.StateSchema(num_uints=0, num_byte_slices=0),
            sp=suggested_params(),
        )
        return send_transactions(self.deployer, [txn]).get("application-index")

    def _add(self, sender, app_id):
        self._noop_call(sender, app_id, b"Add")

    def _deduct(self, sender, app_id):
        self._noop_call(sender, app_id, b"Deduct")

    def _noop_call(self, sender, app_id, method):
        txn = transaction.ApplicationCallTxn(
            sender=sender.get("address"),
            index=app_id,
            on_complete=transaction.OnComplete.NoOpOC,
            app_args=[method],
            accounts=[sender.get("address")],
            sp=suggested_params(),
        )
        send_transactions(sender, [txn])
```

The `_create` function uses `ApplicationCreateTxn` with params of global and local schemas. The counter contract only needs 1 uint. The `_add` and `_deduct` functions use `ApplicationCallTxn` with method param, e.g. "Add" or "Deduct".

# Testing smart contracts implementation
First, we will create an app for testing. Then, we increase the counter with different users and check if the global variable named `Count` is as expected. Finally, we decrease the counter and re-check the global variable.

```python
class TestCounterContract:
    # ...
    def test_add_deduct(self):
        print("deployer creates app")
        app_id = self._create()

        print("user adds")
        self._add(self.users[0], app_id)

        print("user adds")
        self._add(self.users[1], app_id)

        print("user adds")
        self._add(self.users[0], app_id)
        app_global_state = get_app_global_state(app_id)
        assert app_global_state[b"Count"] == 3

        print("user deducts")
        self._deduct(self.users[1], app_id)

        print("user deducts")
        self._deduct(self.users[1], app_id)
        app_global_state = get_app_global_state(app_id)
        assert app_global_state[b"Count"] == 1
```

You can check other 2 test suites in [my repo](https://github.com/liamhieuvu/algorand-first-contracts/blob/master/tests/test_counter_contract.py#L90). Note that, if failed transactions are our expectation in test suits, we can use `with pytest.raises(Exception):`. This will be failed if there is no exception while transaction processing.

# Run test suites
Run all tests

```text
pytest -sv
```

Or we can run a specific test suite

```text
pytest -sv tests/test_counter_contract.py::TestCounterContract::test_two_adds
```

The example result of `pytest -sv`:

```text
============================================= test session starts =============================================
...
collected 3 items                                                                                             

tests/test_counter_contract.py::TestCounterContract::test_add_deduct 
init fund for deployer, users
deployer creates app
user adds
user adds
user adds
user deducts
user deducts
PASSED
tests/test_counter_contract.py::TestCounterContract::test_deduct_below_zero deployer creates app
user deducts counter to below zero but cannot
users add then try to deduct more than add
PASSED
tests/test_counter_contract.py::TestCounterContract::test_two_adds deployer creates app
users opt in to app but cannot, app does not hold local state
user tries to add 2 times in a transaction group
PASSED

======================================== 3 passed in 111.38s (0:01:51) ========================================
```

Reference: [Create and Test Smart Contracts using Python](https://developer.algorand.org/tutorials/create-and-test-smart-contracts-using-python/)



