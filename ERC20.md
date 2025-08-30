

### ERC20 学习笔记：你的第一个数字资产

#### 一、 什么是 ERC20？

ERC20 (Ethereum Request for Comment 20) 是为以太坊区块链上的**同质化代币** (Fungible Tokens) 制定的一个技术标准。

*   **标准 (Standard)**: 它不是一段代码，也不是一个软件，而是一套**规则**或**接口**。所有遵循这套规则的智能合约，都可以被认为是 ERC20 代币。
*   **同质化 (Fungible)**: 这是核心概念。意思是每一个代币都是完全相同、可以互换的。就像你钱包里的一元钱和我钱包里的一元钱，它们没有区别，价值也完全一样。这与 NFT (ERC721) 的“非同质化”（每个都独一无二）形成鲜明对比。

**为什么这个标准如此重要？**
因为它实现了**互操作性 (Interoperability)**。钱包（如 MetaMask）、交易所（如 Uniswap）、DeFi 应用等，只需要编写一次与 ERC20 标准交互的代码，就能支持**所有**符合该标准的代币。这极大地促进了以太坊生态的繁荣。

#### 二、 ERC20 的核心方法 (The Interface)

ERC20 标准规定了所有代币合约**必须**实现的一组函数和事件。下面我们逐一解析。

##### (A) 可选的“元数据”方法 (Read-Only)

这些函数用于描述代币的基本信息，虽然标准中是可选的，但实践中几乎所有代币都会实现。

*   `name() returns (string)`
    *   **作用**: 返回代币的全名，例如 "Tether USD"。
    *   **场景**: 在钱包或交易所中显示给用户看。
*   `symbol() returns (string)`
    *   **作用**: 返回代币的符号，通常是 3-4 个字母，例如 "USDT"。
    *   **场景**: 在 UI 界面中简洁地代表代币。
*   `decimals() returns (uint8)`
    *   **作用**: 返回代币的小数位数。这非常重要，因为它决定了代币的最小单位。
    *   **场景**: USDT 的 `decimals` 是 6，意味着 `1,000,000` 个最小单位等于 `1` USDT。WETH 的 `decimals` 是 18。UI 在显示余额时，需要用原始余额除以 `10**decimals`。

##### (B) 核心的“账本”方法 (Read-Only)

这些函数用于查询代币的状态。

*   `totalSupply() returns (uint256)`
    *   **作用**: 返回当前已发行的代币总数量。
    *   **场景**: 查看代币的整体规模。
*   `balanceOf(address account) returns (uint256)`
    *   **作用**: 查询指定地址 (`account`) 的代币余额。
    *   **场景**: 这是最常用的查询功能，用于显示任何人的持币数量。

##### (C) 核心的“交易”方法 (State-Changing)

这些函数会改变合约的状态（账本），调用它们需要花费 Gas。

*   `transfer(address to, uint256 amount) returns (bool)`
    *   **作用**: 从调用者 (`msg.sender`) 的账户向接收方 (`to`) 地址转移 `amount` 数量的代币。
    *   **场景**: **个人转账**。Alice 想直接给 Bob 转 10 个币，她就调用 `transfer(Bob的地址, 10)`。

*   `approve(address spender, uint256 amount) returns (bool)`
    *   **作用**: 授权 (`approve`) 一个第三方地址 (`spender`) 可以从调用者 (`msg.sender`) 的账户中提取**最多** `amount` 数量的代币。
    *   **注意**: 这个函数**本身不发生转账**，它只是一个**授权**动作。
    *   **场景**: **与智能合约交互**。当你想在 Uniswap 上卖出你的代币时，你必须先 `approve` Uniswap 的路由器合约，授权它可以从你的账户里拿走代币。

*   `transferFrom(address from, address to, uint256 amount) returns (bool)`
    *   **作用**: 由被授权方 (`msg.sender`) 调用，从授权方 (`from`) 的账户中，将 `amount` 数量的代币转移给接收方 (`to`)。
    *   **前提**: `from` 地址必须已经 `approve` 了 `msg.sender` 至少 `amount` 的额度。
    *   **场景**: 紧接着上面的场景，当你执行兑换操作时，Uniswap 的合约就会调用 `transferFrom(你的地址, Uniswap的流动性池地址, amount)`，把你的代币转走。

*   `allowance(address owner, address spender) returns (uint256)`
    *   **作用**: 查询 `owner` 地址授权给了 `spender` 地址多少额度可以提取。
    *   **场景**: DApp 的前端可以用这个函数来检查用户是否已经授权，以及授权了多少额度，从而决定是显示 "Approve" 按钮还是 "Swap" 按钮。

#### 三、 OpenZeppelin 的 ERC20 实现

直接从零实现上述所有方法是复杂且危险的。OpenZeppelin 提供了安全、高效的实现。

