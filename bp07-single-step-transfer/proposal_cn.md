
# BRC20单步transfer方案 v1

### 动机

在brc20协议中铸造transfer铭文到地址A时，将锁定A的一定的available余额在这个transfer铭文中，称为transferable balance。后续这个铭文第一次被转移给B时，这个transferable balance会变成B的available余额。

因为给地址A铸造一个transfer铭文不需要经过A的同意，在没有经过A同意的情况下就可以将他的available余额锁定在铭文中，这构成了一种攻击。对于普通用户来说这一般不是严重的问题，因为这种攻击并没有导致用户的overall余额减少，并且用户可以用比攻击者更低的成本将available余额恢复。但对于需要频繁和大量发送brc20的实体来说，这种攻击方法可能带来很大的困扰和链上交易费用损失。

我们希望在两个方面对brc20的转账机制有所改进：

1. 阻止任意在他人地址上铸造transfer铭文
2. 实现单步transfer，降低转账成本

### 单步transfer机制

在铸造任何铭文的交易脚本中几乎总会附带一个公钥，并且要求签名，主要目的是防止其他人替换铸造交易。如下：

```
OP_PUSHBYTES_32 9e2849b90a2353691fccedd467215c88eec89a5d0dcf468e6cf37abed344d746
OP_CHECKSIG
OP_FALSE
OP_IF
OP_PUSHBYTES_3 6f7264
...
OP_ENDIF
```

这个在taproot脚本分支中的公钥签名校验是一种标准的schnorr签名，它已经被广泛应用在各种功能之中（比如现在的inscription，相对时间锁，闪电网络的Hash时间锁等），预计还会在将来各种新开发的taproot合约中继续作为基本签名校验方法。 它被越来越多的基础设施工具支持是必然的，而且和用户原始地址类型无关。
我们可以简单复用这个机制，把这个公钥对应的地址作为铸造brc20 transfer时锁定的可用余额来源，就能有效避免无授权的transfer铸造。
在legacy transfer中，锁定的transferable余额属于铭文的持有地址，这个地址同时也是可用余额的来源地址。但在公钥已经可以指定余额来源地址而不需要通过铭文持有地址的情况下，铭文持有地址就可以作为新的接收地址，这产生了一种在铸造时就单步转移余额的新方式。如下图所示：

![Single Step Transfer Function](./single-step.png)

这种single-step transfer和legacy transfer的唯一差别是在铸造时选择可用余额来源地址方式不同。对于一个已经铸造的transfer铭文，行为不再有差别。
所有地址默认可以使用legacy transfer或单步transfer，但一个公钥首次铸造有效单步transfer铭文后，这个公钥对应的地址上将永久禁用legacy transfer。

新机制一次解决了两个问题。接下来我们需要讨论一些细节。

### 公钥签名和对应的地址

Ordinals协议并不要求铭文是否包含公钥和签名，由于常见的 `OP_PUSHBYTES_32 + OP_CHECKSIG` 所附带的公钥签名可以失败，可以被绕过从而伪造任何公钥，我们需要使用更为严格的 `OP_CHECKSIGVERIFY` 来校验签名。同时需要明确指定公钥对应的地址类型。参考bitcoin的3种主流地址在ScriptPK中都使用单字节前缀的方法区分地址类型，我们也用单字节来描述几种可以从公钥转换的8种地址类型。
我们定义单步transfer铭文脚本必须以 `OP_PUSHBYTES_32 + OP_CHECKSIGVERIFY + OP_N` 格式开头。其中N指代地址类型：

1. P2TR 空脚本路径
2. P2WPKH 压缩偶数公钥
3. P2WPKH 压缩奇数公钥
4. P2PKH 压缩偶数公钥
5. P2PKH 压缩奇数公钥 
6. P2SH-P2WPKH： Nested SegWit 压缩偶数公钥
7. P2SH-P2WPKH： Nested SegWit 压缩奇数公钥
8. P2TR 密钥路径

比如最简单的空脚本路径的taproot地址，脚本格式如下：

