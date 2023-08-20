# Golang的并发模式

[5 concurrency patterns in Golang - DEV Community](https://dev.to/vietmle_/5-concurrency-patterns-in-golang-dm4?utm_source=dormosheio&utm_campaign=dormosheio)

[Visualizing Concurrency in Go · divan's blog](https://divan.dev/posts/go_concurrency_visualize/)



## Golang内存泄露

[Plugging leaks in Go memory management | Cossack Labs](https://www.cossacklabs.com/blog/investigating-go-memory-leaks/)

## Golang interface

[How To Use Go Interfaces - Bigger on the Inside (chewxy.com)](https://blog.chewxy.com/2018/03/18/golang-interfaces/)

## 有趣的实践例子

把5W行java移植到golang的经验
[Lessons learned porting 50k loc from Java to Go (kowalczyk.info)](https://blog.kowalczyk.info/article/19f2fe97f06a47c3b1f118fd06851fad/lessons-learned-porting-50k-loc-from-java-to-go.html)




#### Channel

[Golang 的 Channel 是一种免费的无锁实现吗？ | 卡瓦邦噶！ (kawabangga.com)](https://www.kawabangga.com/posts/4777)


### 内存对齐

[Memory Alignment. Today we're going to talk about memory… | by 👨‍💻 Dillen Meijboom | Aug, 2022 | Medium](https://medium.com/@dillen.dev/memory-alignment-8760a5dfc4dc)


### go pprof的基本用法

记录的比较凌乱


seconds参数是60秒钟的间隔观察

go tool pprof http://localhost:6060/debug/pprof/profile?seconds=60

go tool pprof http://207.244.236.5:6060/debug/pprof/heap
    -inuse_space：分析应用程序的常驻内存占用情况
    -alloc_objects：分析应用程序的内存临时分配情况

go tool pprof -inuse_space -cum -svg http://13.57.246.198:6060/debug/pprof/heap > heap_inuse.svg
 go tool pprof --alloc_space http://localhost:6060/debug/pprof/heap

##### 分析goroutine

 go tool pprof  http://127.0.0.1:6060/debug/pprof/goroutine

go tool pprof http://localhost:6060/debug/pprof/block

go tool pprof http://localhost:6060/debug/pprof/mutex

### golang的安装

```bash
wget https://studygolang.com/dl/golang/go1.19.4.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.14.3.linux-amd64.tar.gz
echo "export PATH=$PATH:/usr/local/go/bin" >> ~/.profile
go version
go env -w GOPROXY=https://goproxy.cn
```