



### Hardhat 学习资料完整指南

Hardhat 是 Ethereum 开发环境，用于编写、测试、调试和部署智能合约。它灵活、可扩展，支持 Solidity 合约开发，集成多种插件如 OpenZeppelin 用于可升级合约。以下是完整指南，基于官方文档和最新教程（2025 年更新）。我将重点解释你不明白的部分：测试、可升级合约的几种写法、部署方法。内容结构化，便于学习。

#### 1. 安装和设置

- **前提**：Node.js v22 或更高版本，npm/yarn/pnpm。

- **初始化项目**：

  ```
  mkdir my-hardhat-project
  cd my-hardhat-project
  npx hardhat --init
  ```

  选择默认配置，会生成 `hardhat.config.ts`、`contracts/`、`test/`、`scripts/` 等文件夹。

- **配置**：在 `hardhat.config.ts` 中设置 Solidity 版本、网络（如本地 Hardhat 网络、Sepolia 测试网）、插件。
  示例：

  ```typescript
  import { HardhatUserConfig } from "hardhat/config";
  import "@nomicfoundation/hardhat-toolbox";
  
  const config: HardhatUserConfig = {
    solidity: "0.8.28",
    networks: {
      hardhat: {},
      sepolia: {
        url: "YOUR_RPC_URL",
        accounts: ["YOUR_PRIVATE_KEY"]
      }
    }
  };
  export default config;
  ```

- **编译合约**：`npx hardhat compile`。

#### 2. 编写智能合约

