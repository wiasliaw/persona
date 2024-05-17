---
title: "How `try-catch` works in solidity"
tags: solidity
---

`try-catch` 是一般 programming language 常見的錯誤處理機制，可以用來捕捉錯誤並進行處理，但是在 evm 這個設計之下出現了一些怪異的行為。要知道 `try-catch` 如何運作，首先需要知道錯誤如何傳遞出來。

## Opcode overview - `revert`

作為錯誤處理最主要的 opcode，作用是將當前的 execution 停下，並從 memory region 取出一段資料作為 context 的回傳值。

```solidity
assembly {
    revert(offset, size)
}
```

## `require`

最為常見的錯誤處理，只要提供的條件判斷為 false，就會將 reason string 作為錯誤拋出。而 reason string 不是以 `String` 型別拋出，是以 `Error(String)` 型別拋出。

```solidity
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

```solidity
// revert with custom error
error CustomError(string, uint256);
if (condition) revert CustomError("Oops!", 5);
```

```txt
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

### example

以下為觸發 division or modulo by zero 的範例：

```solidity
contract ErrorEmittor {
    function emitPanic(
        uint256 a,
        uint256 b
    ) external pure returns (uint256 c) {
        // emit panic with code 0x12 when b is zero
        c = a / b;
    }
}
```

測試和 Log 如下：

```solidity
import "forge-std/Test.sol";
import {ErrorEmittor} from "../src/ErrorEmittor.sol";

contract ErrorCatcher {
    ErrorEmittor private _instance;

    function setUp() external {
        _instance = new ErrorEmittor();
    }

    function test_catch_panic() external view {
        // call success
        try _instance.emitPanic(10, 2) returns (uint256 c) {
            console.log("get the result: %d", c);
        } catch Panic(uint256) {
            console.log("unreachable log");
        }
        // catch panic
        try _instance.emitPanic(10, 0) returns (uint256) {
            console.log("unreachable log");
        } catch Panic(uint256 panicCode) {
            console.log("get the panicCode: %x", panicCode);
        }
    }
}
```

```console
[PASS] test_catch_panic() (gas: 10256)
Logs:
  get the result: 5
  get the panicCode: 0x12

Traces:
  [10256] ErrorCatcher::test_catch_panic()
    ├─ [342] ErrorEmittor::emitPanic(10, 2) [staticcall]
    │   └─ ← [Return] 5
    ├─ [0] console::log("get the result: %d", 5) [staticcall]
    │   └─ ← [Stop]
    ├─ [286] ErrorEmittor::emitPanic(10, 0) [staticcall]
    │   └─ ← [Revert] panic: division or modulo by zero (0x12)
    ├─ [0] console::log("get the panicCode: %x", 18) [staticcall]
    │   └─ ← [Stop]
    └─ ← [Stop]
```

## `try-catch`

總結一下會被拋出的錯誤有：`Error(string)`, `Panic(uint256)`, `error CustomError()`。

再來回來看 `try-catch` 的行為。首先，`try` 這個關鍵字後面只能接「external function 的呼叫」或是「透過 `new` 關鍵字去建立一個新的合約」

```solidity
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

呼叫後的回傳的資料會透過 returncodecopy opcode 存入 memory region。接著 `catch` 關鍵字後面會附上錯誤資訊的型別並將存入 memory region 的資料做 abi decode，最後由後面的邏輯處理。以下為例，一個用來捕捉 `Error(string)`，另一個用來捕捉 `Panic(uint256)`：

```solidity
address private _addr;
try IERC20(_addr).transfer(from, to, amount) returns (bool) {
    ...
} catch Error(string memory reason) {
    // handle reason string
} catch Panic(uint errorCode) {
    // handle Panic
}
```

而 `catch` 沒有支援捕捉 custom error

```solidity
// ❌
catch CustomError() {
    ...
}
```

如果拋出的錯誤不是 `Error(string)` 或是 `Panic(uint256)`，可以寫一個 default catch 做捕捉。default catch 有兩種寫法：`catch (bytes memory data) {...}` 和 `catch {...}`。[官方文件的寫法](https://docs.soliditylang.org/en/v0.8.24/control-structures.html#try-catch)會讓你以為兩種寫法是可以同時存在的，但是 default catch 只能有一個。**這兩種寫法的差異只在於需不需要錯誤的資訊而已**。

被遺忘的 Custom Error 則可以在 `catch (bytes memory){}` 中以 `bytes memory` 型別被捕捉，開發者可以自行做 abi decode 處理：

```solidity
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

```solidity
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

## `try-catch` disadvantage

try-catch 不好用的原因之一是沒有辦法捕捉 custom error 前面已經提過了；另外一個原因就是就算用了 try-catch 也還是有捕捉不了的錯誤

這裡舉例兩個會讓 `try-catch` 無法按照預期捕捉錯誤的情況：

### reason 1: decode issue

先前提到 `try-catch` 會對 revert 回傳的資料做 abi decode，但是如果 decode 的過程中發生錯誤時，錯誤不會在原本預計的 catch block 被捕捉。

以下合約會以 `"cat"` 作為錯誤資訊。`isCorrectLen` 會調整 revert 回傳的資料長度，正確的回傳長度為 71(0x47)，長度小於 71 則會使 abi decode 發生錯誤：

```txt
// memory layout, total len = 0x04 + 0x20 + 0x20 + 0x03

