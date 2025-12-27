# 过时的技巧

> **更新说明**：本章节列出了在新版本 Solidity、EVM 升级或生态系统变化后不再有效或重要性大幅降低的优化技巧。

## 1. external 比 public 更便宜

**过时原因：** 编译器优化改进

如果函数无法在合约内部调用，为了清晰起见，仍然应该优先选择 external 修饰符，但它对于节省 gas 没有影响。

**时间线：** 约 Solidity 0.8.x 之后效果不明显

---

## 2. != 0 比 > 0 更便宜

**过时原因：** 编译器优化改进

大约在 Solidity 0.8.12 左右，这个说法不再成立。如果你被迫使用旧版本，你仍然可以对其进行基准测试。

**时间线：** Solidity 0.8.12+

---

## 3. 跨交易使用 SELFDESTRUCT 删除合约和清理存储

> **⚠️ 重要：** 这是 Cancun 升级影响最大的变更之一！

**过时原因：** EIP-6780 限制了 SELFDESTRUCT 的行为

### 为什么过时

在 Cancun 升级（2024年3月）之后，`SELFDESTRUCT` 仅在以下情况保留完整功能：
- **同一交易内创建并销毁的合约**

对于已存在的合约调用 `SELFDESTRUCT`：
- ❌ **不再**删除合约代码
- ❌ **不再**清除存储
- ✅ 仅转移 ETH 余额
- ❌ **不再**退还 gas

### 受影响的模式

```solidity
// ❌ 不再有效：跨交易销毁
contract OldPattern {
    function cleanup() external {
        selfdestruct(payable(msg.sender));
        // 代码和存储不会被删除！
    }
}

// ❌ 不再有效：CREATE2 + SELFDESTRUCT 重新部署
contract RedeployPattern {
    function redeploy(bytes32 salt) external {
        address deployed = address(new Contract{salt: salt}());
        Contract(deployed).destroy();
        // 无法在同一地址重新部署
    }
}
```

### 替代方案

- 使用代理模式（UUPS / 透明代理）
- 使用最小克隆（EIP-1167）
- 使用状态标志停用合约