在 `contracts/` 下创建 Solidity 文件。例如，一个简单计数器合约 `Counter.sol`：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract Counter {
  uint public x;
  event Increment(uint by);

  function inc() public {
    x++;
    emit Increment(1);
  }

  function incBy(uint by) public {
    require(by > 0, "incBy: increment should be positive");
    x += by;
    emit Increment(by);
  }
}
```

编译后，在 `artifacts/` 生成 ABI 和字节码。

#### 3. 测试部分（重点）

Hardhat 使用 Mocha（测试框架）和 Chai（断言库）进行测试，支持 Solidity 和 TypeScript 测试。测试文件放在 `test/` 下，使用 Ethers.js 交互合约。

- **推荐插件**：`@nomicfoundation/hardhat-toolbox` 包含 Chai 匹配器和网络助手。

- **测试类型**：

  - **Solidity 测试**：用于单元测试，文件以 `.t.sol` 结尾。使用 Foundry 的 `Test` 库。
    示例 `contracts/Counter.t.sol`：

    ```solidity
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.8.28;
    
    import { Counter } from "./Counter.sol";
    import { Test } from "forge-std/Test.sol";
    
    contract CounterTest is Test {
      Counter counter;
    
      function setUp() public {
        counter = new Counter();
      }
    
      function test_InitialValue() public view {
        assertEq(counter.x(), 0, "Initial value should be 0");
      }
    
      function testFuzz_Inc(uint8 num) public {
        for (uint8 i = 0; i < num; i++) {
          counter.inc();
        }
        assertEq(counter.x(), num, "Value after inc num times should be num");
      }
    
      function test_IncByZero() public {
        vm.expectRevert();
        counter.incBy(0);
      }
    }
    ```

    ### 运行 Foundry 测试

    ```bash
    forge test
    ```

    运行：`npx hardhat test solidity`。

  - **TypeScript 测试**：用于集成测试，使用 Mocha/Chai。
    示例 `test/Counter.ts`：

    ```typescript
    import { expect } from "chai";
    import { ethers } from "hardhat";
    import { loadFixture } from "@nomicfoundation/hardhat-network-helpers";
    
    describe("Counter", function () {
      async function deployCounterFixture() {
        const Counter = await ethers.getContractFactory("Counter");
        const counter = await Counter.deploy();
        return { counter };
      }
    
      it("Should set initial value to 0", async function () {
        const { counter } = await loadFixture(deployCounterFixture);
        expect(await counter.x()).to.equal(0);
      });
    
      it("Should increment by 1", async function () {
        const { counter } = await loadFixture(deployCounterFixture);
        await counter.inc();
        expect(await counter.x()).to.equal(1);
      });
    
      it("Should revert on incBy(0)", async function () {
        const { counter } = await loadFixture(deployCounterFixture);
        await expect(counter.incBy(0)).to.be.revertedWith("incBy: increment should be positive");
      });
    
      it("Should emit Increment event", async function () {
        const { counter } = await loadFixture(deployCounterFixture);
        await expect(counter.incBy(5)).to.emit(counter, "Increment").withArgs(5);
      });
    });
    ```

    - `describe`：分组测试套件。
    - `it`：单个测试用例。
    - `beforeEach` 或 `loadFixture`：设置 fixture，避免重复部署，提高速度。
    - Chai 断言：`expect(value).to.equal(expected)`、`expect(tx).to.be.revertedWith("error")`、`expect(tx).to.emit(contract, "Event").withArgs(args)`。
    - 运行：`npx hardhat test`。指定单个测试：`npx hardhat test --grep "Should increment"`。

- **高级测试**：

  - 模糊测试（fuzzing）：如上例 `testFuzz_Inc`，随机输入测试。
  - 事件和 gas 测试：使用 Chai 匹配器检查事件、gas 消耗。
  - 覆盖率：`npx hardhat coverage`（使用 solidity-coverage 插件）。
  - Gas 报告：设置 `REPORT_GAS=true` 运行测试。

- **最佳实践**：测试覆盖所有边缘案例（如零输入、权限检查）。使用 Hardhat 网络模拟区块链。[[2]](https://hardhat.org/hardhat-runner/docs/guides/test-contracts)[[35]](https://medium.com/%40davidcisar/how-to-test-smart-contracts-8b1321248387)[[37]](https://dev.to/carlomigueldy/unit-testing-a-solidity-smart-contract-using-chai-mocha-with-typescript-3gcj)[[38]](https://hardhat.org/hardhat-runner/docs/guides/test-contracts)

#### 4. 可升级合约的几种写法（重点）

可升级合约允许更新逻辑而不改变地址，使用代理模式。Hardhat 支持 OpenZeppelin 的 `@openzeppelin/hardhat-upgrades` 插件。常见模式：Transparent Proxy、UUPS、Beacon Proxy。

- **安装插件**：`npm install @openzeppelin/contracts-upgradeable @openzeppelin/hardhat-upgrades`。
- **配置**：在 `hardhat.config.ts` 添加 `require("@openzeppelin/hardhat-upgrades");`。

- **模式比较**（表格）：

| 模式              | 描述                                                         | 优点                       | 缺点                       | 适用场景                 |
| ----------------- | ------------------------------------------------------------ | -------------------------- | -------------------------- | ------------------------ |
| Transparent Proxy | 代理区分管理员调用（升级）和用户调用（逻辑）。使用 `TransparentUpgradeableProxy`。 | 简单，避免选择器冲突。     | 代理合约较大，gas 较高。   | 单一代理，管理员升级。   |
| UUPS              | 升级逻辑在实现合约中（`upgradeTo` 函数），代理存储实现地址。使用 `UUPSUpgradeable`。 | gas 高效，可移除升级能力。 | 需保护升级函数，避免锁定。 | 多个实例，追求效率。     |
| Beacon Proxy      | 多个代理指向一个 Beacon 合约，升级 Beacon 原子更新所有代理。使用 `UpgradeableBeacon` 和 `BeaconProxy`。 | 批量升级，gas 低。         | Beacon 需单独管理。        | 多个相同代理（如钱包）。 |

- **实现示例**（使用 UUPS）：

  1. 初始合约 `Box.sol`：

     ```solidity
     // SPDX-License-Identifier: MIT
     pragma solidity ^0.8.24;
     import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
     import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
     
     contract Box is UUPSUpgradeable, OwnableUpgradeable {
       uint256 private _value;
     
       function initialize() public initializer {
         __Ownable_init(msg.sender);
         __UUPSUpgradeable_init();
       }
     
       function store(uint256 value) public {
         _value = value;
       }
     
       function retrieve() public view returns (uint256) {
         return _value;
       }
     
       function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
     }
     ```

     注意：使用 `initializer` 代替构造函数。

  2. 升级版 `BoxV2.sol`（添加新函数）：

     ```solidity
     // SPDX-License-Identifier: MIT
     pragma solidity ^0.8.24;
     import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
     import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
     
     contract BoxV2 is UUPSUpgradeable, OwnableUpgradeable {
       uint256 private _value;
     
       function initialize() public initializer {
         __Ownable_init(msg.sender);
         __UUPSUpgradeable_init();
       }
     
       function store(uint256 value) public {
         _value = value;
       }
     
       function retrieve() public view returns (uint256) {
         return _value;
       }
     
       function increment() public {
         _value += 1;
       }
     
       function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
     }
     ```

  3. 部署脚本 `scripts/deployProxy.js`：

     ```javascript
     const { ethers, upgrades } = require("hardhat");
     
     async function main() {
       const Box = await ethers.getContractFactory("Box");
       console.log("Deploying Box...");
       const box = await upgrades.deployProxy(Box, { initializer: "initialize", kind: "uups" });
       await box.waitForDeployment();
       console.log("Box deployed to:", await box.getAddress());
     }
     
     main();
     ```

  4. 升级脚本 `scripts/upgradeProxy.js`：

     ```javascript
     const { ethers, upgrades } = require("hardhat");
     
     async function main() {
       const BoxV2 = await ethers.getContractFactory("BoxV2");
       const proxyAddress = "YOUR_PROXY_ADDRESS";
       console.log("Upgrading Box...");
       await upgrades.upgradeProxy(proxyAddress, BoxV2);
       console.log("Box upgraded");
     }
     
     main();
     ```

     运行：`npx hardhat run scripts/deployProxy.js --network sepolia`。

- **测试升级**：在测试中验证存储兼容性，使用 `upgrades.validateUpgrade`。

- **注意**：升级前验证存储布局，避免数据丢失。UUPS 更高效，但需保护 `_authorizeUpgrade`。[[3]](https://www.quicknode.com/guides/ethereum-development/smart-contracts/how-to-create-and-deploy-an-upgradeable-smart-contract-using-openzeppelin-and-hardhat)[[4]](https://medium.com/%40arian.web3developer/develop-a-uups-contract-with-hardhat-abe1782140eb)[[13]](https://coinsbench.com/building-upgradable-smart-contracts-with-hardhat-a-comprehensive-guide-1ef53b9decf2)[[15]](https://hardhat.org/ignition/docs/guides/upgradeable-proxies)[[16]](https://doc.confluxnetwork.org/docs/espace/tutorials/upgradableContract/uups)[[17]](https://www.quicknode.com/guides/ethereum-development/smart-contracts/how-to-create-and-deploy-an-upgradeable-smart-contract-using-openzeppelin-and-hardhat)[[21]](https://github.com/OpenZeppelin/openzeppelin-upgrades)

#### 5. 部署方法（重点）

- **基本部署**：使用脚本在 `scripts/` 下。
  示例 `scripts/deploy.js`：

  ```javascript
  const { ethers } = require("hardhat");
  
  async function main() {
    const Counter = await ethers.getContractFactory("Counter");
    const counter = await Counter.deploy();
    await counter.waitForDeployment();
    console.log("Counter deployed to:", await counter.getAddress());
  }
  
  main();
  ```

  运行：`npx hardhat run scripts/deploy.js --network sepolia`。

- **使用 hardhat-deploy 插件**（推荐，用于跟踪部署）：

  - 安装：`npm install hardhat-deploy hardhat-deploy-ethers`。

  - 配置：添加 `require("hardhat-deploy");`。

  - 部署脚本在 `deploy/` 下，例如 `deploy/001_deploy_counter.js`：

    ```javascript
    module.exports = async ({ getNamedAccounts, deployments }) => {
      const { deploy } = deployments;
      const { deployer } = await getNamedAccounts();
      await deploy("Counter", {
        from: deployer,
        log: true
      });
    };
    ```

  - 运行：`npx hardhat deploy --network sepolia`。自动跳过已部署合约。

- **验证合约**（Etherscan）：

  - 安装：`npm install @nomiclabs/hardhat-etherscan`。
  - 配置：添加 API Key。
  - 运行：`npx hardhat verify --network sepolia CONTRACT_ADDRESS "ConstructorArgs"`。

- **最佳实践**：

  - 使用 fixture 或条件逻辑在脚本中。
  - 代理部署：结合升级插件。
  - 确定性部署：配置 `deterministicDeployment` 以固定地址。[[6]](https://docs.polkadot.com/tutorials/smart-contracts/launch-your-first-project/test-and-deploy-with-hardhat/)[[8]](https://github.com/pacelliv/hardhat-deploy-tutorial)[[9]](https://www.hackquest.io/articles/deploying-hardhat-tutorial-a-comprehensive-step-by-step-guide-for-web3-developers)[[12]](https://www.opcito.com/blogs/a-step-by-step-guide-for-smart-contract-deployment-using-hardhat)[[25]](https://hardhat.org/ignition/docs/guides/scripts)[[26]](https://github.com/wighawag/hardhat-deploy)[[34]](https://www.quicknode.com/guides/ethereum-development/smart-contracts/how-to-create-and-deploy-a-smart-contract-with-hardhat)

#### 6. 高级主题和资源

- **插件**：`hardhat-gas-reporter`、`hardhat-ignition` 用于高级部署。
- **调试**：`npx hardhat console` 或使用 VS Code 插件。
- **官方文档**：https://hardhat.org/docs。[[45]](https://hardhat.org/docs)
- **教程**：Alchemy、QuickNode、Medium 等提供示例。[[3]](https://www.quicknode.com/guides/ethereum-development/smart-contracts/how-to-create-and-deploy-an-upgradeable-smart-contract-using-openzeppelin-and-hardhat)[[10]](https://www.youtube.com/watch?v=rxK3UXld8xY)[[33]](https://medium.com/coinmonks/learn-to-deploy-smart-contracts-more-professionally-with-hardhat-1fec1dab8eac)