```
OP_PUSHBYTES_32 9e2849b90a2353691fccedd467215c88eec89a5d0dcf468e6cf37abed344d746
OP_CHECKSIGVERIFY
OP_1
OP_FALSE
OP_IF
...
OP_ENDIF
```

Inscriptions that do not belong to these 8 standard formats are not single-step transfer inscriptions.
After inscribing a single-step transfer,  the specified type address will disable legacy transfer, no mater the available balance sufficent or not.

### Vendicate铭文支持

brc20铭文不接受Cursed铭文和Vendicate铭文，即不接受使用POINTER tag在同一笔交易中铸造多个铭文。我们可以局部放开这个规则，即单步transfer允许Vendicate铭文。
允许在单个交易中铸造多个单步transfer铭文非常重要。
这样我们就可以在单笔交易中铸造多个单步transfer铭文，并指向多个接收者来批量发送brc20余额。尤其是我们可以在花费一个已确认的transfer铭文的同时铸造多个单步transfer给接收者，这样的一对多甚至是多对多的发送和现在一对一发送transfer一样安全。就像现在的brc20市场卖单机制，这种安全性是由交易结构来保证的，而不会受交易是否确认的影响。
这不仅可以明显降低高频转账的费用，更重要的是它加强了brc20的UTXO属性。构建brc20的完整的闪电网络变得可能。
The Lightning Network's channel opening transaction is a transaction with a A+B‘s 2/2 multi-signature address output.
The channel closing transaction is generally a transaction that spends this 2/2 multi-signature utxo and comes with 2 outputs. The script of these 2 outputs have the function of allocating balances to A and B.

To implement a brc20-based lightning network, we don't need to change the channel opening transaction, nor do we need to change the output part of the channel closing transaction. We only need A and B to send the BRC20 balance to this 2/2 multi-signature address respectively, and this balance will enter the channel. Then add a commit transaction of a single-step transfer between the channel opening and closing transactions, that is, it will spend the 2/2 multi-signature of the channel to a commit address output, and then spend the utxo of this commit transaction in the channel closing transaction, and inscribe 2 single-step transfers to 2 outputs at the same time. Due to the utxo characteristics of single-step transfer, it can be well integrated into the functions of the native lightning network, directly inheriting the security of the lightning network without any additional game design. When the channel needs to be closed, just broadcast this commit transaction and the subsequent close transaction.
Even more interesting is that we can adjust the capacity of the brc20 channel of the lightning network without closing and reopening the channel again, which is impossible in the original lightning network. Anyone can expand the channel capacity by continuing to send brc20 to this channel's multi-signature address, or A B can inscribe another transfer to reduce the channel capacity. 

### 升级激活

We can first enable single-step transfer on 5 characters, then enable it in 4-character brc20. This is much safer. And when we need to enable 4 characters, the workload is very small.

### 其他

由于可以显著减少ordinals原始数据量，我们另外建议将来deploy/transfer和mint也可以允许使用Vendicate铭文。
同时可以在mint铭文中也支持附带标准签名，这种mint铭文通过签名的公钥而不是持有者来确定余额归属。这样可以将铭文直接丢弃（或锁定在opreturn中），不占用用户的utxo。

### 致谢

感谢domo在确定单步transfer方案过程中提出了许多改进反馈意见，感谢seesharp指出需要处理铭文utxo膨胀的思路。

### Indexer规则

这里列出索引规则

- 如果将单步transfer直接铸造到op_return,则视为burn
- 如果将单步transfer直接铸造到module的接收地址,则支持充值到module
- Module 的withdraw暂时不支持带签名单步操作
- mint支持带签名的单步操作
- 单步transfer支持批量铸造（直接支持vendicate）
- mint支持vendicate但不支持单笔交易内的多次mint（多次mint的第一次是合法的，仅通过铭文id的i0来过滤）
- deploy也正常支持vendicate，但不支持单笔交易内多次deploy（多次deploy的第一次是合法的，仅通过铭文id的i0来判断）
- 暂时不区分地址是否使用过单步transfer，legacy transfer照常支持使用
