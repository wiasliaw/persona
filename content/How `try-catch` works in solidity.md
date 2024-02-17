---
tags:
  - solidity
---

`try-catch` 是一般 programming language 常見的錯誤處理機制，可以用來捕捉錯誤並進行處理，但是在 evm 這個設計之下出現了一些怪異的行為。要知道 `try-catch` 如何運作，首先需要知道錯誤如何傳遞出來。

## opcode overview - `revert`

將當前的 execution 停下，並從 memory region 取出一段資料作為 context 的回傳值。

```sol
assembly {
    revert(offset, size)
}
```

## `require`

最為常見的錯誤處理，只要提供的條件判斷為 false，就會將 reason string 作為錯誤拋出。而 reason string 不是以 `String` 型別拋出，是以 `Error(String)` 型別拋出。

```sol
require(condition, "reason string")
```

編譯成 evm bytecode 之後，可以看到 `revert` opcode 的存在，並從 memory region 裡面取出資料做拋出。

```txt
// 編譯後為 `revert(offset, size)`
// memory layout
[offset]   : 0x08c379a0 // bytes4, error selector of `Error(String)`
[offset+4] : 0x20       // bytes32, string's offset (it is a reference type)
[offset+24]: size       // bytes32, string length
[offset+44]: 0x...      // hex value of "reason string"
```

## `revert`

在 Custom Error 出現之後越來越常被使用，主因是 Custom Error 不能和 `require` 搭配使用，且 `revert` 有向後兼容，所以 `revert` 也可以處理 reason string 並以 `Error(String)` 型別做拋出。

`revert` 處理 reason string 的編譯結果和 memory layout 與 `require` 處理 reason string 基本相同，所以以 Custom Error 為例。

```sol
// revert with custom error
error CustomError(string, uint256);
if (condition) revert CustomError("Oops!", 5);

// 編譯後為 `revert(offset, size)`
// memory layout
[ptr]   : 0x....       // error selector of CustomError();
[ptr+4] : 0x40         // bytes32, String's offset (it is a reference type)
[ptr+24]: 0x05         // uint256
[ptr+44]: string_size  // string length
[ptr+64]: string_data  // hex data of string
```

## `assert`

雖然幾乎不會用到，但是還是需要提一下。`assert` 主要用於處理 panic 和 invariants，會以 `Panic(uint256)` 作為錯誤型別拋出。

### panic error code

Solidity compiler 會在一些情況下將錯誤以 panic 的方式處理，這些情況在 docs 已經整理成一個表格如下：

| code | description                                                                                                                             |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------- |
| 0x00 | Used for generic compiler inserted panics                                                                                               |
| 0x01 | If you call `assert` with an argument that evaluates to false                                                                           |
| 0x11 | If an arithmetic operation results in underflow or overflow outside of an `unchecked { ... }` block                                     |
| 0x12 | If you divide or modulo by zero (e.g. `5 / 0` or `23 % 0`)                                                                              |
| 0x21 | If you convert a value that is too big or negative into an enum type                                                                    |
| 0x22 | If you access a storage byte array that is incorrectly encoded                                                                          |
| 0x31 | If you call `.pop()` on an empty array                                                                                                  |
| 0x32 | If you access an array, `bytesN` or an array slice at an out-of-bounds or negative index (i.e. `x[i]` where `i >= x.length` or `i < 0`) |
| 0x41 | If you allocate too much memory or create an array that is too large                                                                    |
| 0x51 | If you call a zero-initialized variable of internal function type                                                                       |

### invariants

