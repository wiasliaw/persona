---
title: "[context] foundry global config"
tags:
  - foundry
---
Foundry 的設定檔分為 project level 和 global level，可以將一些設定放到 global config 裡面，以避免重複設定

## global config file

global config 為 `$HOME/.foundry/foundry.toml`，建立或是修改 `foundry.toml` 即可

## personal config

筆者是將有關 fork 和 verification 相關的參數都設定在全域，這樣就不需要為每個專案都設定一次 RPC endpoint 和 verification 相關的 api key。設定如下：

```toml
# $HOME/.foundry/foundry.toml
[rpc_endpoints]
mainnet  = "https://..."

[etherscan]
mainnet = { key = "API_KEY", chain = "mainnet" }
```

## benefits

設定之後可以在測試中跑 fork test，填入設定的 endpoint name 即可：

```solidity
import "forge-std/Test.sol";

contract ForkTest is Test {
    function testFork() external {
        vm.createSelectFork("mainnet");
    }
}
```

或是執行 deployment：

```solidity
import "forge-std/Script.sol";

contract ForkTest is Script {
    function run() external {
        vm.createSelectFork("mainnet");
    }
}
```

```bash
forge script ... --rpc-url mainnet
```

相關的 cli 也可以使用已設定的 rpc endpoint，將 endpoint name 可以接在 `--rpc-url` 後面即可：

```bash
cast chain-id --rpc-url mainnet
```

## reference

- https://book.getfoundry.sh/reference/config/overview#global-configuration
