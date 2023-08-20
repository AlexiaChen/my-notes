
就是最近比较流行的ETH的合约，Dapp开发环境，我看了文档，感觉比truffle方便多了。

## 安装nodejs

```bash
sudo apt update sudo apt install curl git curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash - sudo apt-get install -y nodejs
```
或者如果在WSL2下

```bash
sudo apt install curl
curl -k -o- https://raw.githubusercontent.com/nvm-sh/nvm/master/install.sh | bash
nvm ls
nvm install --lts # or nvm install node
node --version
npm --version
nvm use --lts // or nvm use node    nvm use <version-number>
```

```bash
wget --no-check-certificate https://raw.githubusercontent.com/nvm-sh/nvm/master/inst  
all.sh
bash ./install.sh
```

## 创建一个新的hardhat工程

```bash
mkdir hardhat-tutorail && cd hardhat-tutorial
yarn init # npm init
yarn add --dev hardhat # npm install --save-dev hardhat
npx hardhat  # choose create an empty hardhat.config.ts
```

当Hardhat运行时，它会从当前工作目录开始搜索最近的`hardhat.config.js`文件。这个文件通常存在于你的项目的根部，一个空的`hardhat.config.js`足以让Hardhat工作。你的全部设置都包含在这个文件中。

### hardhat的架构

Hardhat是围绕任务和插件的概念设计的。Hardhat的大部分功能来自于插件，你可以自由选择你想使用的插件。

#### Tasks

每次你从命令行运行Hardhat时，你都在运行一个任务。例如，`npx hardhat compile`正在运行`compile`任务。要查看你的项目中当前可用的任务，请运行`npx hardhat`。你可以通过运行`npx hardhat help [task]`来探索任何任务。

