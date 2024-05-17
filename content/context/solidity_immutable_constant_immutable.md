---
title: solidity constant and immutable variable
tags:
  - solidity
---

solidity 可以將變數註記為 `constant` 或是 `immutable`

## `constant`

```solidity
contract Foo {
    uint256 constant public bar = 1_000;
}
```

- 將變數註記為 `constant` 時就需要賦值，合約部署後就無法修改
- 由於是 hardcode 在 bytecode 裡面，不需要儲存在 storage region，也不會被分配 storage slot

## `immutable`

```solidity
contract Foo {
    uint256 constant immutable bar;

    constructor() {
        bar = 1_000;
    }
}
```

- 特性和 constant 幾乎相同，而 immutable 可以在 constructor 才賦值，合約部署後就無法修改
- 由於是 hardcode 在 bytecode 裡面，不需要儲存在 storage region，也不會被分配 storage slot
