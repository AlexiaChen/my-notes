
# 在solidity中调用另一个合约的函数


当我们编写智能合约时，我们可以以这样的方式编写它，即它们可以与现有部署的合约进行互动。这个功能非常强大，因为它允许代码重用，即像对待库一样对待部署的合同。在这个领域已经做了很多努力，但在编写时仍有争议。例如，如果重复使用的合同有问题，会发生什么情况（就像发生在[Parity Multisig Hacked. Again. Yesterday, Parity Multisig Wallet was… | by Anthony Akentiev | Chain.Cloud company blog | Medium](https://medium.com/chain-cloud-company-blog/parity-multisig-hack-again-b46771eaa838)）？

在这篇文章中，我不是在争论部署的合约的不可更改性，或者我们是否应该或不应该与部署的合约互动。相反，我将专注于不同的技术来调用部署的合同的功能。我可以看到它的一些用例，我将把它留给读者来实现他们相信的东西。

假设我们已经部署了一个非常简单的合约，名为 "Deployed"，允许用户设置一个变量。

```d
pragma solidity ^0.6.12;
contract Deployed {
    uint public a = 1;
    
    function setA(uint _a) public returns (uint) {
        a = _a;
        return a;
    }
    
}
```

而我们想在以后部署另一个名为 "Existing"的合同，以改变 "Deployed "合同中 "a "的变量。

```d
pragma solidity ^0.6.12;
contract Deployed {
    
    function setA(uint) public returns (uint) {}
    
    function a() public pure returns (uint) {}
    
}
contract Existing  {
    
    Deployed dc;
    
     constructor(address _t) public {
        dc = Deployed(_t);
    }
 
    function getA() public view returns (uint result) {
        return dc.a();
    }
    
    function setA(uint _val) public returns (uint result) {
        dc.setA(_val);
        return _val;
    }
    
}
```

以上就是通过线上的合约地址，把地址强制转换为Deployed的合约类型，来调用，当然，这个合约类型你需要在Existing合约的上方声明，不然solidity编译器不知道Deployed合约的类型是什么。

我们不需要 "Deployed "合同的完整实现，而只需要ABI所要求的函数签名。既然我们有 "Deployed "合同的地址，我们可以用这个地址初始化 "Existing "合同，并相应地使用现有的setA和getA函数与 "Deployed "合同交互。

这很简单，而且实际上是与Deployed合同进行交互的推荐方式。然而，如果我们没有Deployed合同的ABI呢？我们仍然可以调用Deployed合同的 "setA "函数。

```d
pragma solidity ^0.4.18;
contract ExistingWithoutABI  {
    
    address dc;
    
    function ExistingWithoutABI(address _t) public {
        dc = _t;
    }
    
    function setA_Signature(uint _val) public returns(bool success){
        require(dc.call(bytes4(keccak256("setA(uint256)")),_val));
        return true;
    }
}
```
本质上你还是需要知道setA函数的签名。

函数签名的长度为4个字节，生成它的公式是用keccak256函数进行Hash，像这样

```d
bytes4(keccak256(“setA(uint256)”))
```

我们可以在调用方法中向setA传递一个值。然而，由于call（以及delegatecall）方法只是将值传递给合同地址，而不会得到任何返回值，所以除非我们检查出 "Delegate "合同的状态，否则它不知道setA是否正确完成了它的工作。

如果我们想从setA中获得返回值怎么办？不幸的是，除非我们使用 solidity 的汇编代码 [Inline Assembly — Solidity 0.8.18 documentation (soliditylang.org)](https://docs.soliditylang.org/en/develop/assembly.html) ，否则没有办法做到这一点。你准备好了吗？

```d
pragma solidity ^0.4.18;
contract ExistingWithoutABI  {
    
    address dc;
    
    function ExistingWithoutABI(address _t) public {
        dc = _t;
    }
    
    function setA_ASM(uint _val) public returns (uint answer) {
        
        bytes4 sig = bytes4(keccak256("setA(uint256)"));
        assembly {
            // move pointer to free memory spot
            let ptr := mload(0x40)
            // put function sig at memory spot
            mstore(ptr,sig)
            // append argument after function sig
            mstore(add(ptr,0x04), _val)

            let result := call(
              15000, // gas limit
              sload(dc_slot),  // to addr. append var to _slot to access storage variable
              0, // not transfer any ether
              ptr, // Inputs are stored at location ptr
              0x24, // Inputs are 36 bytes long
              ptr,  //Store output over input
              0x20) //Outputs are 32 bytes long
            
            if eq(result, 0) {
                revert(0, 0)
            }
            
            answer := mload(ptr) // Assign output to answer var
            mstore(0x40,add(ptr,0x24)) // Set storage pointer to new space
        }
    }
}
```

solidity的汇编代码以 "汇编 "关键字开始，并以{}包裹。我希望我在代码中的评论是清楚的。为了在没有ABI的情况下得到setA的返回值，我们必须了解内存在EVM中是如何工作的。在第64个字节（0x40）处有空闲的内存，所以我们首先将内存指针移到那里。然后，我们将函数签名和参数的十六进制数依次附加到该处。函数签名是4字节（0x04），参数是32字节（0x20），所以我们总共有36字节（0x24）。

一旦完成，我们进行神奇的 "调用"，将结果存储在第64个字节的位置，并返回一个布尔值，即1或0，如果调用失败（返回0），交易将恢复。假设一切成功，我们返回第64个字节的值，这就是我们想要的答案。

试试remix中的代码，自己看看吧。

编码愉快。

当然，对于address.call的调用，在新版本的solidity中有新的用法: [addresses - How to use address.call{}() in Solidity - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/96685/how-to-use-address-call-in-solidity) , 可读性更高。


# 怎样在你的solidity代码中调用另一个合约

记住，一般来说在同一个sol文件中的不同的contract关键字描述的合约，会分开部署到不同的地址，其实就是编译器把他们拆分开了。下面演示，通过interface接口来调用对接口实现的合约。也就是把具体的合约地址，转换为interface的类型，这个就是合约的多态性。

```d
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

interface ICounter {
    function count() external view returns (uint);

    function increment() external;
}

contract Counter is ICounter {
    uint public count;

    function increment() external {
        count += 1;
    }
}



contract MyContract {
    function incrementCounter(address _counter) external {
        ICounter(_counter).increment();
    }

    function getCount(address _counter) external view returns (uint) {
        return ICounter(_counter).count();
    }
}

```

```d
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

contract Counter {
    uint public count;

    function increment() external {
        count += 1;
    }
}

interface ICounter {
    function count() external view returns (uint);

    function increment() external;
}

contract MyContract {
    function incrementCounter(address _counter) external {
        ICounter(_counter).increment();
    }

    function getCount(address _counter) external view returns (uint) {
        return ICounter(_counter).count();
    }
}

```

上面两个合约语法不一样，但是要做的功能都是一样的，第一个用interface来约束了Counter合约，可读性更高，is这个关键字意味着继承的意思。可以继承contract也可以继承interface。

# 用solidity与其他合约交互



对于address.call的这种形式来说，有两种，一种是call，另一种是delegatecall，delegatecall是callcode的bug修复版本，所以callcode一般是过时失效了。那么这两个调用具体有什么不同呢？

在Solidity中，有几种方法来委托合同之间的调用。一个部署的合约总是驻留在一个地址上，Solidity中的这个地址-对象提供了三种方法来调用其他合约。

call - 执行另一个合同的代码
delegatecall - 执行另一个合同的代码，但被callee合同有caller合同的状态（存储）。
callcode - (deprecated)

也可以像这样为一个调用提供gas和ETH代币。

someAddress.call.gas(1000000).value(1 ether)("register", "MyName")。
delegatecall方法是对callcode的错误修正，它没有保留msg.sender和msg.value，所以callcode已被废弃，将来会被删除。

值得注意的是，delegatecall涉及到调用合约的安全风险，因为callee合约可以访问/操纵调用合约的存储。由于EVM的限制，call和delegatecall都不可能收到来自callee合约的返回值。

当然，从一个给定的地址调用合约上的函数存在固有的安全风险，这种调用合约的方式破坏了 Solidity 的类型安全。因此，call、callcode和delegatecall应该只作为最后手段使用。一般都通过ABI的方式把地址强制转换类型去调用。

DELEGATECALL基本上是说，我是一个合同，我允许（委托）你对我的storage做任何你想做的事情。DELEGATECALL对caller合约（就是发起delegatecall的合约）来说是一个安全风险，它需要相信接收合约会善待caller合约的存储。

DELEGATECALL是一个新的操作码，是对CALLCODE的错误修复，CALLCODE没有保留msg.sender和msg.value。如果Alice调用Bob，后者对Charlie进行DELEGATECALL，那么DELEGATECALL中的msg.sender就是Alice（而如果使用CALLCODE，msg.sender就是Bob）。


- contract Alice  --call--> contract Bob (msg.sender = Alice) --delegatecall--> contract Charline (msg.sender = Alice)
- contract Alice  --call--> contract Bob (msg.sender = Alice)  --callcode--> contract Charline (msg.sender =Bob)

当D对E做CALL时，D的代码在E的上下文中运行：E的storage被使用。

当D对E做CALLCODE/deletegatecall时，E的代码在D的上下文中运行（所以E能访问D的storage）。因此，设想E的代码在D中。

```d
contract D {
  uint public n;
  address public sender;

  function callSetN(address _e, uint _n) {
    _e.call(bytes4(sha3("setN(uint256)")), _n); // E's storage is set, D is not modified 
  }

  function callcodeSetN(address _e, uint _n) {
    _e.callcode(bytes4(sha3("setN(uint256)")), _n); // D's storage is set, E is not modified 
  }

  function delegatecallSetN(address _e, uint _n) {
    _e.delegatecall(bytes4(sha3("setN(uint256)")), _n); // D's storage is set, E is not modified 
  }
}

contract E {
  uint public n;
  address public sender;

  function setN(uint _n) {
    n = _n;
    sender = msg.sender;
    // msg.sender is D if invoked by D's callcodeSetN. None of E's storage is updated
    // msg.sender is C if invoked by C.foo(). None of E's storage is updated

    // the value of "this" is D, when invoked by either D's callcodeSetN or C.foo()
  }
}

contract C {
    function foo(D _d, E _e, uint _n) {
        _d.delegatecallSetN(_e, _n);
    }
}
```

当D对E做CALLCODE时，E里面的msg.sender是D，如上面代码中的注释。

当账户C调用D，而D对E做DELEGATECALL时，E里面的msg.sender是C。也就是说，E的msg.sender和msg.value与D相同。