[0x40]: 0x8c379a0      | error selector of `Error(string)`, len = 0x04
[0x60]: 0x20           | string offset, len = 0x20
[0x80]: 0x03           | string length, len = 0x20
[0xa0]: 'cat'          | string data, len = 0x03
```

```solidity
contract Emit {
    function revv(bool isCorrectLen) external {
        uint256 len = isCorrectLen ? 0x47 : 0x44;

        assembly {
            let ptr := 0x40
            mstore(ptr, 0x08c379a0)          // error selector of `Error(string)`
            mstore(add(ptr, 0x20), 0x20)     // string offset
            mstore(add(ptr, 0x43), 0x636174) // 'cat'
            mstore(add(ptr, 0x40), 0x3)      // string length = 3
            revert(add(ptr, 0x1c), len)
        }
    }
}
```

測試和 log 如下，因為 decode 成 `Error(string)` 中出現錯誤，所以只能以 `bytes memory` 的型別被捕捉：

```solidity
contract Catcherrr {
    Emit private immutable emitter;

    constructor() {
        emitter = new Emit();
    }

    function test_cat() external {
        try emitter.revv(false) {
            console.log("call success");
        } catch Error(string memory reason) {
            console.logString(reason);
        } catch (bytes memory err) {
            console.logBytes(err);
        }
        console.log("error had been caught");
    }
}
```

```txt
[PASS] test_cat() (gas: 7805)
Logs:
  0x08c379a000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000003
  error had been catched

Traces:
  [7805] Catcherrr::test_cat()
    ├─ [357] Emit::revv(false)
    │   └─ ← [Revert]
    ├─ [0] console::logBytes(0x08c379a000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000003) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("error had been catched") [staticcall]
    │   └─ ← [Stop]
    └─ ← [Stop]
```

### reason 2: return bomb

return bomb 也是一個有趣的議題。在 abi decode 之前，需要將 revert 回傳的資料儲存在 memory region 裡面，而存取超出當前 memory region 範圍的資料時，則會觸發 memory expansion 去擴展 memory region 的範圍。memory expansion 是需要消耗 gas，如果 revert 回傳的資料過於龐大，則會消耗掉大量的 gas 並讓交易 revert 掉。所以如果要嘗試去捕捉未知合約發出來的錯誤，是有可能捕捉到一顆 gas bomb 的。

以下以給定的 revert length 和 gas 為例 (主要從 [karam](https://twitter.com/0xkarmacoma) 的[範例](https://github.com/karmacoma-eth/halmos-sandbox/blob/193236164d19f16b0f47bf1f1a09c442afafc16a/test/66_returnbomb.t.sol#L9)做修改)：

```solidity
contract PlasticBombs {
    function bomb() external {
        assembly {
          // revert with hardcode length
            revert(0x00, 0xffff)
        }
    }
}

contract Trigger {
    PlasticBombs private _bombs = new PlasticBombs();

    event LogCase1(bytes);
    event LogCase2();

    function tryCase1() external {
        try _bombs.bomb() {
            // unreachable
        } catch (bytes memory data) {
            // default branch and care about error information
            emit LogCase1(data);
        }
    }

    function tryCase2() external {
        try _bombs.bomb() {
            // unreachable
        } catch {
            // default branch without error information
            emit LogCase2();
        }
    }
}
```

測試和 Log 如下：

```solidity
contract TestAnother is Test {
    Trigger private _instance = new Trigger();

    function test_case1() external {
        // hardcode the gas
        (bool succ,) = address(_instance).call{gas: 100_000}(abi.encodeCall(Trigger.tryCase1, ()));
        assertTrue(succ);
    }

    function test_case2() external {
        // hardcode the gas
        (bool succ,) = address(_instance).call{gas: 100_000}(abi.encodeCall(Trigger.tryCase2, ()));
        assertTrue(succ);
    }
}
```

```console
[FAIL. Reason: assertion failed] test_case1() (gas: 108189)
Traces:
  [108189] TestAnother::test_case1()
    ├─ [100000] Trigger::tryCase1()
    │   ├─ [14446] PlasticBombs::bomb()
    │   │   └─ ← [Revert]
    │   └─ ← [OutOfGas] EvmError: OutOfGas
    ├─ [0] VM::assertTrue(false) [staticcall]
    │   └─ ← [Revert] assertion failed
    └─ ← [Revert] assertion failed

[PASS] test_case2() (gas: 28600)
Traces:
  [28600] TestAnother::test_case2()
    ├─ [20341] Trigger::tryCase2()
    │   ├─ [14446] PlasticBombs::bomb()
    │   │   └─ ← [Revert]
    │   ├─ emit LogCase2()
    │   └─ ← [Stop]
    ├─ [0] VM::assertTrue(true) [staticcall]
    │   └─ ← [Return]
    └─ ← [Stop]
```

從 `test_case1` 的 log 可以看出，external call 的執行是成功的，但是 revert 回傳的資料長度觸發 memory expansion 將所有的 gas 都消耗掉了。而從 `test_case2` 可以看出，default branch `catch {}` 並不會將 revert 回傳的資料存入 memory region 也不會對其做 abi decode。

## Conclusion

回顧一下 `try-catch` 做了什麼事：

1. 呼叫外部合約
2. 如果需要處理錯誤資訊，則將資料存入 memory region (可能是 success 或是 revert)
3. 將資料 decode 之後由 try block 或是 catch block 處理

現在越來越多的合約都轉向使用 Custom Error 來節省 gas 開銷，只能針對 external function 但又沒辦法按照預期捕捉 Custom Error 的 `try-catch` 用起來就不是那麼方便。

如果只是單純不要讓交易 revert，這樣寫 `try Call() catch {}` 是可行的，不需要處理 revert 回傳的資料，就不會有 returndatacopy 造成 return bomb。但是如果要處理 revert 回傳的資料，請用在**可信任的合約**或是**在 protocol 內部做錯誤處理**。

## Reference

- https://twitter.com/transmissions11/status/1516621010270769162
- https://twitter.com/0xkarmacoma/status/1763746082537017725
