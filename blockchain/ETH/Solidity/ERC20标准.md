[EIP-20: Token Standard (ethereum.org)](https://eips.ethereum.org/EIPS/eip-20)

## 简介

ERC-20（EIP 20）由 Fabian Vogelsteller 提出于 2015 年 11 月。这是一个能实现智能合约中代币的应用程序接口标准。

ERC-20 的功能示例包括：

-   将代币从一个帐户转到另一个帐户
-   balanceOf获取帐户的当前代币余额
-   totalSupply获取网络上可用代币的总供应量
-   批准（approve）一个帐户中一定的代币金额由第三方帐户使用

如果智能合约实施了下列方法和事件，它可以被称为 ERC-20 代币合约， 一旦部署，将负责跟踪以太坊上创建的代币。

```sol
function name() public view returns (string)
function symbol() public view returns (string)
function decimals() public view returns (uint8)
function totalSupply() public view returns (uint256)
function balanceOf(address _owner) public view returns (uint256 balance)
function transfer(address _to, uint256 _value) public returns (bool success)
function transferFrom(address _from, address _to, uint256 _value) public returns (bool success)
function approve(address _spender, uint256 _value) public returns (bool success)
function allowance(address _owner, address _spender) public view returns (uint256 remaining)

event Transfer(address indexed _from, address indexed _to, uint256 _value)
event Approval(address indexed _owner, address indexed _spender, uint256 _value)

```


## 代币规范

### 方法

注意。  
  
- 以下规范使用 Solidity 0.4.17 (或以上) 的语法  
- 调用者必须从返回（bool success）中处理 false。调用者决不能假定false永远不会被返回!  

name

返回代币的名称 - 例如 "MyToken"。  
  
OPTIONAL - 这个方法可以用来提高可用性，但是接口和其他合约不应该期望这些值出现。  
  
function name() public view returns (string)  

symbol  

返回代币的符号。例如："HIX"。  
  
OPTIONAL - 这个方法可以用来提高可用性，但是接口和其他合约不能期望这些值的出现。  
  
function symbol() public view returns (string)  

小数(decimals)  

返回代币使用的小数的数量--例如，8，意味着将代币金额除以100000000 （有8个0），以获得其用户表示。  
  
OPTIONAL - 这个方法可以用来提高可用性，但是接口和其他合约不可以期望这些值出现。  
  
function decimals() public view returns (uint8)  

totalSupply  
返回代币供应总量。  
  
function totalSupply() public view returns (uint256)  

balanceOf 
返回地址为_owner的另一个账户(该ERC20代币账户)的账户余额。  
  
function balanceOf(address \_owner) public view returns (uint256 balance)  

transfer  
将_value数额的代币转移到地址_to，并且必须触发transfer事件。如果消息调用者的账户余额没有足够的代币可以花费，该函数应该throw。  
  
注意 0值的转移必须被视为正常的转移，并触发转移事件。  
  
function transfer(address \_to, uint256 \_value) public returns (bool success)  

transferFrom  
从地址\_from向地址_to转移_value数量的代币，并且必须触发Transfer事件。  
  
transferFrom方法用于提取工作流，允许合约代表你转移代币。例如，这可用于允许合约代表你转移代币和/或以子货币收取费用。该函数应该throw，除非_from账户通过某种机制故意授权消息的发送者。  
  
注意0值的转移必须被视为正常的转移，并触发transfer事件。  
  
function transferFrom(address \_from, address \_to, uint256 \_value) public returns (bool success)  

approve

允许_spender从你的账户中多次取款，最多不超过_value的金额。如果这个函数被再次调用，它将用_value覆盖当前的allowance。  
  
注意：为了防止像这里描述的和这里讨论的那样的攻击媒介，客户应该确保在创建用户界面时，首先将allowance设置为0，然后再为同一spender设置为另一个值。然而，合约本身不应该强制执行它，以允许向后兼容之前部署的合约  
  
function approve(address \_spender, uint256 \_value) public returns (bool success)  

allowance  
返回_spender仍然被允许从_owner那里提取的金额。  
  
function allowance(address \_owner, address \_spender) public view returns (uint256 remaining)  
  
### 事件

Transfer

必须在代币被转移时触发，包括0值转移。

创建新代币的代币合约应该在创建代币时触发Transfer事件，\_from地址设置为0x0。

event Transfer(address indexed \_from, address indexed \_to, uint256 \_value)

Approval
必须在任何成功调用approve(address \_spender, uint256 \_value)时触发。

event Approval(address indexed \_owner, address indexed \_spender, uint256 \_value)

## 实现

已经有很多符合ERC20标准的代币部署在以太坊网络上。不同的团队编写了不同的实施方案，这些方案有不同的权衡：从节省gas到提高安全性。

[openzeppelin-contracts/ERC20.sol at 9b3710465583284b8c4c5d2245749246bb2e0094 · OpenZeppelin/openzeppelin-contracts (github.com)](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/9b3710465583284b8c4c5d2245749246bb2e0094/contracts/token/ERC20/ERC20.sol)

[Tokens/EIP20.sol at fdf687c69d998266a95f15216b1955a4965a0a6d · ConsenSys/Tokens (github.com)](https://github.com/ConsenSys/Tokens/blob/fdf687c69d998266a95f15216b1955a4965a0a6d/contracts/eip20/EIP20.sol)

