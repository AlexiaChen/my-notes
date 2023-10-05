这篇主要讲以太坊合并后(Post-Merge)的状态转换函数(STF)

世界计算机是一个去中心化的状态机，让我们来看看状态转换函数

听起来像废话？文章将解释每次validator收到新区块时发生的事情

![[Pasted image 20221009144608.png]]

以太坊， 最准确的描述是分布式状态机--一种有两个主要组成部分的计算模型。

- 状态，系统在某一时刻的配置
- 交易，改变状态的一组指令

一个状态是一个静态的时间快照，交易是改变状态的指令，状态转换函数是将交易应用于状态的机制。

输入：当前状态，区块
输出：新的状态

![[Pasted image 20221009144820.png]]

共识规范：`state_transition`

首先，处理所有没有区块的empty slot（`process_slots`）。检查区块的签名（`verify_block_signature`），如果有效，通过执行底层交易来处理区块（`process_block`）。

![[Pasted image 20221009145044.png]]

有时，一个slot过去了，却没有一个区块被提出来；也许区块提出者(block proposer)不在线，或者网络放弃了这个区块。如果需要的话，状态转换函数在empty slot中移动，并触发epoch的改变。

输入：当前状态、slot
没有输出

![[Pasted image 20221009145207.png]]

共识规范：`process_slots`

当当前slot小于预定slot时，向前推进（`process_slot`）。如果下一个slot将是一个新的epoch，则执行与共识相关的进程，为下一个epoch做准备（`process_epoch`）

![[Pasted image 20221009145333.png]]
 一个epoch是32个slots/blocks。当`process_slots`通过这个边界时validator必须处理许多基于共识的和内务的职责。

输入：当前状态
没有输出

![[Pasted image 20221009145448.png]]

共识规范： `process_epoch` ^49a5f2

下面是共识规范，你可以看到它非常密集。

![[Pasted image 20221009145638.png]]

即使在这个层面上，事情也是非常密集的。我将添加一些资源以进一步深入研究。

![[Pasted image 20221009145721.png]]

> 从上面来看，感觉你发出去的交易的确认时间，可能最糟糕的情况是需要32个slots，也就是6分多钟才可以被确认不被回滚。运气好的话，那就是刚好在epoch切换的边界上，也就是12秒确认。不过相比PoW来说，确认速度确实更快了，只是确认的间隔区块变长了，PoW确认几乎可以认为是间隔1个区块就可以了。

请深入研究以下函数
- `process_justification_and_finalization`
- `process_inactivity_updates`
- `process_rewards_and_penalties`
- `process_slashings`
- `process_slashings_reset`
- `process_participation_flag_updates`

如果有疑问，请对照 [[ETH之Casper FFG共识]] 研究

深入研究:

- `process_randao_mixes_reset`

请参考 [[ETH之随机性和RANDAO]]

最后，我们到了好的部分；是时候处理这个区块了! 再一次检查标题，validator将把区块（以及它所携带的交易）送入EVM。它以一些共识的内务处理结束。

输入：当前状态，区块
没有输出

![[Pasted image 20221009150348.png]]

共识规范：`process_block`

在核心部分，validator执行最终检查（`process_block_header`），并将区块送入EVM（`process_execution_payload`）。

![[Pasted image 20221009150552.png]]
共识规范： `process_execution_payload`

这个函数的第一块内容只是更多的检查。validator将交易传递给EVM的时刻是`execution_engine.notify_new_payload(payload)` 这个函数做的。

![[Pasted image 20221009150701.png]]

最后，validator回到其共识职责。它添加到RANDAO（`process_randao`），对其存款合约的观点进行投票（`process_eth1_data`），管理剔除slashing、证明attestation等（`process_operations`），确认同步委员会(the sync committee)（`process_sync_aggregate`）。

![[Pasted image 20221009150852.png]]

总结:

```python
def ethereum_state_transition: 
	process_slots 
	if new_block = new_epoch: 
		process_epoch 
		process_blocks 
	
for：
	ethereum_state_transition() 

```

#### 一些其他资源

- [Upgrading Ethereum | 3.5 Beacon Chain State Transition Function (eth2book.info)](https://eth2book.info/altair/part3/transition)
- [ethereum/consensus-specs: Ethereum Proof-of-Stake Consensus Specifications (github.com)](https://github.com/ethereum/consensus-specs)