> 自定义自己的任务参考 [Creating a task | Ethereum development environment for professionals by Nomic Foundation (hardhat.org)](https://hardhat.org/hardhat-runner/docs/advanced/create-task)


#### Plugins

在你最终使用什么工具方面，Hardhat是没有主见的，但它确实带有一些内置的默认值。所有这些都可以被重写(override)。大多数情况下，使用某个工具的方法是通过使用一个将其整合到Hardhat中的插件。

在本教程中，我们将使用我们推荐的插件。
`@nomicfoundation/hardhat-toolbox`，它拥有你开发智能合约所需的一切。

```bash
yarn add --dev @nomicfoundation/hardhat-toolbox @nomicfoundation/hardhat-network-helpers @nomicfoundation/hardhat-chai-matchers @nomiclabs/hardhat-ethers @nomiclabs/hardhat-etherscan chai ethers hardhat-gas-reporter solidity-coverage @typechain/hardhat typechain @typechain/ethers-v5 @ethersproject/abi @ethersproject/providers # npm install --save-dev @nomicfoundation/hardhat-toolbox
```

在你的`hardhat.config.js`中添加突出显示的一行，使其看起来像这样。

```typescript
require("@nomicfoundation/hardhat-toolbox"); 
/** @type import('hardhat/config').HardhatUserConfig */ 
module.exports = { 
solidity: "0.8.9", 
};
```

到此为止，项目的目录结构为:

```txt
mathxh@MathxH:~/Project/hardhat-tutorial$ tree -L 1 .
.
├── hardhat.config.js
├── node_modules
├── package.json
└── yarn.lock

1 directory, 3 files
```

## 编写并编译合约

我们将创建一个简单的智能合约，实现一个可以转让的代币。代币合约最常被用来交换或储存价值。我们不会在本教程中深入研究合约的Solidity代码，但有一些我们实现的逻辑，你应该知道：

- 有一个固定的代币总供应量（total supply），不能被改变。
- 整个供应量（supply）被分配给部署合约的地址。
- 任何人都可以收到代币。
- 任何拥有至少一个代币的人都可以转让代币。
- 代币是不可分割的。你可以转让1、2、3或37个代币，但不能转让2.5个。

> 你可能听说过ERC-20，它是以太坊的一个代币标准。DAI和USDC等代币实现了ERC-20标准，这使得它们都可以与任何可以处理ERC-20代币的软件兼容。为了简单起见，我们要建立的代币并没有实施ERC-20标准。

### 编写合约

首先，创建一个名为`contracts`的新目录，并在该目录内创建一个名为`Token.sol`的文件。

将下面的代码粘贴到该文件中，花一分钟时间阅读该代码。它很简单，而且充满了解释 Solidity 基本知识的注释。

> 最好安装hardhat的VSCode插件

```solidity
//SPDX-License-Identifier: UNLICENSED

// Solidity files have to start with this pragma.
// It will be used by the Solidity compiler to validate its version.
pragma solidity ^0.8.9;


// This is the main building block for smart contracts.
contract Token {
    // Some string type variables to identify the token.
    string public name = "My Hardhat Token";
    string public symbol = "MHT";

    // The fixed amount of tokens, stored in an unsigned integer type variable.
    uint256 public totalSupply = 1000000;

    // An address type variable is used to store ethereum accounts.
    address public owner;

    // A mapping is a key/value map. Here we store each account's balance.
    mapping(address => uint256) balances;

    // The Transfer event helps off-chain applications understand
    // what happens within your contract.
    event Transfer(address indexed _from, address indexed _to, uint256 _value);

    /**
     * Contract initialization.
     */
    constructor() {
        // The totalSupply is assigned to the transaction sender, which is the
        // account that is deploying the contract.
        balances[msg.sender] = totalSupply;
        owner = msg.sender;
    }

    /**
     * A function to transfer tokens.
     *
     * The `external` modifier makes a function *only* callable from *outside*
     * the contract.
     */
    function transfer(address to, uint256 amount) external {
        // Check if the transaction sender has enough tokens.
        // If `require`'s first argument evaluates to `false` then the
        // transaction will revert.
        require(balances[msg.sender] >= amount, "Not enough tokens");

        // Transfer the amount.
        balances[msg.sender] -= amount;
        balances[to] += amount;

        // Notify off-chain applications of the transfer.
        emit Transfer(msg.sender, to, amount);
    }

    /**
     * Read only function to retrieve the token balance of a given account.
     *
     * The `view` modifier indicates that it doesn't modify the contract's
     * state, which allows us to call it without executing a transaction.
     */
    function balanceOf(address account) external view returns (uint256) {
        return balances[account];
    }
}
```

> \*.sol 用于 Solidity 文件。我们建议将文件名与它所包含的合约相匹配，这也是一种常见的做法。


### 编译合约

要编译合约，在终端运行`npx hardhat compile`。编译任务是内置任务之一。

```bash
npx hardhat compile
```

编译完成后，项目目录下会多出两个文件夹: `cache`和`artifacts`， 其中artifacts有合约的ABI文件

```txt
mathxh@MathxH:~/Project/hardhat-tutorial$ tree -L 1 .
.
├── artifacts
├── cache
├── contracts
├── hardhat.config.js
├── node_modules
├── package.json
└── yarn.lock

4 directories, 3 files
```

##  测试合约

在构建智能合约时，编写自动化测试是至关重要的，因为你的用户的钱是危在旦夕。

为了测试我们的合约，我们将使用Hardhat network，一个为开发而设计的本地Ethereum网络（类似于truffle中的ganache）。它内置在Hardhat中，被作为默认网络使用。你不需要设置任何东西来使用它。

在我们的测试中，我们将使用`ethers.js` [Documentation (ethers.io)](https://docs.ethers.io/v5/) 与我们在上一节建立的以太坊合约进行交互，我们将使用`Mocha` [Mocha - the fun, simple, flexible JavaScript test framework (mochajs.org)](https://mochajs.org/) 作为我们的测试运行器。

### 编写测试用例

在我们的项目根目录下创建一个名为test的新目录，并在其中创建一个名为Token.js的新文件。

让我们从下面的代码开始。我们接下来会解释它，但现在先把这个粘贴到Token.js中。

> 注意，这些js库基本在之前的操作中yarn就安装好了

```javascript
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Token contract", function () {
  it("Deployment should assign the total supply of tokens to the owner", async function () {
    const [owner] = await ethers.getSigners();

    const Token = await ethers.getContractFactory("Token");

    const hardhatToken = await Token.deploy();

    const ownerBalance = await hardhatToken.balanceOf(owner.address);
    expect(await hardhatToken.totalSupply()).to.equal(ownerBalance);
  });
});
```
一般不出问题就编译过了，如果有错误，请查看，hardhat.config.js文件是否有问题，是否require了hardhat的toolbox。

下面来解释下代码，getSigners的意思是, ethers.js中的Signer是一个代表以太坊账户的对象 [Signers (ethers.io)](https://docs.ethers.io/v5/api/signer/) 。它被用来向合约和其他账户发送交易。在这里，我们得到了一个我们所连接的节点的账户列表，在这种情况下是Hardhat Network，我们只保留第一个账户(通过js的解构)。

ethers变量在全局范围内可用。如果你喜欢你的代码始终是明确的，你可以在顶部添加这一行。（新版中的hardhat可以不加这一行）

```js
const { ethers } = require("hardhat");
```

在ethers.js中，ContractFactory是一个用于部署新智能合约的抽象，所以这里的Token是我们的token合约实例的工厂。

在ContractFactory上调用deploy()将开始部署，并返回一个解析为Contract的Promise。这是为你的每个智能合约功能提供方法的对象。

一旦合约被部署，我们可以在hardhatToken上调用我们的合约方法。在这里，我们通过调用合约的balanceOf()方法获得所有者账户的余额。

回想一下，部署Token的账户会得到它的全部supply。默认情况下，ContractFactory和Contract实例连接到第一个Signer。这意味着所有者变量中的账户执行了部署，而balanceOf()应该返回整个供应量。

在这里，我们再次使用我们的合约实例在我们的Solidity代码中调用一个智能合约函数。 totalSupply()返回代币的供应量，我们正在检查它是否等于所有者余额，因为它应该是。

为了做到这一点，我们使用了Chai，这是一个流行的JavaScript断言库。这些断言函数被称为 "匹配器"，我们在这里使用的函数来自于 @nomicfoundation/hardhat-chai-matchers插件，它用许多对测试智能合约有用的匹配器对Chai进行了扩展。

#### 使用不同的账户

如果你需要通过从默认账户（或ethers.js术语中的Signer）发送交易来测试你的代码，你可以使用ethers.js合约对象的connect()方法将其连接到不同的账户，像这样。

```js
const { expect } = require("chai");

describe("Token contract", function () {
  // ...previous test...

  it("Should transfer tokens between accounts", async function() {
    const [owner, addr1, addr2] = await ethers.getSigners();

    const Token = await ethers.getContractFactory("Token");

    const hardhatToken = await Token.deploy();

    // Transfer 50 tokens from owner to addr1
    await hardhatToken.transfer(addr1.address, 50);
    expect(await hardhatToken.balanceOf(addr1.address)).to.equal(50);

    // Transfer 50 tokens from addr1 to addr2
    await hardhatToken.connect(addr1).transfer(addr2.address, 50);
    expect(await hardhatToken.balanceOf(addr2.address)).to.equal(50);
  });
});
```

#### 用fixtures复用公共的测试setups

我们写的两个测试从它们的设置开始，在这种情况下意味着部署代币合约。在更复杂的项目中，这种设置可能涉及多个部署和其他事务。在每个测试中这样做意味着大量的代码重复。此外，在每个测试开始时执行许多事务会使测试套件变得更慢。

你可以通过使用Fixtures来避免代码重复并提高测试套件的性能。fixtures是一个设置函数，只在第一次调用时运行。在随后的调用中，Hardhat将把网络的状态重置为最初执行fixture后的状态，而不是重新运行它。

```js
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");
const { expect } = require("chai");

describe("Token contract", function () {
  async function deployTokenFixture() {
    const Token = await ethers.getContractFactory("Token");
    const [owner, addr1, addr2] = await ethers.getSigners();

    const hardhatToken = await Token.deploy();

    await hardhatToken.deployed();

    // Fixtures can return anything you consider useful for your tests
    return { Token, hardhatToken, owner, addr1, addr2 };
  }

  it("Should assign the total supply of tokens to the owner", async function () {
    const { hardhatToken, owner } = await loadFixture(deployTokenFixture);

    const ownerBalance = await hardhatToken.balanceOf(owner.address);
    expect(await hardhatToken.totalSupply()).to.equal(ownerBalance);
  });

  it("Should transfer tokens between accounts", async function () {
    const { hardhatToken, owner, addr1, addr2 } = await loadFixture(
      deployTokenFixture
    );

    // Transfer 50 tokens from owner to addr1
    await expect(
      hardhatToken.transfer(addr1.address, 50)
    ).to.changeTokenBalances(hardhatToken, [owner, addr1], [-50, 50]);

    // Transfer 50 tokens from addr1 to addr2
    // We use .connect(signer) to send a transaction from another account
    await expect(
      hardhatToken.connect(addr1).transfer(addr2.address, 50)
    ).to.changeTokenBalances(hardhatToken, [addr1, addr2], [-50, 50]);
  });
});
```

在这里，我们写了一个deployTokenFixture函数，它做了必要的设置，并返回我们以后在测试中使用的每个值。然后在每个测试中，我们使用loadFixture来运行fixture并获得这些值。loadFixture将在第一次运行设置，并在其他测试中快速返回到该状态。

#### 全覆盖测试

现在我们已经涵盖了测试合约所需的基础知识，这里有一个完整的代币测试套件，其中有很多关于Mocha和如何构建测试的额外信息。我们建议彻底阅读它。

```js
// This is an example test file. Hardhat will run every *.js file in `test/`,
// so feel free to add new ones.

// Hardhat tests are normally written with Mocha and Chai.

// We import Chai to use its asserting functions here.
const { expect } = require("chai");

// We use `loadFixture` to share common setups (or fixtures) between tests.
// Using this simplifies your tests and makes them run faster, by taking
// advantage of Hardhat Network's snapshot functionality.
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

// `describe` is a Mocha function that allows you to organize your tests.
// Having your tests organized makes debugging them easier. All Mocha
// functions are available in the global scope.
//
// `describe` receives the name of a section of your test suite, and a
// callback. The callback must define the tests of that section. This callback
// can't be an async function.
describe("Token contract", function () {
  // We define a fixture to reuse the same setup in every test. We use
  // loadFixture to run this setup once, snapshot that state, and reset Hardhat
  // Network to that snapshot in every test.
  async function deployTokenFixture() {
    // Get the ContractFactory and Signers here.
    const Token = await ethers.getContractFactory("Token");
    const [owner, addr1, addr2] = await ethers.getSigners();

    // To deploy our contract, we just have to call Token.deploy() and await
    // its deployed() method, which happens onces its transaction has been
    // mined.
    const hardhatToken = await Token.deploy();

    await hardhatToken.deployed();

    // Fixtures can return anything you consider useful for your tests
    return { Token, hardhatToken, owner, addr1, addr2 };
  }

  // You can nest describe calls to create subsections.
  describe("Deployment", function () {
    // `it` is another Mocha function. This is the one you use to define each
    // of your tests. It receives the test name, and a callback function.
    //
    // If the callback function is async, Mocha will `await` it.
    it("Should set the right owner", async function () {
      // We use loadFixture to setup our environment, and then assert that
      // things went well
      const { hardhatToken, owner } = await loadFixture(deployTokenFixture);

      // `expect` receives a value and wraps it in an assertion object. These
      // objects have a lot of utility methods to assert values.

      // This test expects the owner variable stored in the contract to be
      // equal to our Signer's owner.
      expect(await hardhatToken.owner()).to.equal(owner.address);
    });

    it("Should assign the total supply of tokens to the owner", async function () {
      const { hardhatToken, owner } = await loadFixture(deployTokenFixture);
      const ownerBalance = await hardhatToken.balanceOf(owner.address);
      expect(await hardhatToken.totalSupply()).to.equal(ownerBalance);
    });
  });

  describe("Transactions", function () {
    it("Should transfer tokens between accounts", async function () {
      const { hardhatToken, owner, addr1, addr2 } = await loadFixture(
        deployTokenFixture
      );
      // Transfer 50 tokens from owner to addr1
      await expect(
        hardhatToken.transfer(addr1.address, 50)
      ).to.changeTokenBalances(hardhatToken, [owner, addr1], [-50, 50]);

      // Transfer 50 tokens from addr1 to addr2
      // We use .connect(signer) to send a transaction from another account
      await expect(
        hardhatToken.connect(addr1).transfer(addr2.address, 50)
      ).to.changeTokenBalances(hardhatToken, [addr1, addr2], [-50, 50]);
    });

    it("should emit Transfer events", async function () {
      const { hardhatToken, owner, addr1, addr2 } = await loadFixture(
        deployTokenFixture
      );

      // Transfer 50 tokens from owner to addr1
      await expect(hardhatToken.transfer(addr1.address, 50))
        .to.emit(hardhatToken, "Transfer")
        .withArgs(owner.address, addr1.address, 50);

      // Transfer 50 tokens from addr1 to addr2
      // We use .connect(signer) to send a transaction from another account
      await expect(hardhatToken.connect(addr1).transfer(addr2.address, 50))
        .to.emit(hardhatToken, "Transfer")
        .withArgs(addr1.address, addr2.address, 50);
    });

    it("Should fail if sender doesn't have enough tokens", async function () {
      const { hardhatToken, owner, addr1 } = await loadFixture(
        deployTokenFixture
      );
      const initialOwnerBalance = await hardhatToken.balanceOf(owner.address);

      // Try to send 1 token from addr1 (0 tokens) to owner (1000 tokens).
      // `require` will evaluate false and revert the transaction.
      await expect(
        hardhatToken.connect(addr1).transfer(owner.address, 1)
      ).to.be.revertedWith("Not enough tokens");

      // Owner balance shouldn't have changed.
      expect(await hardhatToken.balanceOf(owner.address)).to.equal(
        initialOwnerBalance
      );
    });
  });
});
```

以下是我写的全测试:

```js
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Token contract", function () {
    async function deployTokenFixture() {
        const Token = await ethers.getContractFactory("Token");
        const [owner, addr1, addr2] = await ethers.getSigners();
        console.log('owner: %s\n addr1: %s\n addr2: %s\n', owner, addr1, addr2)
        const hardhatToken = await Token.deploy();

        await hardhatToken.deployed();

        // Fixtures can return anything you consider useful for your tests
        return { Token, hardhatToken, owner, addr1, addr2 };
    }

    describe("Deployment", function() {
        it("Should set the right owner", async function () {
            const { hardhatToken, owner } = await loadFixture(deployTokenFixture);
            expect(await hardhatToken.owner()).to.equal(owner.address);
        });
        
        it("Deployment should assign the total supply of tokens to the owner", async function () {
            const {hardhatToken, owner} = await loadFixture(deployTokenFixture)
            const ownerBalance = await hardhatToken.balanceOf(owner.address);
            expect(await hardhatToken.totalSupply()).to.equal(ownerBalance);
        });
    })
  
    describe("Transactions", function () {
        it("Should transfer tokens between accounts", async function() {
            const { hardhatToken, owner, addr1, addr2 } = await loadFixture(
                deployTokenFixture
            );
    
            // Transfer 50 tokens from owner to addr1
            await hardhatToken.transfer(addr1.address, 50);
            expect(await hardhatToken.balanceOf(addr1.address)).to.equal(50);
    
            // Transfer 50 tokens from addr1 to addr2
            await hardhatToken.connect(addr1).transfer(addr2.address, 50);
            expect(await hardhatToken.balanceOf(addr2.address)).to.equal(50);
        });

        it("should emit Transfer events", async function () {
            const { hardhatToken, owner, addr1, addr2 } = await loadFixture(
              deployTokenFixture
            );
      
            // Transfer 50 tokens from owner to addr1
            await expect(hardhatToken.transfer(addr1.address, 50))
              .to.emit(hardhatToken, "Transfer")
              .withArgs(owner.address, addr1.address, 50);
      
            // Transfer 50 tokens from addr1 to addr2
            // We use .connect(signer) to send a transaction from another account
            await expect(hardhatToken.connect(addr1).transfer(addr2.address, 50))
              .to.emit(hardhatToken, "Transfer")
              .withArgs(addr1.address, addr2.address, 50);
          });

        it("Should fail if sender doesn't have enough tokens", async function () {
            const { hardhatToken, owner, addr1 } = await loadFixture(
                deployTokenFixture
            );
            const initialOwnerBalance = await hardhatToken.balanceOf(owner.address);
        
            // Try to send 1 token from addr1 (0 tokens) to owner (1000 tokens).
            // `require` will evaluate false and revert the transaction.
            await expect(
                hardhatToken.connect(addr1).transfer(owner.address, 1)
            ).to.be.revertedWith("Not enough tokens");
        
            // Owner balance shouldn't have changed.
            expect(await hardhatToken.balanceOf(owner.address)).to.equal(
                initialOwnerBalance
            );
        });
    })
});
```

注意每次的it函数loadFixture函数调用的Fixture函数对合约的部署，都是最初的状态，addr1，addr2都是没有金额的，也就是it函数的测试，不依赖其他it函数中的状态变化。


## 在Hardhat network中调试合约

Hardhat内置了Hardhat Network，这是一个为开发而设计的本地Ethereum网络。它允许你部署你的合约，运行你的测试和调试你的代码，所有这些都在你的本地机器的范围内。这是Hardhat连接的默认网络，所以你不需要为它的工作设置任何东西。只要运行你的测试。

### Solidity中的console.log

在hardhat框架下开发的合约可以临时在solidity合约源码中加入以下日志打印来调试合约:

```sol
pragma solidity ^0.8.9;
import "hardhat/console.sol";

contract Token {
// ...
}
```

然后你就可以向transfer()函数添加一些console.log调用，就像你在JavaScript中使用它一样。

```solidity
function transfer(address to, uint256 amount) external {
    require(balances[msg.sender] >= amount, "Not enough tokens");

    console.log(
        "Transferring from %s to %s %s tokens",
        msg.sender,
        to,
        amount
    );

    balances[msg.sender] -= amount;
    balances[to] += amount;

    emit Transfer(msg.sender, to, amount);
}
```

然后运行npx hardhat test就可以看到合约打印出来的日志了。

## 部署到一个实时的网络

之前都是在本地网络中运行，现在要部署到一个实时的网络，比如以太坊的测试网或者其他之类的。

一旦你准备好与其他人分享你的dApp，你可能想把它部署到一个实时网络。这样，其他人就可以访问一个不在你的系统上本地运行的实例。  
  
以太坊的 "主网 "网络与真钱打交道，但也有独立的 "测试网 "网络没有。这些测试网提供了共享的暂存环境，很好地模仿了现实世界的场景，而没有把真金白银置于危险之中，以太坊有几个，比如Goerli和Sepolia。我们建议你将你的合约部署到Goerli测试网。  
  
在软件层面，部署到测试网与部署到主网是一样的。唯一的区别是你连接到哪个网络。让我们看看使用ethers.js部署合约的代码是什么样子的。  
  
使用的主要概念是Signer、ContractFactory和Contract，我们在测试部分解释过。与测试相比，没有什么新的东西需要做，因为当你测试你的合约时，你实际上是在向你的开发网络进行部署。这使得代码非常相似，甚至是相同的。  
  
让我们在项目根目录下创建一个新的目录scripts，并在该目录下的deploy.js文件中粘贴以下内容。 

```js
async function main() {
  const [deployer] = await ethers.getSigners();

  console.log("Deploying contracts with the account:", deployer.address);

  console.log("Account balance:", (await deployer.getBalance()).toString());

  const Token = await ethers.getContractFactory("Token");
  const token = await Token.deploy();

  console.log("Token address:", token.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

要告诉Hardhat连接到一个特定的以太坊网络，你可以在运行任何任务时使用 --network参数，像这样。

```bash
npx hardhat run scripts/deploy.js --network <network-name>
```

在我们目前的配置中，在没有 --network 参数的情况下运行它，会导致代码针对 Hardhat Network 的一个嵌入式实例运行。在这种情况下，当Hardhat结束运行时，部署实际上会丢失，但它对于测试我们的部署代码是否有效仍然很有用。

```bash
npx hardhat run scripts/deploy.js # 这条命令很快结束，所以hardhat network也很快结束了
```

所以可以单独启动hardhat network，这样就可以支持钱包，或者外部客户端连接进来，比如MetaMask，你的Dapp前端，或者hardhat的scripts。

```bash
npx hardhat node
npx hardhat run --network localhost scripts/deploy.js
```
如果你单独启动harhat network的节点，并且运行以上的命令部署，那么在节点的终端会打印出类似以下的日志，说明合约部署在单独的hardhat network上成功了。

```txt
WARNING: These accounts, and their private keys, are publicly known.  
Any funds sent to them on Mainnet or any other live network WILL BE LOST.  
  
eth_accounts  
eth_chainId (2)  
eth_getBalance  
eth_accounts  
eth_blockNumber  
eth_chainId (2)  
eth_estimateGas  
eth_getBlockByNumber  
eth_gasPrice  
eth_sendTransaction  
Contract deployment: Token  
Contract address: 0x5fbdb2315678afecb367f032d93f642f64180aa3  
Transaction: 0x1fc2fe4da2aab042c39c1227756fa5300a5649763bb4a56022e17977df68ecca  
From: 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266  
Value: 0 ETH  
Gas used: 620608 of 620608  
Block #1: 0x3dbb5ce870fecea1cb2c6b2a940e9c2ae003b58476c21a383d51437597c44afd  
  
eth_chainId  
eth_getTransactionByHash
```

当然，hardhat还可以fork以太坊主网的状态，这样就可以本地联调主网的合约，更多详情请参考
[Hardhat Network | Ethereum development environment for professionals by Nomic Foundation](https://hardhat.org/hardhat-network/docs/overview)

### 部署到远程网络

要部署到远程网络，如mainnet或任何testnet，你需要在hardhat.config.js文件中添加一个网络条目。在这个例子中我们将使用Goerli，但你也可以类似地添加任何网络。

```js
require("@nomicfoundation/hardhat-toolbox");

// Go to https://www.alchemyapi.io, sign up, create
// a new App in its dashboard, and replace "KEY" with its key
const ALCHEMY_API_KEY = "KEY";

// Replace this private key with your Goerli account private key
// To export your private key from Metamask, open Metamask and
// go to Account Details > Export Private Key
// Beware: NEVER put real Ether into testing accounts
const GOERLI_PRIVATE_KEY = "YOUR GOERLI PRIVATE KEY";

module.exports = {
  solidity: "0.8.9",
  networks: {
    goerli: {
      url: `https://eth-goerli.alchemyapi.io/v2/${ALCHEMY_API_KEY}`,
      accounts: [GOERLI_PRIVATE_KEY]
    }
  }
};
```

我们使用的是Alchemy [Alchemy - the web3 development platform](https://www.alchemy.com/) ，这是一个Web3 Dapp的开发平台，但将网址指向任何以太坊节点或网关都可以。去拿你的ALCHEMY_API_KEY然后回来。

要在Goerli上部署，你需要发送一些Goerli ether到将要进行部署的地址。你可以从faucet获得testnet ether，一个免费分发测试ETH的服务。这里有一些用于Goerli的。

- Chainlink龙头 [Faucets | Chainlink](https://faucets.chain.link/) 
- 炼金术Goerli龙头 [Goerli Faucet](https://goerlifaucet.com/)

在交易之前，你必须将Metamask的网络改为Goerli。其他测试网的Faucet资料在这里 [Networks | ethereum.org](https://ethereum.org/en/developers/docs/networks/#ethereum-testnets) 

最后运行 npx hardhat run scripts/deploy.js --network goerli 

## 合约的Verify和Publish

如果了解合约的开发和发布，大家就可以在etherscan上通过搜索合约地址，就可以看到对应的合约源码，如果没有标题上的这步骤，就看不到，只是一堆字节码。所以这个是怎么办到的呢？

[Verify & Publish Contract Source Code | Etherscan](https://etherscan.io/verifyContract)

### 在etherscan上verify 合约

要在Etherscan上更新代币信息，必须对代币的代币合约地址进行验证。这是为了确保合约代码正是部署在区块链上的内容，同时也允许公众审计和阅读合约。Etherscan确保所有的代币合约必须经过验证，才能用合约owner提交的信息进行更新。

如果您是合约owner，并希望验证您的合约owner，请按照以下步骤进行。

- 验证和发布

在合约地址下，在 "transcation "标签旁边，你将能够找到 "code "标签。然后点击 "verify and publish"。

![[Pasted image 20221029121701.png]]

- 验证合约代码

对于Truffle部署的合约，合约所有者可以使用Etherscan新的Beta源代码合约验证器，它支持验证合约代码页面上方的 "run "选项。

![[Pasted image 20221029121752.png]]

输入了所需的信息，合约的名称，编译器版本，优化选项，并输入了完整的合约代码。合约代码应该像部署的那样，在一个文件中，平铺直叙(Flatten)，所有的导入都要删除。

构造函数参数和其他库也可以在同一页面的底部输入。

![[Pasted image 20221029121849.png]]

在点击VERIFY and publish按钮之前，请完成reCAPTCHA，合约应该得到验证。

- 验证合约详情

![[Pasted image 20221029121935.png]]

当合约被验证后，"代码 "页面将被填入合约的详细信息。该合约的源代码现在可以在Etherscan上公开获得。

详情也可以参考 ChainLink的博客 [How To Verify a Smart Contract on Etherscan | Chainlink Blog](https://blog.chain.link/how-to-verify-a-smart-contract-on-etherscan/)

如果用Hardhat开发，可以使用hardhat的ethersca插件做到，详情请看 [Verifying your contracts | Ethereum development environment for professionals by Nomic Foundation (hardhat.org)](https://hardhat.org/hardhat-runner/docs/guides/verifying)

[hardhat-etherscan | Ethereum development environment for professionals by Nomic Foundation](https://hardhat.org/hardhat-runner/plugins/nomiclabs-hardhat-etherscan)

如果用ETHerscan来verify和publish，那么选择一个工具 [paulrberg/multisol: CLI application for verifying Solidity contracts on Etherscan (github.com)](https://github.com/paulrberg/multisol) ，这个工具相当于flatten，把一个合约打包在一个目录下，到时候在etherscan上上传合约代码的时候，就上传这个目录下的合约代码就行了。

还有就是目前我的hardhat verify不能正在工作，即使挂了翻墙代理也不行，解决不了问题，所以还是用multisol这个工具把打包的合约上传合约到Remix IDE里面编译，记住编译器的版本，很重要，选择 multi-part，和MIT协议  上传合约。如果碰到有constructor的合约，verify需要传递参数，需要把你的部署的constructor的input参数记住下来，用abi-hashex来编码 [ABI Encoding Service Online for Solidity Smart Contracts by HashEx](https://abi.hashex.org/) 然后就可以verify和publish成功了。用Remix也可以用来发布合约到goerli等测试网和正式网，只要在enviroment中选择inject provider metamask就行，确保你的账号里面有钱。

