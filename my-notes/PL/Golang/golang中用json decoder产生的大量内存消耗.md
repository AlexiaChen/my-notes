因为项目中的RPC接口经过大量调用，通过pprof看到的结果就是消耗在了reflect.,mapassign 等由json decoder引起的问题。

![unknown](https://user-images.githubusercontent.com/8574915/117593903-870a0880-b16f-11eb-93a1-f22aea0a038d.png)

上图就是我本地复现的情况

经过google，发现/thanos也有这个[issue](https://github.com/thanos-io/thanos/issues/1160)，经过他们讨论被一个[临时的PR](https://github.com/thanos-io/thanos/pull/1173)缓解了，他们的svg调用图为以下，看起来跟我们的pprof结果很一致。

![](https://user-images.githubusercontent.com/2804025/47689031-29121700-dc3c-11e8-87c7-f0c78ee7135e.png)

最终的临时方案就是用json.Unmarshal 替换NewDecoder().decode。   这样会好一些，但是也只是缓解该现象。没有彻底解决，所以当时他们这个[issue](https://github.com/thanos-io/thanos/issues/448) 还是打开的。

也许reflect相关的GC问题比较难修复，go官方的issue也提到了，但是还没有close https://github.com/golang/go/issues/28783     静观其变吧。要不就是直接把json这个换成其他的格式，但是这与我们项目的的目的背道而驰，因为几乎所有区块链都支持json RPC。

最后还是发现一个中文社区的朋友也遇到我类似的问题: http://javagoo.com/go/mapassign.html, 他的解决方案是:

> 既然decodeMap消耗内存严重，我们就放弃使用map。Transfer rpc之前，先将map转成string，然后Judge再将string转成map。

所以他是设法绕过一些问题。我看也有一些项目是用protobuf等格式取代json的，但是我们的项目行不通，json无法取代。

当然这个问题如果有大量RPC访问，是会触发OOM的，json.decode还有RLP.decode都会消耗大量内存，以太坊就有很类似的两个issues(因为从pprof的结果上上看很一致)，最终经过一番调查都没有内存泄露，给出的建议是加入rate limiter:

-   https://github.com/ethereum/go-ethereum/issues/22567 
-  https://github.com/ethereum/go-ethereum/issues/22529

当然，还有一个就是尽量把json的缓存，还有其他缓存加上。