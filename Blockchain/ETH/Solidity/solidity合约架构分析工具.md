
# Surya

Surya是ConsenSys创建的一个分析solidity只能合约的工具，可以分析合约的结构，调用图，调用trace  [ConsenSys/surya: A set of utilities for exploring Solidity contracts (github.com)](https://github.com/ConsenSys/surya)

## 安装

```bash
sudo apt install graphviz
npm install -g surya
```

在VSCode上可以用solidity visual developer这个插件，这个插件调用了surya这个命令。但是很遗憾，由于VSCode架构，在WSL2上使用不了这个插件，即使安装上去，一些功能也不正常，必须在宿主机上。下面只列出常用功能

## 调用图

```bash
surya graph contracts/**/*.sol | dot -Tpng > MyContract.png
```
以上就是分析所有合约的调用图，包括目录的子目录下的sol文件

![[Pasted image 20221019161530.png]]

有一个新的标志（-s/-simple），它使命令图只显示合同调用图，而不是函数调用图。这对high-level的分析是非常有用的。

接受的标志

- -i/--import - 通过获取正确的文件来自动解决所有导入问题。
- -c/--content - 允许将文件内容作为参数传入，而不是文件路径。
- -s/--simple - 只显示合约之间的调用，不指定函数。
- -m/--修改器 - 允许打印从函数到修改器的边缘（当后者在函数定义中被调用时）。
- -l/--libraries - 当使用 "Using ... for "语法时，禁止打印从函数到库的边际（以防止像SafeMath这样的库污染调用图）。

```bash
surya graph -i -s -l contracts/**/*.sol | dot -Tpng > MyContract.png
```

## ftrace

ftrace命令输出一个树状的函数调用跟踪，它源于定义的 "CONTRACT::FUNCTION"，并遍历 "所有| internal | external "类型的调用。exteranl调用被标记为橙色，internal调用不着色。

```bash
surya ftrace APMRegistry::_newRepo all MyContract.sol
```

![[Pasted image 20221019161723.png]]

接受的标志

-i/--import - 通过获取正确的文件来自动解决所有导入问题。
-c/--content - 允许将文件内容作为参数传入，而不是文件路径。
-j/--json - 返回一个JSON对象，而不是一个树状的函数调用跟踪（当Surya被用作另一个包的依赖时，这一点非常有用）。

## flatten

flatten命令输出源代码的扁平化版本，所有导入语句都被相应的源代码所取代。引用已经被导入的文件的导入语句，将被简单地注释掉。

```bash
surya flatten MyContract.sol
```

## describe

describe命令显示了所提供的文件中的合约和方法的摘要。

```bash
surya describe *.sol
```

![[Pasted image 20221019162236.png]]

接受的标志

- -i/--import - 通过获取正确的文件来自动解决所有导入问题。
- -c/--content - 允许将文件内容作为参数传入，而不是文件路径。

## 继承

inheritance 继承命令输出一个DOT格式的继承树图。对于Windows机器，`>`应该用 `-o` 代替。

```bash
surya inheritance MyContract.sol | dot -Tpng > MyContract.png
```

![[Pasted image 20221019162527.png]]
接受的标志

- -i/--import - 通过获取正确的文件来自动解决所有导入问题。
- -c/--content - 允许将文件内容作为参数传入，而不是文件路径。

## 依赖

依赖关系命令输出一个给定合约的继承图的c3线性化。合约将从继承最多的开始列出，也就是说，如果同一个函数在多个合约中被定义，solidity编译器将使用最先列出的合约中的定义。

```bash
surya dependencies Exchange Exchange.sol
```

接受的标志

-i/--import - 通过获取正确的文件来自动解决所有导入问题。
-c/--content - 允许将文件内容作为参数传入，而不是文件路径。


## 生成Markd报告

mdreport命令创建了一个Markdown描述报告，其中的表格包含了关于系统的文件、合同及其功能的信息。和describe很像，但输出到一个格式良好的Markdown文件。

```bash
surya mdreport report_outfile.md MyContract.sol

```

# Solgraph

这个相当于Surya的graph功能，[raineorshine/solgraph: Visualize Solidity control flow for smart contract security analysis. ⇆ (github.com)](https://github.com/raineorshine/solgraph)

## 安装

```bash
sudo apt install graphviz
npm install -g solgraph
# or
sudo npm install -g solgraph --unsafe-perm=true --allow-root
```

## 使用

```bash
solgraph MyContract.sol > MyContract.dot
dot -Tpng MyContract.dot -o MyContract.png
```

# sol2uml

这个工具可以生成solidity合约的UML类图，并且合约的storage布局图也可以生成 [naddison36/sol2uml: Solidity contract visualisation tool (github.com)](https://github.com/naddison36/sol2uml)  这个storage就是有slot那种概念，熟悉ETH storage的就知道了。

## 安装并升级

```bash
npm link sol2uml --only=production
npm upgrade sol2uml -g
npm ls sol2uml -g
```
## 使用UML类图

要生成contracts文件夹及其子文件夹下的所有合同的示意图  

```bash
sol2uml class ./contracts
```

从Etherscan上经过验证的源代码生成EtherDelta的合约图。输出将是工作文件夹中的svg文件`0x8d12A197cB00D4747a1fe03395095ce2A5CC6819.svg`。  

```bash
sol2uml class 0x8d12A197cB00D4747a1fe03395095ce2A5CC6819  
```

从Etherscan Ropsten上经过验证的源代码生成EtherDelta的合约图。输出将是工作文件夹中的一个svg文件`0xa19833bd291b66aB0E17b9C6d46D2Ec5fEC15190.svg`。  

```bash
sol2uml class 0xa19833bd291b66aB0E17b9C6d46D2Ec5fEC15190 -n ropsten
```
  
要在某个根文件夹下生成所有Solidity文件，并将svg文件输出到一个特定的位置  

```bash
sol2uml class path/to/contracts/root/folder -o ./outputFile.svg  
```


要在一个单一的Solidity文件中生成所有合同的图表，将png格式的输出文件放到输出文件./someFile.png中  

```bash
sol2uml class path/to/contracts/root/folder/solidity/file.sol -f png -o ./someFile.png 
```

 
生成contracts和 node_modules/@openzeppelin 文件夹下所有 Solidity 文件的图表。输出将是工作文件夹中的 contracts.svg 和 contracts.png 文件。  

```bash
sol2uml class ./contracts,node_modules/@openzeppelin -f all -v  
```


生成一个工作文件夹下所有Solidity文件的图表，忽略了solparse、@solidity-parser和ethlint文件夹下的文件，这些文件将在node_modules文件夹下。  

```bash
sol2uml class -i solparse,@solidity-parser,ethlint  
```

  


