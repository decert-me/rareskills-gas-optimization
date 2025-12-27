在进行 gas 优化之前，首先需要准确地测量和评估代码的 gas 消耗。本章将介绍几种常用的 gas 评估方法，帮助你识别优化点并验证优化效果。

## 为什么需要 Gas 评估

Gas 评估是优化工作的基础：

1. **建立基线** - 了解当前代码的 gas 消耗情况
2. **识别瓶颈** - 找出最消耗 gas 的操作
3. **验证优化效果** - 对比优化前后的差异
4. **防止性能退化** - 在 CI/CD 中监控 gas 消耗变化

## 1. 使用 Foundry 的 Gas Report

[Foundry](https://learnblockchain.cn/article/22641) 提供了强大的 gas 报告功能，可以自动统计测试中每个函数的 gas 消耗。

### 1.1 基础用法

在运行测试时添加 `--gas-report` 标志：

```bash
forge test --gas-report
```

输出示例：
```
| src/MyToken.sol:MyToken contract |                 |       |        |       |         |
|----------------------------------|-----------------|-------|--------|-------|---------|
| Deployment Cost                  | Deployment Size |       |        |       |         |
| 500000                           | 2500            |       |        |       |         |
| Function Name                    | min             | avg   | median | max   | # calls |
| transfer                         | 51343           | 51343 | 51343  | 51343 | 1       |
| mint                             | 48000           | 48000 | 48000  | 48000 | 2       |
| approve                          | 46000           | 46000 | 46000  | 46000 | 1       |
```

### 1.2 配置 Gas Report

在 `foundry.toml` 中配置 gas 报告选项：

```toml
[profile.default]
gas_reports = ["*"]  # 报告所有合约
# gas_reports = ["MyToken", "MyNFT"]  # 只报告特定合约

# 排除某些合约
gas_reports_ignore = ["MockContract", "TestHelpers"]
```

### 1.3 测试示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/MyToken.sol";

contract MyTokenTest is Test {
    MyToken token;
    address user1 = address(0x1);
    address user2 = address(0x2);

    function setUp() public {
        token = new MyToken();
        token.mint(user1, 1000 ether);
    }

    // 测试 transfer 的 gas 消耗
    function testTransferGas() public {
        vm.prank(user1);
        token.transfer(user2, 100 ether);
    }

    // 测试 approve 的 gas 消耗
    function testApproveGas() public {
        vm.prank(user1);
        token.approve(user2, 100 ether);
    }

    // 对比不同场景的 gas 消耗
    function testFirstTransferVsSubsequent() public {
        // 首次转账（冷存储 -> 热存储）
        vm.prank(user1);
        token.transfer(user2, 100 ether);

        // 后续转账（热存储 -> 热存储）
        vm.prank(user1);
        token.transfer(user2, 100 ether);
    }
}
```

运行测试：
```bash
forge test --gas-report --match-contract MyTokenTest
```

### 1.4 Gas Report 的优势

✅ **自动化** - 无需修改合约代码
✅ **统计全面** - 显示 min/avg/median/max 值
✅ **易于对比** - 直观展示所有函数的 gas 消耗

## 2. 使用 Gas Snapshots

Gas Snapshots 是 Foundry 的另一个强大功能，用于跟踪 gas 消耗的变化。

### 2.1 创建 Snapshot

首次运行以下命令创建 gas 快照：

```bash
forge snapshot
forge snapshot --snap <FILE_NAME> # 自定义的gas 快照文件名
```

默认会在项目根目录生成一个 `.gas-snapshot` 文件，记录每个测试的 gas 消耗：

```
MyTokenTest:testApproveGas() (gas: 46123)
MyTokenTest:testFirstTransferVsSubsequent() (gas: 102456)
MyTokenTest:testTransferGas() (gas: 51343)
```



### 2.2 对比 Snapshot

修改代码后，再次运行 `forge snapshot`，Foundry 会自动对比并显示差异：

```bash
forge snapshot --diff
```

输出示例：
```
MyTokenTest:testTransferGas() (gas: 51243 (↓ -100 | -0.19%))
MyTokenTest:testApproveGas() (gas: 46123 (no change))
MyTokenTest:testMintGas() (gas: 48150 (↑ +150 | +0.31%))
```

### 2.3 检查 Snapshot 差异

使用 `--check` 标志，如果 gas 消耗增加超过阈值则失败：

```bash
forge snapshot --check
```


### 2.4 精细化的 Snapshot 测试

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

contract GasOptimizationTest is Test {
    // 测试优化前的版本
    function testUnoptimized() public {
        // 未优化的实现
        uint256 sum;
        for (uint256 i = 0; i < 100; i++) {
            sum += i;
        }
    }

    // 测试优化后的版本
    function testOptimized() public {
        // 优化后的实现
        uint256 sum;
        for (uint256 i; i < 100;) {
            sum += i;
            unchecked { ++i; }
        }
    }
}
```

运行并对比：
```bash
forge snapshot --match-test "test(Un)?[Oo]ptimized"
```

### 2.5 Snapshot 的最佳实践

✅ **提交到版本控制** - 将 `.gas-snapshot` 文件提交到 Git
✅ **在 PR 中审查** - 关注 gas 消耗的变化
✅ **设置告警阈值** - 显著的 gas 增加应该引起注意
✅ **为关键函数编写专门测试** - 确保核心功能的 gas 效率

## 3. 使用 gasleft() 在合约内部测量

`gasleft()` 是 Solidity 内置函数，返回当前剩余的 gas 数量，可以用来精确测量特定代码段的 gas 消耗。

### 3.1 基本用法

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract GasMeter {
    event GasConsumed(string operation, uint256 gasUsed);

    function measureGas() public {
        uint256 gasBefore = gasleft();

        // 要测量的操作
        uint256 result = expensiveOperation();

        uint256 gasAfter = gasleft();
        uint256 gasUsed = gasBefore - gasAfter;

        emit GasConsumed("expensiveOperation", gasUsed);
    }

    function expensiveOperation() internal pure returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < 100; i++) {
            sum += i * i;
        }
        return sum;
    }
}
```

### 3.2 创建可复用的 Gas 测量库

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

library GasProfiler {
    struct Checkpoint {
        uint256 gasLeft;
        string label;
    }

    /// @notice 开始 gas 测量
    function start() internal view returns (uint256) {
        return gasleft();
    }

    /// @notice 结束 gas 测量并返回消耗的 gas
    function end(uint256 startGas) internal view returns (uint256) {
        return startGas - gasleft();
    }

    /// @notice 测量代码块的 gas 消耗
    function measure(
        string memory label
    ) internal view returns (uint256 gasUsed) {
        uint256 gasBefore = gasleft();
        // 注意：这个函数返回 gasBefore，需要在外部计算 gasUsed
        return gasBefore;
    }
}

contract UsingGasProfiler {
    using GasProfiler for *;

    event GasReport(string label, uint256 gasUsed);

    function complexOperation() public {
        // 测量存储写入
        uint256 g1 = GasProfiler.start();
        storageWrite();
        emit GasReport("Storage Write", GasProfiler.end(g1));

        // 测量内存操作
        uint256 g2 = GasProfiler.start();
        memoryOperation();
        emit GasReport("Memory Operation", GasProfiler.end(g2));

        // 测量循环
        uint256 g3 = GasProfiler.start();
        loopOperation();
        emit GasReport("Loop Operation", GasProfiler.end(g3));
    }

    uint256 public data;

    function storageWrite() internal {
        data = 12345;
    }

    function memoryOperation() internal pure {
        uint256[] memory arr = new uint256[](100);
        for (uint256 i = 0; i < 100; i++) {
            arr[i] = i;
        }
    }

    function loopOperation() internal pure {
        uint256 sum;
        for (uint256 i = 0; i < 100; i++) {
            sum += i;
        }
    }
}
```

