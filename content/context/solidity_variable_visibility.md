---
title: solidity variable visibility
tags:
  - solidity
---

> State variables are variables whose values are permanently stored in contract storage.

## visibility

總共三種： `public`, `private`, `internal`

### `public`

```solidity
contract Foo {
    uint256 public a;
}
```

- 變數資料儲存於 storage region 中，compiler 會為其分配 storage slot
- 當合約被繼承時，依舊可以被繼承的合約使用
- compiler 會自動為變數建立 getter function，可以外部存取

### `internal`

```solidity
contract Foo {
    uint256 internal a;
}
```

- 變數資料儲存於 storage region 中，compiler 會為其分配 storage slot
- 當合約被繼承時，依舊可以被繼承的合約使用

### `private`

```solidity
contract Foo {
    uint256 internal a;
}
```

- 變數資料儲存於 storage region 中，compiler 會為其分配 storage slot
