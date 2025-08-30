Foundry 是一个现代化的 Solidity 开发工具链，专注于智能合约的 **开发、测试和部署**。以下是 Foundry 项目的完整开发和测试流程：

---

## **1. 项目初始化**
### **（1）创建新项目**
```bash
forge init my_project  # 初始化新项目
cd my_project
```
项目结构：
```
my_project/
├── src/       # Solidity 合约代码（*.sol）
├── test/      # 测试脚本（Solidity 或 JavaScript）
├── script/    # 部署脚本
├── foundry.toml  # 配置文件
└── lib/       # 依赖库（如 OpenZeppelin）
```

### **（2）安装依赖（如 OpenZeppelin）**
```bash
forge install openzeppelin/openzeppelin-contracts
```
依赖会安装在 `lib/` 目录，并在 `remappings.txt` 中生成路径映射。

---

## **2. 开发智能合约**
### **（1）编写合约**
在 `src/` 下创建 Solidity 文件（例如 `Counter.sol`）：
```solidity
// src/Counter.sol
pragma solidity ^0.8.13;

contract Counter {
    uint256 public count;

    function increment() public {
        count++;
    }
}
```

### **（2）编译合约**
```bash
forge build
```
- 编译后的 ABI 和字节码会生成在 `out/` 目录。

---

## **3. 测试智能合约**
Foundry 的测试基于 **Solidity 测试脚本**（比 JavaScript 测试更快、更贴近链上环境）。

### **（1）编写测试**
在 `test/` 下创建测试文件（例如 `Counter.t.sol`）：
```solidity
// test/Counter.t.sol
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;

    function setUp() public {
        counter = new Counter();
    }

    function testIncrement() public {
        assertEq(counter.count(), 0);
        counter.increment();
        assertEq(counter.count(), 1);
    }

    function testFailDecrement() public {
        counter.increment();
        assertEq(counter.count(), 0);  // 这会失败
    }
}
```

### **（2）运行测试**
```bash
forge test
```
**输出示例**：
```
[PASS] testIncrement() (gas: 28334)
[FAIL] testFailDecrement() (gas: 21517)
```

### **（3）高级测试功能**
| 功能         | 命令/代码                          | 说明               |
| ------------ | ---------------------------------- | ------------------ |
| **Gas 报告** | `forge test --gas-report`          | 查看函数消耗的 Gas |
| **模糊测试** | `forge test --match-test testFuzz` | 随机输入测试       |
| **主网分叉** | `forge test --fork-url <RPC_URL>`  | 模拟主网环境       |
| **事件检查** | `vm.expectEmit()`                  | 验证事件是否触发   |

---

## **4. 部署与交互**
### **（1）启动本地测试节点（Anvil）**
```bash
anvil
```
- 默认运行在 `http://127.0.0.1:8545`，提供 10 个测试账户。

### **（2）部署合约**
#### **方式 1：直接部署**
```bash
forge create --rpc-url http://127.0.0.1:8545 \
    --private-key <测试账户私钥> \
    src/Counter.sol:Counter
```

#### **方式 2：使用部署脚本**
1. 在 `script/` 下创建部署脚本（例如 `DeployCounter.s.sol`）：
   ```solidity
   // script/DeployCounter.s.sol
   pragma solidity ^0.8.13;
   
   import "forge-std/Script.sol";
   import "../src/Counter.sol";
   
   contract DeployCounter is Script {
       function run() public {
           vm.startBroadcast();
           new Counter();
           vm.stopBroadcast();
       }
   }
   ```
2. 运行脚本：
   ```bash
   forge script script/DeployCounter.s.sol \
       --rpc-url http://127.0.0.1:8545 \
       --private-key <私钥> \
       --broadcast
   ```

### **（3）与合约交互（Cast）**
```bash
# 读取合约状态
cast call <合约地址> "count()" --rpc-url http://127.0.0.1:8545

# 调用合约方法
cast send <合约地址> "increment()" \
    --rpc-url http://127.0.0.1:8545 \
    --private-key <私钥>
```

---

## **5. 高级功能**
### **（1）主网分叉测试**
```bash
forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/<API_KEY>
```
- 在本地模拟主网环境，测试合约与真实协议的交互。

### **（2）Gas 优化**
- 使用 `forge snapshot` 生成 Gas 快照，对比优化前后差异：
  ```bash
  forge snapshot --gas-report
  ```

### **（3）集成 CI/CD**
在 GitHub Actions 中添加 Foundry 测试：
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: foundry-rs/foundry-toolchain@v1
      - run: forge test
```

---

## **6. 调试技巧**
| 问题             | 解决方案                                                     |
| ---------------- | ------------------------------------------------------------ |
| **测试失败**     | 使用 `-vvv` 输出详细日志：`forge test -vvv`                  |
| **交易回滚**     | 在测试中用 `vm.expectRevert()` 捕获预期错误                  |
| **合约状态异常** | 使用 `console.log` 打印调试信息（需导入 `forge-std/console.sol`） |

---

## **总结：Foundry 开发测试流程**
1. **初始化项目**：`forge init`  
2. **编写合约**：`src/*.sol`  
3. **编写测试**：`test/*.t.sol` → `forge test`  
4. **部署合约**：`anvil` + `forge create` 或 `forge script`  
5. **交互调试**：`cast call/send`  

Foundry 的优势在于：
- **极快的 Solidity 原生测试**（无需 JavaScript）。
- **强大的作弊码（Cheatcodes）** 模拟复杂场景。
- **无缝的主网分叉测试**。

如果有具体问题（如模糊测试、Gas 优化），可以进一步探讨！ 🚀