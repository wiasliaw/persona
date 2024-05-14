---
title: "[context] enable formal verification in foundry"
tags:
  - context
  - foundry
---

Solidity compiler 內建一些 formal verification 的設定 (參考文件為 v0.8.25)，簡介如下：

## config

### `contracts`

- 指定哪些合約需要被 model checker 分析
- 給定合約的路徑，以及合約名稱

```toml
[profile.default.model_checker]
contracts = { "src/MyContracts.sol" = ["ContractA", "ContractB"] }
```

### `engine`

- 指定使用哪個 model checker engine
- 選項有：`all`, `bmc`, `chc`, `none`

```toml
[profile.default.model_checker]
engine = "chc" # type: string
```

### `solvers`

- 指定使用哪個 model checker solver
- 選項有：`cvc4`, `eld`, `smtlib2`, `z3`
- 建議選擇 z3，z3 可以同時支援 bmc 和 chc engine

```toml
[profile.default.model_checker]
solvers = ["z3"] # type: string[]
```

### `invariants`

- 內建的 invariants，目前只能檢查兩個 invariants
  - contract invariant
  - reentrancy invariant

```toml
[profile.default.model_checker]
invariants = ["contract", "reentrancy"] # type: string[]
```

### `targets`

- SMTCheck 建立的 target
- 選項有
  - `assert` 針對 `assert()` 的 condition 做驗證
  - `underflow`
  - `overflow`
  - `divByZero`
  - `constantCondition`
  - `popEmptyArray`
  - `outOfBounds`

```toml
[profile.default.model_checker]
targets = ["assert", "underflow"] # type: string[]
```

## example

```solidity
contract FVSample {
    function allow_age_18_for_alcoholic(uint8 age) external pure returns (bool) {
        assert(age >= 18); // assert failed
        return true;
    }

    function addition(uint256 a, uint256 b) external pure returns (uint256 c) {
        c = a + b; // overflow
    }

    function subtraction(uint256 a, uint256 b) external pure returns (uint256 c) {
        c = a - b; // underflow
    }
}
```

```toml
[profile.default.model_checker]
contracts = { "src/FV.sol" = ["FVSample"] }
engine = "chc"
solvers = ["z3"]
invariants = ["contract", "reentrancy"]
targets = ["assert", "underflow", "overflow"]
```

Log 如下：

```console
Warning (6328): CHC: Assertion violation happens here.
Counterexample:

age = 0
 = false

Transaction trace:
FVSample.constructor()
FVSample.allow_age_18_for_alcoholic(0)
Warning: CHC: Assertion violation happens here.
Counterexample:

age = 0
 = false

Transaction trace:
FVSample.constructor()
FVSample.allow_age_18_for_alcoholic(0)
 --> src/FV.sol: 6:9:
  |
6 |         assert(age >= 18);
  |         ^^^^^^^^^^^^^^^^^

Warning (4984): CHC: Overflow (resulting value larger than 2**256 - 1) happens here.
Counterexample:

a = 1
b = 115792089237316195423570985008687907853269984665640564039457584007913129639935
c = 0

Transaction trace:
FVSample.constructor()
FVSample.addition(1, 115792089237316195423570985008687907853269984665640564039457584007913129639935)
Warning: CHC: Overflow (resulting value larger than 2**256 - 1) happens here.
Counterexample:

a = 1
b = 115792089237316195423570985008687907853269984665640564039457584007913129639935
c = 0

Transaction trace:
FVSample.constructor()
FVSample.addition(1, 115792089237316195423570985008687907853269984665640564039457584007913129639935)
  --> src/FV.sol:11:13:
   |
11 |         c = a + b;
   |             ^^^^^

Warning (3944): CHC: Underflow (resulting value less than 0) happens here.
Counterexample:

a = 0
b = 1
c = 0

Transaction trace:
FVSample.constructor()
FVSample.subtraction(0, 1)
Warning: CHC: Underflow (resulting value less than 0) happens here.
Counterexample:

a = 0
b = 1
c = 0

Transaction trace:
FVSample.constructor()
FVSample.subtraction(0, 1)
  --> src/FV.sol:15:13:
   |
15 |         c = a - b;
   |             ^^^^^
```

## reference

- https://docs.soliditylang.org/en/latest/smtchecker.html
- https://book.getfoundry.sh/reference/config/solidity-compiler#model-checker