**参考：**
- [Chapter 2 第4节](./chapter_2.md#4-如果合约只用于一次性使用则在构造函数中使用-selfdestruct)

**时间线：** Cancun 升级（2024年3月）后

---

## 4. L2 上极致优化 Calldata 中的零字节

> **⚠️ 重要：** EIP-4844 对 L2 的 calldata 优化策略产生革命性影响！

**过时原因：** EIP-4844 引入的 Blob 交易改变了 L2 的数据发布机制

### 为什么重要性降低

在 EIP-4844 之前，L2 需要将所有交易数据作为 calldata 发布到 L1：
- 零字节：4 gas
- 非零字节：16 gas
- 差距：**4倍**

在 EIP-4844 之后，L2 主要使用 Blob 交易发布数据：
- Blob 数据：**不区分**零字节和非零字节
- 总成本降低：**80-90%**

### 受影响的优化

```solidity
// ⚠️ 在 L2 上重要性大幅降低
contract CalldataOptimization {
    // 使用虚荣地址（前导零）
    address public constant TOKEN = 0x0000000000000000000000000000000000000001;

    // 压缩地址到 uint160
    function compress(address addr) external pure returns (uint160) {
        return uint160(addr);
    }

    // 使用短函数名
    function t() external {} // 而不是 transfer()
}
```

### 新的优化优先级（L2）

**高优先级：**
- ✅ 存储优化（变量打包、临时存储）
- ✅ 计算优化（循环、unchecked）
- ✅ 合约架构优化

**低优先级：**
- 🔻 Calldata 零字节优化
- 🔻 虚荣地址
- 🔻 函数名压缩

### 仍需优化的场景

- L1 主网合约（完全适用）
- L1↔L2 跨链消息（仍使用 calldata）
- 极大数据量的操作

**参考：** [Chapter 5](./chapter_5.md) 完整分析

**时间线：** Cancun 升级（2024年3月）后

---

## 5. 某些编译器特定的优化技巧

**过时原因：** 编译器自动优化改进

### 部分被编译器自动优化的技巧

随着 Solidity 编译器的不断改进（0.8.20 → 0.8.28），某些手动优化已被编译器自动应用：

#### 5.1 PUSH0 优化（0.8.20+）

```solidity
// 编译器现在自动使用 PUSH0
function getZero() public pure returns (uint256) {
    return 0;  // 自动使用 PUSH0 操作码
}
```

**影响：** 无需手动优化零值初始化

#### 5.2 部分循环优化（0.8.22+）

```solidity
// 编译器可能自动识别安全的循环
for (uint256 i = 0; i < arr.length; i++) {
    // 某些情况下编译器会自动优化
}
```

**影响：** 但仍建议显式使用 `unchecked { ++i }`

#### 5.3 简单位移优化

```solidity
// 编译器可能自动优化为位移
uint256 x = y * 2;   // 可能优化为 y << 1
uint256 z = y / 4;   // 可能优化为 y >> 2
```

**影响：** 对于 2 的幂次乘除法，编译器可能自动优化

### 建议

- ✅ 仍然测试优化效果
- ✅ 关注编译器版本差异
- ✅ 使用最新的稳定版本（0.8.24+）
- ⚠️ 不要盲目信任编译器，始终基准测试

**时间线：** Solidity 0.8.20+ 逐步改进

---

## 6. 过度依赖 SSTORE2 用于临时数据

**过时原因：** 临时存储（Transient Storage）提供了更好的方案

### 为什么重要性降低

**SSTORE2（传统方案）：**
- 将数据作为字节码存储
- 成本：~200 gas/字节（写入）
- 持久性：永久存储
- 适用：需要长期保存的大量数据

**临时存储（EIP-1153）：**
- 仅在交易期间存在
- 成本：100 gas（TSTORE/TLOAD）
- 持久性：交易结束后清零
- 适用：单交易内的临时状态

### 对比示例

```solidity
// ❌ 使用 SSTORE2 存储临时数据（不再推荐）
contract OldApproach {
    mapping(uint256 => address) tempDataPointers;

    function processBatch(bytes[] calldata items) external {
        // 将临时数据存储到 SSTORE2
        address pointer = SSTORE2.write(abi.encode(items));
        tempDataPointers[block.number] = pointer;

        // 处理...

        // 需要手动清理
    }
}

// ✅ 使用临时存储（推荐）
contract NewApproach {
    mapping(uint256 => bytes) private transient tempData;

    function processBatch(bytes[] calldata items) external {
        for (uint256 i = 0; i < items.length; i++) {
            tempData[i] = items[i];  // TSTORE: 100 gas
        }

        // 处理...

        // 交易结束后自动清零，无需手动清理
    }
}
```

### 何时仍使用 SSTORE2

- ✅ 需要永久保存的大量数据（> 24 KB）
- ✅ 需要跨交易访问的数据
- ✅ 链上 NFT 元数据存储

### 何时使用临时存储

- ✅ 单交易内的临时状态
- ✅ 重入锁
- ✅ 闪电贷状态管理
- ✅ 批量操作的中间数据

**参考：**
- [Chapter 1 第6.5节](./chapter_1.md#65-使用临时存储transient-storage节省高达-99-的-gas)
- [Chapter 10 第1节](./chapter_10.md#1-临时存储transient-storage---eip-1153)

**时间线：** Cancun 升级（2024年3月）后

---

## 总结

### 过时技巧分类

**编译器改进导致：**
1. external vs public
2. != 0 vs > 0
3. 某些循环和位移优化

**EVM 升级导致：**
4. 跨交易 SELFDESTRUCT（Cancun - EIP-6780）
5. L2 calldata 零字节优化（Cancun - EIP-4844）
6. SSTORE2 用于临时数据（Cancun - EIP-1153）

### 建议

1. **保持更新**：使用 Solidity 0.8.24+ 和 Cancun EVM
2. **重新评估**：定期审查优化策略是否仍然有效
3. **基准测试**：始终测试验证优化效果
4. **关注升级**：跟踪以太坊升级和编译器更新
5. **使用现代工具**：Solady、临时存储、Blob 交易等

