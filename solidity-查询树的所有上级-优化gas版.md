在 Solidity 中，为了优化 `gas` 费用，通常有以下几种策略：

1. **减少存储操作**：存储操作是最昂贵的，因此应尽可能减少写入 `storage` 的次数。
2. **避免重复计算**：尽量避免重复的计算或操作，尤其是在循环中。
3. **使用内联函数和修饰符**：可以将常用操作封装为修饰符或内联函数，以便于代码复用。
4. **控制数组分配**：在内存中创建数组时，分配的空间大小需要预先估算，以避免动态分配带来的开销。

基于上述策略，我们可以在树形结构的场景下，尝试以下优化方法：

1. **减少循环中的存储读取**：将重复读取 `storage` 中的数据放到局部变量中。
2. **合并逻辑**：将某些计算和条件判断合并，减少不必要的检查。
3. **在循环中优化存储**：尽量减少 `mapping` 的访问次数。
4. **优化数组的创建**：提前计算所需的数组大小。

### 优化后的代码
以下是针对上述策略的优化版本：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract OptimizedAncestorTree {
    // 映射：节点 => 父节点
    mapping(uint256 => uint256) public parent;

    // 设置某个节点的父节点
    function setParent(uint256 node, uint256 parentNode) external {
        require(node != parentNode, "Node cannot be its own parent");
        parent[node] = parentNode;
    }

    // 查询某个节点的所有上级节点（优化版）
    function getAncestors(uint256 node) external view returns (uint256[] memory) {
        // 通过一次遍历先计算上级节点数量（减少动态数组操作）
        uint256 count;
        uint256 current = node;

        // 计算上级节点数量
        while (parent[current] != 0 && parent[current] != current) {
            count++;
            current = parent[current];
        }

        // 创建固定大小的数组来保存上级节点
        uint256[] memory ancestors = new uint256[](count);
        current = node;

        // 倒序填充数组，避免额外的数组倒序开销
        while (count > 0) {
            current = parent[current];
            ancestors[--count] = current; // 直接填充到数组的倒数位置
        }

        return ancestors;
    }
}
```

### 关键优化点说明
1. **避免重复存储读取**：
   - 在 `getAncestors` 中，我们通过 `current` 局部变量暂存 `parent` 中的值，避免重复读取 `storage`（每次读取 `storage` 都需要较高的 `gas`）。

2. **预计算节点数量，避免动态数组分配**：
   - 先计算上级节点的数量 `count`，然后一次性创建指定大小的 `ancestors` 数组，而不是动态调整数组大小。
   - 这样可以大幅减少动态分配和 `gas` 费用。

3. **倒序填充，避免额外操作**：
   - 在找到上级节点后，直接从数组的尾部开始填充（倒序填充），避免最后需要再反转数组的开销。

### `Gas` 优化效果
- 通过减少循环中的重复存储读取、控制数组的分配和简化逻辑，`gas` 开销相比原来版本会显著降低。
- 此外，如果对合约逻辑有更多了解，可以考虑进一步优化，比如使用哈希表或其他更高效的数据结构来代替线性查找。

### 实验优化方法
如果希望在实际环境中测试 `gas` 优化效果，可以使用以下方法：

1. **使用 Hardhat 或 Remix 编译、部署合约**。
2. **执行测试用例，使用 `gas` 分析工具（如 Remix 的 `gas` 分析器）**。
3. **对比优化前后的 `gas` 费用**。

如果有更复杂的需求（例如需要频繁变更父节点或更复杂的树形结构），可以根据实际使用场景进一步优化合约结构。
