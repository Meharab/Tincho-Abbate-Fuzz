# Fuzz Exercise by Tincho Abbate
This is a short and simple exercise to start sharpening your smart contract fuzzing skills with Foundry.

The scenario is simple. There's a registry contract that allows callers to register by paying a fixed fee in ETH. If the caller sends too little ETH, execution should revert. If the caller sends too much ETH, the contract should give back the change.

Things look good according to the unit test we coded in the `Registry.t.sol` contract.

Your goal is to code at least one fuzz test for the Registry contract. By following the brief specification above, the test must be able to detect a bug in the register function.

## [H-1] Potential loss of funds can occur if the user exceeds the fixed price of `1e18`
**Description:** When invoking `Registry::register`, a participant can exceed the fixed price of `1e18`, as there are no additional checks to verify if the price has been exceeded or no actions to send the user their change.

**Impact:** The participant's balance is not refunded when exceeding the fixed price of `1e18`, potentially resulting in a loss of funds

**Proof of Concepts:**
```solidity
    function test_fuzzer(uint256 amount) public {
        amount = bound(amount, 2e18, 10e18);
        vm.deal(alice, amount);
        console.log("Alice's pre balance: %s", address(alice).balance);
        
        vm.prank(alice);
        registry.register{value: amount}();
        console.log("Registry's post balance: %s", address(registry).balance);

        // registry balance should not exceed the fixed price of 1e18 
        assert(address(registry).balance > 1e18);
        // no change is given back to the sender after exceeding the entry fixed price of 1e18 
        console.log("Alice's post balance: %s", address(alice).balance);
    }
```
```bash
    [47154] RegistryTest::test_fuzzer(43)
        ├─ [0] console::log("Bound result", 8000000000000000044 [8e18]) [staticcall]
        │   └─ ← [Stop] 
        ├─ [0] VM::deal(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], 8000000000000000044 [8e18])
        │   └─ ← [Return] 
        ├─ [0] console::log("Alice's pre balance: %s", 8000000000000000044 [8e18]) [staticcall]
        │   └─ ← [Stop] 
        ├─ [0] VM::prank(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6])
        │   └─ ← [Return] 
        ├─ [22318] 0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f::register{value: 8000000000000000044}()
        │   └─ ← [Stop] 
        ├─ [0] console::log("Registry post balance: %s", 8000000000000000044 [8e18]) [staticcall]
        │   └─ ← [Stop] 
        ├─ [0] console::log("Alice's post balance: %s", 0) [staticcall]
        │   └─ ← [Stop] 
        └─ ← [Stop] 

    Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 22.74ms (21.18ms CPU time)
```

**Recommended Mitigation:** Prevent the entire transaction from going through if msg.value exceeds `1e18`.

```diff
+   error Registry__PaymentExceedsPrice(uint256 expected, uint256 actual);
    
    function register() external payable {
        if(msg.value < PRICE) {
            revert PaymentNotEnough(PRICE, msg.value);
        }
+       if (msg.value > PRICE) { 
+           revert Registry__PaymentExceedsPrice(PRICE, msg.value); 
+       }

       registry[msg.sender] = true;
    }
```

Alternatively, you can implement a refund mechanism that forwards the remaining amount to `msg.sender` while registering the participant:

```diff
+   error Registry__RefundFailed(uint256 amount);

    function register() external payable {
        if(msg.value < PRICE) {
            revert PaymentNotEnough(PRICE, msg.value);
        }
+       if (msg.value > PRICE) { 
+       uint256 refundAmount = msg.value - PRICE;
+       (bool ok, ) = msg.sender.call{value: refundAmount}("");
+           if (!ok) { revert Registry__RefundFailed(refundAmount); }
+        }

        registry[msg.sender] = true;    
    }
```


## Foundry

**Foundry is a blazing fast, portable and modular toolkit for Ethereum application development written in Rust.**

Foundry consists of:

- **Forge**: Ethereum testing framework (like Truffle, Hardhat and DappTools).
- **Cast**: Swiss army knife for interacting with EVM smart contracts, sending transactions and getting chain data.
- **Anvil**: Local Ethereum node, akin to Ganache, Hardhat Network.
- **Chisel**: Fast, utilitarian, and verbose solidity REPL.

## Documentation

https://book.getfoundry.sh/

## Usage

### Build

```shell
$ forge build
```

### Test

```shell
$ forge test
```

### Format

```shell
$ forge fmt
```

### Gas Snapshots

```shell
$ forge snapshot
```

### Anvil

```shell
$ anvil
```

### Deploy

```shell
$ forge script script/Counter.s.sol:CounterScript --rpc-url <your_rpc_url> --private-key <your_private_key>
```

### Cast

```shell
$ cast <subcommand>
```

### Help

```shell
$ forge --help
$ anvil --help
$ cast --help
```
