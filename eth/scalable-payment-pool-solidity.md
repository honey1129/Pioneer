> * 链接：https://medium.com/cardstack/scalable-payment-pools-in-solidity-d97e45fc7c5c 作者：[Hassan Abdel-Rahman](https://medium.com/@habdelra?source=post_page-----d97e45fc7c5c--------------------------------)
> * 译文出自：[登链翻译计划](https://github.com/lbc-team/Pioneer)
> * 译者：[影无双](https://learnblockchain.cn/people/58)
> * 本文永久链接：[learnblockchain.cn/article…](https://learnblockchain.cn/article/1)

# 利用Merkle树低成本实现可扩展支付池



我最近一直在研究一个有趣的问题：支付池（payment pool）。



支付池是一种通用机制，可用于模拟“一对多”或“多对多”支付通道。这个想法是，可以从各种来源将通证存放到池中，然后根据“规则”(在链上或链外实施)将池中的通证分配给许多接收者。本质上，我们的方法有能力就按大量小额付款汇总到一个结算中，从而节省了大量的 gas 。

考虑一个类似于Spotify的场景，在此案例中，每当有人听一首歌，就向音乐家支付版税。流媒体服务可以用大量通证为音乐家的支付池提供资金。然后，当人们播放音乐时，这些播放日志将在每个付款周期中汇总，汇总的付款金额将被反馈到支付池，从而以比在链上交易（每次听歌时都触发交易）少的 gas 消耗的方式来支付音乐家之间的版税。



## 先看看：基于数组的支付池

我们最初对这种链上支付池智能合约的构想非常简单。

在我们的Tally协议，协议通过“分析”矿工贡献的GPU周期来计算奖励，和流媒体场景很类似，开始的想法是支付池智能合约将从各种通证收集智能合约接收[ERC-20](https://learnblockchain.cn/tags/ERC20)通证，通过分析签名的应用程序使用日志和其他链上信号来确定支付池内的通证分配。 *分析矿工*将就支付池中的付款金额达成共识，并通过提交一系列收款人地址和付款金额的形式提交给支付池，这些地址和付款金额将写入支付池管理的帐本中。

此解决方案最明显的缺点是，支付池处理的是无限的收款人及其付款金额，这意味着这种交易可能会超出 gas 限值。这将需要支付池能监控 gas 预算，并跟踪收款人列表，以便如果超出 gas 预算时，以便可以在后续交易中继续执行。

我们进行了一些实验，在当前的区块 gas limit 下，可以支持 200多名收款人，再多就会超出 gas 限制，假设按 gas 价格为35 gwei ， gas 限额为10,000,000，需要耗费 0.35 个 ETH（当前大约 1000 块），相当于存储每个收款人支付的 gas 费是5 块。

显然，这种方法无法扩展。



## Merkle树

在寻找更好的方法时，我们受到[这篇以太坊研究文章](https://ethresear.ch/t/pooled-payments-scaling-solution-for-one-to-many-transactions/590)的启发

这个想法是，与其让支付池管理收款人及其付款金额的帐本，不如建立一个保存收款人及其付款金额的[Merkle树]([https://learnblockchain.cn/tags/Merkle%E6%A0%91](https://learnblockchain.cn/tags/Merkle树))，然后收款人通过提交Merkle证明来提取相应的金额。然后，Merkle证明将成为仅适用于收款人的密钥，该密钥可在支付池中解锁收款人的通证。

Merkle树方法的优点在于，我们只需要向支付池中写入32字节的Merkle根，并且可以存在Merkle树中的收款人数量没有上限。无论Merkle树代表多少收款人，我们都只需要为树写一个32字节的Merkle根：对于无数收款人， gas 费则可以分计。

我们中许多人都知道：[Merkle 树](https://blog.ethereum.org/2015/11/15/merkling-in-ethereum/)是一种新颖的二叉树结构体，它使我们能够轻松，廉价地确认树中是否确实存在节点。 Merkle树构成了以太坊的基础，并有助于以太坊节点无需区块链的完整历史即可验证区块的能力。

Merkle树最重要的方面是：

1. 每个节点是该节点的子级哈希值之和的哈希值
2. 根节点的哈希受树中每个节点的影响
3. 我们可以通过将节点的哈希值及其“叔叔”节点加在一起，以确定它们是否与根节点匹配，来确定树中是否存在节点。

![Merkle根](https://img.learnblockchain.cn/pics/20201103113757.png)


我们将关心的数据放在Merkle树的叶节点中。有许多可用的代码库可以执行此操作，给库中提供一个数组，库将对数组进行排序，并使用提供的已排序数组构成Merkle树的叶节点来构建Merkle树结构体。库会提供Merkle树的根，它也可以为任何节点提供证明，其中证明是该节点的哈希与叔叔们 hash列表，当与节点的哈希值加在一起时，就是默克尔根。

我们可以验证节点是否确实存在于Merkle树中的方法是添加带有其证明的节点，然后查看该结果是否等于根节点。OpenZepplin 有一个[Solidity 库](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/cryptography/MerkleProof.sol) 用来验证。

![1_3_TyiY19z9Gz0o5Qt3UEnw](https://img.learnblockchain.cn/pics/20201103115039.png)
<center>在此示例中，检查树中是否存在L2，我们通过在hash(L2)上加入哈希A和哈希B，来确认总和的哈希是否“根节点”的哈希值。</center>

## Merkle树支付池

我们如何在支付池中利用Merkle树？

这种方法利用了需要链上和链下机制的方法。为了生成Merkle树，我们可以使用链下程序(例如NodeJS模块)从收款人及其付款金额列表中构建Merkle树。采用这种方法，每个节点都是收款人地址及其付款金额的字符串连接。

考虑以下收款人清单：

![收款人清单](https://img.learnblockchain.cn/pics/20201103115624.png)

我们可以将此列表转换为如下所示的数组：

![列表](https://img.learnblockchain.cn/2020/09/28/16012645932286.jpg)

然后，我们可以从该列表构建Merkle树，合约所有者可以将Merkle根提交到支付池合约。而且，我们也可以在一些地方(例如IPFS)发布这些叶节点及其证明数据，以便收款人可以访问此数据。



得到如下所示的列表：![1_-lXXOgHSfYj2cvNVxrkYcw](https://img.learnblockchain.cn/pics/20201103121009.png)

<center>(请注意，这些并不是这些节点的实际Merkle证明，而是一些随机的十六进制来传达想法)</center>



然后，收款人可以使用金额和证明作为函数的参数来调用支付池合约上的函数，以便提取其通证。

思路是`paymentPool.withdraw()`函数将通过`msg.sender`及通证数量重建叶子节点。`withdraw`函数可以哈希该叶节点并将哈希的叶节点添加到证明(这是构成证明的哈希的十六进制表示)。如果哈希叶节点和提供的证明之和的哈希值等于合约所有者提交的Merkle根，那么`paymentPool.withdraw()`函数可以允许将通证从支付池转移到`msg.sender`。

此外，我们需要跟踪每个收款人的提款，以便` msg.sender`无法重复调用` paymentPool.withdraw()`函数。



因此，这种方法是一个好的开始。我们已经解锁了从支付池中支付收款人所需数量通证的功能，而不必产生大量的gas费，此外，这意味着我们支付池中收款人的数量和所需要的 gas 费解耦了。收款人的证明基本上就像钥匙一样，仅用于从收款人的地址发起的交易，可用于从池中解锁该收款人的通证。

但是我们仍然面临一些挑战。

1. 如果收款人只想提取一部分通证怎么办？
2. 如何表示链上某个地址可用的通证数量及证明？
3. 多个付款周期呢？ Merkle根更新后，我们可以使用旧的证明吗？



## 改进

为了解决上述挑战，我们在每个收款人的证明中添加了元数据，并在支付池中引入了“付款周期”的概念。

### 付款周期

在支付池智能合约中，我们跟踪每个Merkle根提交所描绘的付款周期。合约所有者向支付池提交Merkle根表示当前付款周期已结束，新的付款周期开始。

在支付池智能合约中，我们维护一个映射属性，该属性将付款周期映射到管理该付款周期中在该支付池的Merkle根。这样，当用证明调用`paymentPool.withdraw()`函数时，如果我们知道生成证明所依据的付款周期，则可以使用正确的Merkle根来验证证明。

这使收款人可以使用旧的证明来提取付款。同时，这确实意味着证明与特定数量的通证相关。你无法提取超过在叶子节点的哈希值对应的通证数量。

只要支付池跟踪每个收款人已提取多少通证，就可以确保从分配给该收款人的累计通证中减去已提取的通证数量。

### 证明元数据

要克服的另一个挑战是如何提取少于创建证明时的通证数量。此外，我们如何使用户更容易将证明与特定付款周期相关联，以便可以使用正确的Merkle根来验证提款请求？

针对这些挑战，我们引入了将元数据附加到证明本身的想法。我们可以将付款周期号和收款人可收到的累计通证数量(用于在Merkle树中生成收款人的付款叶子节点)合并到证明中。这样，收款人调用`paymentPool.withdraw()`函数指定提取最多金额，该金额不超过证明的通证数量。

简单吧？收款人正在调用` paymentPool.withdraw()`，其中包含所需的通证数量以及一个特殊的密钥，该密钥仅对他们有效，以从支付池中解锁这些通证。

这是它的工作方式：正如我在上面提到的，证明实际上只是一系列已被序列化为十六进制格式的姑妈及叔叔哈希的数组。为了将元数据包括在证明中，我们需要向该证明数组添加几个额外的项。



具体来说，要添加与Merkle根相对应的付款周期号(我们可以在将Merkle根提交给合约之前，在PaymentPool智能合约上调用`paymentPool.numPaymentCycles()`来获得该值)以及收款人可提取的累计通证数量。在`paymentPool.withdraw()`函数中，需要从证明中剥离元数据，以便`paymentPool.withdraw()`函数知道证明所涉及的支付周期以及通证的数量是此收款人Merkle树中叶节点的一部分。



这样`paymentPool.withdraw()`函数才能查找到正确的Merkle根用作证明，同样通过`msg.sender`及在出现在证明元数据中的通证数量来正确构造叶节点哈希。

出现在`paymentPool.withdraw(amount，proof)`中的金额(amount)是收款人希望从所证明的通证总数中提取的通证数量。

![](https://img.learnblockchain.cn/2020/09/28/16012646815076.jpg)
<center>图：Cardstack通过元数据验证的Merkle树实现的支付池</center>


这种方法还需要提供一个链上函数，允许任何人通过证明的收款人来查看可用于特定证明的通证数量。

## 重要注意事项

我曾在此解决方案中提到过几次，默克尔树需要跟踪通证的累计数量，这意味着收款人列表及其数量只能随时间增长-我们永远都不会看到收款人的累计数量在随后的付款周期中减少。



这是为什么呢？这是这种特定方法的细微差别：我们为每个付款周期构建的Merkle树需要反映收款人的累计付款金额，并且应在支付池中保留取款额的映射，其差额是可以允许收款人提供的任何有效证据提取的部分(当差额为负时，显然不允许提取)。



如果后续付款周期中的累计付款额减少了，则意味着计算可用提取的通证数量（已提取的金额与证明元数据中的累计总数之差）将是不正确的，并对可用余额产生影响导致他们无法提取所有通证。



该解决方案也很依赖于链下技术，尤其是需要发布收款人的证明(IPFS可能是不错的地坊)。你可能还希望发布收款人在付款周期中可以收到的通证数量，甚至可能需要提供dApp的链接，该链接可以显示支付池中证明可用的余额。



此外，值得注意的是，在本文中提到的所有解决方案，并未涉及如何确保支付池中的资金已全部到位，从而使收款人可以连续进行提款。在我们提供的代码示例中，我们确实确保支付池有足够的资金，然后再尝试将通证调用` payingPool.withdraw()`功能时将通证转移到收款人。这里可以想到的一种方法是当支付池余额下降到特定阈值以下时发出支付池通证余额警告事件。

# What’s Next(下一步是什么)

你可以在我们的[GitHub代码库](https://github.com/cardstack/merkle-tree-payment-pool)中找到代码(用于构建证明和元数据的合约和javascript库)，代码库中的README文件和测试在代码级别演示了如何利用这种方法。

如果你觉得此解决方案对你有用，欢迎在你自己的合约中使用它，改进它。 



------

本翻译由 [Cell Network](https://www.cellnetwork.io/?utm_souce=learnblockchain) 赞助支持。


