## 前言

我之前在PVSS的那个项目里面说到了那个多项式的承诺(commitment)不是pederseon承诺, 而是commit = g^r。承诺其实就是对某个信息m的绑定，以达到信息隐藏的目的，这样就可以不暴露出来，等到需要用的时候，在进行暴露。所以这时候你可以觉得Hash函数也可以用作承诺，但是相比pedersen commitment有更大的优势。我后面会说到。

## Pedersen Commitment

先来解释下什么是pedersen commitment吧。

下文中的sender，prover，commiter是同一个对象。 receiver， verfier也是一个对象。从这里也可以看出来，pedersen commitment后面会大量用在零知识证明领域。

- 1. prover 知道一个秘密消息m
- 2. prover随机生成一个随机秘密r
- 3. prover生成pedersen承诺c = C(m, r)。  这个承诺函数C具体我们现在不用管，后面会告诉你算法
- 4. prover公开承诺c
- 5. 然后prover接着公开m， r
- 6. verifier接收到c, m, r。然后计算并检查C(m, r) = c 等式是否成立。（前面5步一定要严格按顺序执行）

以上前5步骤中， prover不能随意修改m和r，否则是不行的，并且在第5步之前，承诺c不能包含秘密消息m的任何有关线索。

还有需要注意，这个随机秘密r，必须每次commit的时候，都要不一样。

pedersen commitment使用一个素阶群G_q, q是大素数，并且这个群需要离散对数下困难。还需要这个G_q中任意两个无关的生成元g和h， g和h不能有关系。因为是素阶群，所以任意元素都可以作为生成元，所以随便选好了。随机秘密r选在Z_q上。秘密消息m只要是它的子集就行，然后pedersen commitment就是 C(m, r) = (g^m) * (h^r)

## Pedersen Commitment的作用

承诺在密码学上相当于秘密地将m写在一个密封的、防篡改的、单独编号（的信封中，信封由写消息的人(Prover)保管。信封的内容不能更改（绑定属性），消息不能泄漏（隐藏属性）。在密码学带来的改进中，我们不需要检查信封是否真的密封了，而且信封可以通过网络传输。

下面我们通过一个例子来说明，有三个任务Alice， Bob， Valery 。

没有pedersen commitment的场景:

- Alice偷偷地在{0,1}这两个数中选择出秘密消息m，把它写在纸上，把它放进信封，封好，给Bob和Valery看，但自己保留了信封。
- Bob在{0,1}中宣布了一个他猜测的秘密消息m_b；他和Valery还不知道m 是否等于m_b，但是Alice知道猜测结果是否正确。
- Alice公开说她选择了秘密消息m，并把信封给了Valery。(Valery是公证人)
- Valery检查m是否等于m_b,在有必要的情况下才打开信封查看是否有存在m的纸张，然后接下来Bob才来验证

有pedersen commitment的场景:

- Alice偷偷地在{0,1}这两个数中选择出秘密消息m, 然后生成承诺c = C(m, r)。
- Bob在{0,1}中宣布了一个他猜测的秘密消息m_b；他和Valery还不知道m 是否等于m_b，但是Alice知道猜测结果是否正确。
- Alice公开说她选择了秘密消息m，和秘密随机数r
-  Valery检查m是否等于m_b,在有必要的情况下才通过C(m,r) 是否等于 c， 然后接下来Bob才来验证

这里用承诺代替了信封，因为承诺不可篡改，也不直接暴露秘密消息m。如果是信封，Bob就可以骗人，在看到信封内容后说我的m_b就是m啊，发生反悔。

Alice也不能通过秘密r的选取进行欺骗，因为C(0, r) = C(1, r)。 更不可能反悔秘密消息m。

这么来看，确实Commitment可以用Hash函数来做，但是为什么呢？ 

## pedersen commitment 和Hash-based commitment

那么pedersen commitment和Hash commitment到底有什么区别呢？貌似都可以完成承诺的功能啊，pedersen commitment具有以下优势:

-因为是在群上做的，所以有加法同态性质

举个例子，在椭圆曲线上的素阶群为例， 选择两个G点: P和Q点。

```
commit(m,r) := m*P+r*Q
```

以上公式是常规情况，那么我们再推广以下，因为素阶群的任意元素都可以作为生成元，所以，我们也可以选择n个P，P1到Pn对多个秘密消息m_1到m_n进行commit:

```
commit(m_1,m_2,..., m_n,r) := m_1*P1 + m_2*P2 + ... + m_n * Pn + r*Q
```

然后，最重要的来了，Pedersen Commitment具有加法同态性质:

```
commit(m_1 + m_2, r1 + r2) = (m_1+m_2)*P + (r1+r2)*Q = m_1*P + r1*Q + m_2*P + r_2*Q =  (m_1*P + r1*Q) + (m_2*P + r_2*Q) = commit(m_1, r1) + commit(m_2, r2)

```

也就是pedersen commitment算法由了加法同态性质.

Hash commitment是没有加法同态性质的，因为Hash函数的输出是杂乱无章的，

## 参考

- [What is a Pedersen commitment?](https://crypto.stackexchange.com/questions/64437/what-is-a-pedersen-commitment)
- [What are the pros and cons of Pedersen commitments vs hash-based commitments?](https://crypto.stackexchange.com/questions/54735/what-are-the-pros-and-cons-of-pedersen-commitments-vs-hash-based-commitments)
- [What is the reason of using Pedersen Commitment scheme over HMAC?](https://crypto.stackexchange.com/questions/60279/what-is-the-reason-of-using-pedersen-commitment-scheme-over-hmac)

