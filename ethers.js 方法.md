在Hardhat环境中，`import { ethers } from "hardhat"` 导入的是一个增强版的 **ethers.js 库**，它集成了Hardhat特有的功能（如简化合约部署和测试）。以下是主要方法和属性的详细说明：

---

### 核心功能模块
1. **Provider 提供者**  
   `ethers.provider`：连接以太坊网络的接口（默认连接Hardhat本地节点）  
   ```javascript
   const blockNumber = await ethers.provider.getBlockNumber();
   ```

2. **Signers 签名者**  
   `ethers.getSigners()`：获取测试账户列表（含私钥的以太坊账户）  
   ```javascript
   const [owner, addr1] = await ethers.getSigners();
   ```

3. **合约工厂 ContractFactory**  
   `ethers.getContractFactory(name, [signer])`：编译合约并生成部署工厂  
   ```javascript
   const MyContract = await ethers.getContractFactory("MyToken");
   const contract = await MyContract.deploy(); // 部署合约
   ```

4. **已部署合约 Contract**  
   `ethers.getContractAt(abi, address, [signer])`：连接已部署合约  
   ```javascript
   const contract = await ethers.getContractAt("IERC20", "0x123...abc");
   ```

---

### 关键实用工具（`ethers.utils`）
| 方法                | 用途             | 示例                                                        |
| ------------------- | ---------------- | ----------------------------------------------------------- |
| `parseEther`        | ETH → Wei        | `ethers.utils.parseEther("1.5")` → `1500000000000000000`    |
| `formatEther`       | Wei → ETH        | `ethers.utils.formatEther(bigNumber)` → `"1.5"`             |
| `parseUnits`        | 自定义单位转换   | `parseUnits("100", 6)` → 100 USDC（6位小数）                |
| `formatUnits`       | 反解析单位       | `formatUnits(bigNumber, 9)` → `"0.000000001"`               |
| `keccak256`         | 计算哈希         | `keccak256(ethers.utils.toUtf8Bytes("Hello"))`              |
| `solidityKeccak256` | Solidity风格哈希 | `solidityKeccak256(["address", "uint256"], [addr, amount])` |
| `encodePacked`      | 紧密编码         | `encodePacked(["uint8", "string"], [1, "test"])`            |
| `verifyMessage`     | 验证签名         | `verifyMessage("Hello", signature)` → 签名地址              |

---

### 加密与钱包（`ethers.Wallet`）
```javascript
const wallet = new ethers.Wallet(privateKey, ethers.provider); // 创建钱包
wallet.sendTransaction({ to: address, value: ethers.utils.parseEther("0.1") });
```

---

### 数据类型处理
- **BigNumber**：大整数处理  
  ```javascript
  const bigNum = ethers.BigNumber.from("1000000000000000000");
  bigNum.add(1); // 安全数学运算
  ```
- **Interface**：ABI编解码  
  ```javascript
  const iface = new ethers.utils.Interface(["function transfer(address,uint256)"]);
  iface.encodeFunctionData("transfer", ["0x...", 100]);
  ```

---

### Hardhat特有扩展
- **测试网络集成**：自动连接Hardhat本地节点（无需手动配置Provider）
- **零配置账户**：`getSigners()`直接获取10个预充值测试账户
- **简化部署流程**：`getContractFactory`自动处理编译和链接

---

### 完整使用示例
```javascript
async function main() {
  // 1. 获取账户
  const [deployer] = await ethers.getSigners();
  
  // 2. 部署合约
  const Contract = await ethers.getContractFactory("MyContract");
  const contract = await Contract.deploy();
  
  // 3. 调用合约
  await contract.doSomething();
  
  // 4. 发送ETH
  await deployer.sendTransaction({
    to: "0x742d35Cc6634C0532925a3b844Bc454e4438f44e",
    value: ethers.utils.parseEther("0.5")
  });
}

main().catch(console.error);
```

> 💡 提示：Hardhat的`ethers`对象是标准ethers.js的扩展，完整API参考[ethers.js文档](https://docs.ethers.io/v5/)。