```solidity
// 在你的合约中，你只需要这样做：
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor() ERC20("My Token", "MYT") {
        // ...
    }
}
```

OpenZeppelin 还提供了一些非常有用的**内部函数 (Internal Functions)**，它们通常以下划线 `_` 开头，只能在你的合约内部或子合约中调用：

*   `_mint(address account, uint256 amount)`
    *   **作用**: **凭空创造**新的代币，并分配给 `account` 地址。这会增加 `totalSupply`。
    *   **场景**: 在合约部署时，为创始人分配初始代币；或者在一个众筹合约中，当用户发送 ETH 时，为用户铸造新的代币。

*   `_burn(address account, uint256 amount)`
    *   **作用**: **销毁** `account` 地址持有的代币。这会减少 `totalSupply`。
    *   **场景**: 实现通缩模型，或者当用户用代币兑换某种服务后，将代币销毁。

*   `_transfer(address from, address to, uint256 amount)`
    *   **作用**: `transfer` 和 `transferFrom` 函数内部调用的核心逻辑。
*   `_approve(address owner, address spender, uint256 amount)`
    *   **作用**: `approve` 函数内部调用的核心逻辑。

#### 四、 核心工作流：`approve` + `transferFrom`

这是初学者最容易混淆的地方，但它对 DeFi 至关重要。

**为什么不直接用 `transfer` 去调用合约？**
如果你直接 `transfer` 代币给一个合约地址，该合约**无法知道是谁转给了它**，也无法自动触发下一步操作。这种 "推" (Push) 的模式信息量太少。

`approve` + `transferFrom` 是一种 "拉" (Pull) 的模式：
1.  **你 (Alice)** 告诉 **代币合约 (MYT)**: "我授权**DEX合约**可以从我的账户里拉走最多 100 个 MYT"。
    *   `Alice -> MYT.approve(DEX_Address, 100)`
2.  **你 (Alice)** 告诉 **DEX合约**: "请帮我用 50 个 MYT 兑换成 ETH"。
    *   `Alice -> DEX.swap(MYT_Address, 50)`
3.  **DEX合约** 收到你的请求后，它自己去调用 **代币合约 (MYT)**: "我来拉走 Alice 授权给我的 50 个 MYT"。
    *   `DEX -> MYT.transferFrom(Alice_Address, DEX_Address, 50)`

这个流程让 DEX 合约可以代表你安全地操作你的代币，从而完成复杂的交易逻辑。

#### 五、 应用场景

ERC20 代币是 Web3 世界的基石，应用无处不在：

1.  **稳定币 (Stablecoins)**: 价值与法币（如美元）锚定的代币，是 DeFi 世界的通用货币。
    *   **例子**: USDT, USDC, DAI。
2.  **治理代币 (Governance Tokens)**: 持有者可以对协议的未来发展进行投票，参与社区治理。
    *   **例子**: UNI (Uniswap), AAVE (Aave), MKR (MakerDAO)。
3.  **功能型代币 (Utility Tokens)**: 用于访问一个产品或服务的特定功能。
    *   **例子**: LINK (Chainlink) 用于支付预言机服务，BAT (Basic Attention Token) 用于在 Brave 浏览器生态中奖励用户和创作者。
4.  **奖励/积分代币**: 作为平台对用户的奖励或忠诚度积分。
    *   **例子**: 各种 Play-to-Earn 游戏中的代币，或者社区贡献者的奖励。

六、主要代码

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount)
        external
        returns (bool);
    function allowance(address owner, address spender)
        external
        view
        returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount)
        external
        returns (bool);
}
```

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "./IERC20.sol";

contract ERC20 is IERC20 {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(
        address indexed owner, address indexed spender, uint256 value
    );

    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    string public name;
    string public symbol;
    uint8 public decimals;

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
    }

    function transfer(address recipient, uint256 amount)
        external
        returns (bool)
    {
        balanceOf[msg.sender] -= amount;
        balanceOf[recipient] += amount;
        emit Transfer(msg.sender, recipient, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address sender, address recipient, uint256 amount)
        external
        returns (bool)
    {
        allowance[sender][msg.sender] -= amount;
        balanceOf[sender] -= amount;
        balanceOf[recipient] += amount;
        emit Transfer(sender, recipient, amount);
        return true;
    }

    function _mint(address to, uint256 amount) internal {
        balanceOf[to] += amount;
        totalSupply += amount;
        emit Transfer(address(0), to, amount);
    }

    function _burn(address from, uint256 amount) internal {
        balanceOf[from] -= amount;
        totalSupply -= amount;
        emit Transfer(from, address(0), amount);
    }

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }

    function burn(address from, uint256 amount) external {
        _burn(from, amount);
    }
}
```

