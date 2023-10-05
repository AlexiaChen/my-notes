
https://medium.com/mycrypto/understanding-event-logs-on-the-ethereum-blockchain-f4ae7ba50378


基本与合约中的emit出来的事件的参数一一对应

```solidity
_allowances[owner][spender] = amount;
emit Approval(owner, spender, amount);


function transferFrom(address from, address to, uint256 amount) public virtual returns (bool) {
        address spender = _msgSender();
        _spendAllowance(from, spender, amount);
        _transfer(from, to, amount);
        return true;
    }

```

```txt
Transfer(address,address,uint256)  -> Keccak256 -> 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef

Transfer (index_topic_1 address from, index_topic_2 address to, uint256 value)
[topic0] 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
[topic1] 0x000000000000000000000000443b7a096752119142499e3aa93730341ebebbdb
[topic2] 0x0000000000000000000000008ea35d1eb3458543babce041ca4eee5e0692c5d8

Approval(address,address,uint256) -> Keccak256 -> 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925

 Approval (index_topic_1 address owner, index_topic_2 address spender, uint256 value)
[topic0] 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925
[topic1] 0x0000000000000000000000005438e339403a596ad1c411c3dfb2d96dd88f3fbc
[topic2] 0x000000000000000000000000881d40237659c251811cec9c364ef91dc08d300c

```

topic0 其实就是事件签名的Keccak256 Hash结果，这个可以在以太坊get_logs里面拿到过滤后的event列表:  https://github.com/OpenBazaar/go-ethwallet/blob/31db816d691ce615e012a18be2ad6fc3c7cce856/wallet/erc20_wallet.go#L534

一个代码示例:

```go
package main

import (
        "context"
        "fmt"
        "log"
        "math/big"

        "github.com/ethereum/go-ethereum"
        "github.com/ethereum/go-ethereum/common"

        // "github.com/ethereum/go-ethereum/core/types"
        "github.com/ethereum/go-ethereum/ethclient"
)

func main() {

        fmt.Println("Starting Ethereum client")
        // Connect to the Ethereum network
        client, err := ethclient.Dial("https://eth-mainnet.g.alchemy.com/v2/FPZM_sa3ko9badX3vqcLsrPBNlth_PHJ")
        if err != nil {
                log.Fatal("sss", err)
        }

        fmt.Println("Connected to the Ethereum network")

        // ERC20 contract address
        contractAddress := common.HexToAddress("0xdac17f958d2ee523a2206206994597c13d831ec7")
        // Transaction hash
        txHash := common.HexToHash("0xd6f48c660e5363ab88fd5648f89b20b5f0fba02c5e774ed371d9bcb29a23488d")

        // Get the transaction details
        receipt, err := client.TransactionReceipt(context.Background(), txHash)
        if err != nil {
                log.Fatal(err)
        }

        // Check for Approval event

        approvalQuery := ethereum.FilterQuery{
                Addresses: []common.Address{contractAddress},
                Topics:    [][]common.Hash{{common.HexToHash("0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925")}},
                FromBlock: receipt.BlockNumber.Sub(receipt.BlockNumber, big.NewInt(1024)),
                ToBlock:   receipt.BlockNumber,
        }

        approvalLogs, err := client.FilterLogs(context.Background(), approvalQuery)
        if err != nil {
                log.Fatal(err)
        }

        if len(approvalLogs) > 0 {
                fmt.Println("Transfer was authorized by Approval event")

                for _, vLog := range approvalLogs {
                        fmt.Println("Approval event received")
                        fmt.Println("Owner:", vLog.Topics[1].Hex())
                        fmt.Println("Spender:", vLog.Topics[2].Hex())

                        // Amount
                        amount := new(big.Int)
                        amount.SetBytes(vLog.Data)
                        fmt.Println("Amount:", amount)
                }
        } else {
                fmt.Println("Transfer was not authorized by Approval event")
        }

}
```