### 3.3 在测试中使用 gasleft()

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

contract GasComparisonTest is Test {
    function testCompareImplementations() public {
        // 测量实现 A
        uint256 gasBeforeA = gasleft();
        uint256 resultA = implementationA();
        uint256 gasUsedA = gasBeforeA - gasleft();

        // 测量实现 B
        uint256 gasBeforeB = gasleft();
        uint256 resultB = implementationB();
        uint256 gasUsedB = gasBeforeB - gasleft();

        // 验证结果相同
        assertEq(resultA, resultB, "Results should be equal");

        // 输出 gas 对比
        console.log("Implementation A gas:", gasUsedA);
        console.log("Implementation B gas:", gasUsedB);
        console.log("Gas saved:", gasUsedA > gasUsedB ? gasUsedA - gasUsedB : 0);

        // 断言优化版本更省 gas
        assertLt(gasUsedB, gasUsedA, "Implementation B should use less gas");
    }

    function implementationA() internal pure returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < 100; i++) {
            sum = sum + i;
        }
        return sum;
    }

    function implementationB() internal pure returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < 100;) {
            sum = sum + i;
            unchecked { ++i; }
        }
        return sum;
    }
}
```

### 3.4 注意事项

⚠️ **测量开销** - `gasleft()` 本身消耗约 2 gas，在精确测量时需要考虑

⚠️ **编译器优化** - 某些情况下编译器可能会优化掉未使用的代码

⚠️ **外部调用影响** - 跨合约调用时，gas 的计算可能更复杂

⚠️ **Context 依赖** - 测量结果可能受到调用上下文的影响（如存储槽的冷/热状态）

### 3.5 高级测量技巧

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract AdvancedGasMeasurement {
    // 测量多个操作并返回详细报告
    struct GasReport {
        uint256 operation1Gas;
        uint256 operation2Gas;
        uint256 operation3Gas;
        uint256 totalGas;
    }

    function detailedGasReport() public view returns (GasReport memory) {
        uint256 totalStart = gasleft();

        uint256 g1 = gasleft();
        // 操作 1
        uint256 temp1 = heavyComputation(50);
        uint256 op1Gas = g1 - gasleft();

        uint256 g2 = gasleft();
        // 操作 2
        uint256 temp2 = heavyComputation(100);
        uint256 op2Gas = g2 - gasleft();

        uint256 g3 = gasleft();
        // 操作 3
        uint256 temp3 = heavyComputation(150);
        uint256 op3Gas = g3 - gasleft();

        uint256 totalGas = totalStart - gasleft();

        return GasReport({
            operation1Gas: op1Gas,
            operation2Gas: op2Gas,
            operation3Gas: op3Gas,
            totalGas: totalGas
        });
    }

    function heavyComputation(uint256 n) internal pure returns (uint256) {
        uint256 result;
        for (uint256 i = 0; i < n; i++) {
            result += i ** 2;
        }
        return result;
    }
}
```