Solidity compiler 有內建簡易的[形式化驗證的工具](https://docs.soliditylang.org/en/v0.8.24/smtchecker.html)，可以對一些簡單的行為做形式化驗證。不建議使用，要做形式化驗證可以找更專業的工具。在 Foundry 可以修改設定來啟用：

```sol
// FV.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract FormalVerification {
    function add(uint256 a, uint256 b) external pure returns (uint256 ret) {
        ret = a + b;
        assert(ret == a + b);
    }
}
```

```toml
# foundry.toml
[profile.default.model_checker]
contracts = {'./src/FV.sol' = ['FormalVerification']}
engine = 'chc'
targets = ['assert', 'underflow', 'overflow']
timeout = 100000
```

Logs 如下，抓到並給出在什麼樣的輸入下會有 Overflow

```txt
Warning (4984): CHC: Overflow (resulting value larger than 2**256 - 1) happens here.
Counterexample:

a = 115792089237316195423570985008687907853269984665640564039457584007913129639935
b = 1
ret = 0

Transaction trace:
FormalVerification.constructor()
FormalVerification.add(115792089237316195423570985008687907853269984665640564039457584007913129639935, 1)
Counterexample:

a = 115792089237316195423570985008687907853269984665640564039457584007913129639935
b = 1
ret = 0

Transaction trace:
FormalVerification.constructor()
FormalVerification.add(115792089237316195423570985008687907853269984665640564039457584007913129639935, 1)
 --> src/FV.sol: 6:15:
  |
6 |         ret = a + b;
  |               ^^^^^

Info (1391): CHC: 2 verification condition(s) proved safe! Enable the model checker option "show proved safe" to see all of them.
```

## `try-catch`

總結一下會被拋出的錯誤有：`Error(string)`, `Panic(uint256)`, `error CustomError()`。再來回來看 `try-catch` 的行為。

首先，`try` 這個關鍵字後面只能接「external function 的呼叫」或是「透過 `new` 關鍵字去建立一個新的合約」

```sol
// external function call
address private _addr;
try IERC20(_addr).transfer(from, to, amount) returns (bool) {
    ...
}

// new contract
try new ERC20("sample", "SMT") returns (ERC20 erc20) {
    ...
}
```

接著 `catch` 關鍵字的後面要由拋出錯誤的資訊並在之後的 block 中處理。一個用來捕捉 `Error(string)`，另一個用來捕捉 `Panic(uint256)`

```sol
address private _addr;
try IERC20(_addr).transfer(from, to, amount) returns (bool) {
    ...
} catch Error(string memory reason) {
    // handle reason string
} catch Panic(uint errorCode) {
    // handle Panic
}
```

奇怪的地方來了，`catch` 沒有支援捕捉 custom error

```sol
❌
catch CustomError() {
    ...
}
```

如果拋出的錯誤不是 `Error(string)` 或是 `Panic(uint256)`，可以寫一個 default catch 做捕捉。[官方文件的寫法](https://docs.soliditylang.org/en/v0.8.24/control-structures.html#try-catch)會讓你以為兩種寫法是可以同時存在的，但是 default catch 只能有一個。這兩種寫法的差異只在於需不需要錯誤的資訊而已。被遺忘的 Custom Error 則會在這裡以 `bytes memory` 的型別被捕捉。

```sol
address private _addr;
try IERC20(_addr).transfer(from, to, amount) returns (bool) {
    ...
} catch Error(string memory reason) {
    ...
} catch Panic(uint errorCode) {
    ...
} catch (bytes memory data) {
    ...
}
```

**or**

```sol
address private _addr;
try IERC20(_addr).transfer(from, to, amount) returns (bool) {
    ...
} catch Error(string memory reason) {
    ...
} catch Panic(uint errorCode) {
    ...
} catch {
    ...
}
```

## `try-catch` Cons

### decode error

`revert` opcode 會從 memory region 取出來的資料作為錯誤資訊拋出，到 `try-catch` 時會去做 decode，如果 decode 出現錯誤，則不會被捕捉到。

以下範例以會回傳 `cat` 作為錯誤資訊，現在將回傳的資訊裁掉一部分讓 `try-catch` 沒有辦法 decode，

```sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";

contract Emit {
    function revv(uint256 id) external pure {
        // [0x40] 0x0000..error selector
        // [0x60] string offset
        // [0x80] string length
        // [0x100] 'cat'
        assembly {
            let ptr := 0x40
            mstore(ptr, 0x08c379a0) // error selector of `Error(string)`
            mstore(add(ptr, 0x20), 0x20) // string offset
            mstore(add(ptr, 0x43), 0x636174) // 'cat'
            mstore(add(ptr, 0x40), 0x3) // string length = 3
            revert(add(ptr, 0x1c), 0x63)
        }
    }
}

contract Catcherrr {
    Emit private immutable emitter;

    constructor() {
        emitter = new Emit();
    }

    function test_cat() external {
        try emitter.revv(0) {
            console.log("call success");
        } catch Error(string memory reason) {
            console.logString(reason);
        }
        console.log("error had been catched");
    }
}
```

從 log 可以看到 `error had been catched` 沒有被印出來，`try-catch` 捕捉不到錯誤。

```
Running 1 test for src/RetDecodeF.sol:Catcherrr
[FAIL. Reason: ] test_cat() (gas: 3445)
Traces:
  [3445] Catcherrr::test_cat()
    ├─ [241] Emit::revv(0) [staticcall]
    │   └─ ←
    └─ ←

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 5.66ms
```

## conclusion

現在越來越多的合約都轉向使用 Custom Error 來節省 gas 開銷，只能針對 external function 又沒辦法完全捕捉 Custom Error 的 `try-catch` 還是少用比較好。

## reference

- https://twitter.com/_prestwich/status/1516621028998275076
- https://twitter.com/transmissions11/status/1516621010270769162