## 4. 示例：完整的 Gas 评估流程

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";

// 待优化的合约
contract TokenV1 {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] = balances[msg.sender] - amount;
        balances[to] = balances[to] + amount;
    }
}

// 优化后的合约
contract TokenV2 {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        unchecked {
            balances[msg.sender] -= amount;
            balances[to] += amount;
        }
    }
}

// 完整的测试和评估
contract GasComparisonTest is Test {
    TokenV1 tokenV1;
    TokenV2 tokenV2;

    address user1 = address(0x1);
    address user2 = address(0x2);

    function setUp() public {
        tokenV1 = new TokenV1();
        tokenV2 = new TokenV2();

        // 初始化余额
        vm.store(
            address(tokenV1),
            keccak256(abi.encode(user1, 0)),
            bytes32(uint256(1000 ether))
        );
        vm.store(
            address(tokenV2),
            keccak256(abi.encode(user1, 0)),
            bytes32(uint256(1000 ether))
        );
    }

    // 方法 1: 使用 forge test --gas-report
    function testV1Transfer() public {
        vm.prank(user1);
        tokenV1.transfer(user2, 100 ether);
    }

    function testV2Transfer() public {
        vm.prank(user1);
        tokenV2.transfer(user2, 100 ether);
    }

    // 方法 2: 使用 gasleft() 精确对比
    function testGasComparison() public {
        // 测量 V1
        vm.prank(user1);
        uint256 gasBeforeV1 = gasleft();
        tokenV1.transfer(user2, 100 ether);
        uint256 gasUsedV1 = gasBeforeV1 - gasleft();

        // 重置状态
        setUp();

        // 测量 V2
        vm.prank(user1);
        uint256 gasBeforeV2 = gasleft();
        tokenV2.transfer(user2, 100 ether);
        uint256 gasUsedV2 = gasBeforeV2 - gasleft();

        // 输出对比
        console.log("V1 Gas Used:", gasUsedV1);
        console.log("V2 Gas Used:", gasUsedV2);
        console.log("Gas Saved:", gasUsedV1 - gasUsedV2);
        console.log(
            "Improvement:",
            (gasUsedV1 - gasUsedV2) * 100 / gasUsedV1,
            "%"
        );

        // 断言优化有效
        assertLt(gasUsedV2, gasUsedV1, "V2 should be more gas efficient");
    }
}
```

### 运行完整评估

```bash
# 1. 生成 gas report
forge test --gas-report --match-contract GasComparisonTest

# 2. 创建 snapshot
forge snapshot --match-contract GasComparisonTest

# 3. 查看详细输出
forge test --match-test testGasComparison -vv
```


## 总结

Gas 评估是优化工作的基础，本章介绍了三种主要的评估方法：

| 方法 | 优点 | 适用场景 |
|------|------|----------|
| **Forge Gas Report** | 自动化、全面 | 整体性能评估 |
| **Gas Snapshots** | 追踪变化、CI 集成 | 防止性能退化 |
| **gasleft()** | 精确、灵活 | 细粒度测量 |


推荐做法是：

1. **建立基线** - 在优化前先测量当前性能
2. **使用多种工具** - 结合 gas report、snapshot 和 gasleft()
3. **版本控制** - 提交 `.gas-snapshot` 文件
4. **关注关键路径** - 重点优化高频调用的函数
5. **验证优化效果** - 优化后必须再次测量

掌握这些工具后，你就可以准确识别优化点，验证优化效果，并在开发过程中持续监控 gas 效率。

下一章我们将开始学习具体的 gas 优化技巧。


