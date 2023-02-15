## 目录



## Kademlia

### 论文摘录备忘

[Petar Maymounkov](https://pdos.csail.mit.edu/~petar/) & [David Mazières](https://www.scs.stanford.edu/~dm). **Kademlia: A Peer-to-Peer Information System Based on the XOR Metric**. Oct, 2002



The Kademlia protocol ensures that every node knows of at least one node in each of its subtrees, if that subtree contains a node. With this guarantee, any node can locate any other node by its ID. 

In a fully-populated binary tree of 160-bit IDs, the magnitude of distance between two IDs is the height of the smallest subtree containing them both. When a tree is not fully populated, the closest leaf to an ID x is the leaf whose ID shares the longest common prefix of x.

XOR is unidirectional. For any given point `x` and distance `D > 0`  there is exactly one point `y` such that `d(x, y) = D`. Unidirectionality ensures that all lookups for the same key converge along the same path, regardless of the originating node. Thus, caching `<key, value>` pairs along the lookup path alleviates hot spots. 

For each `0 <= i < 160` , every node keeps a list of `<IP address, UDP port, Node ID>` triples for nodes of distance between 2<sup>i</sup> and 2<sup>i+1</sup> from itself. we call these lists `k-bucets`.

Each k-bucket is kept sorted by time last seen----least-recently seen node at the head, most-recently seen at  the tail. For small values of `i`, the k-buckets will generally be empty(as no appropriate nodes will exist). For large one, the lists can grow up to size `k`, where `k` is a system-wide replication parameter.

`k` is chosen such that any given `k` nodes are very unlikely to fail within an hour of each other(for example `k` = 20). 

 One cannot flush nodes' routing state by flooding the system with new nodes(DoS attacks). Kademlia nodes will only insert the new nodes in the k-buckets when old nodes leave the system.

`FIND_NODE` takes a 160-bit ID as an argument. The recipient of the RPC returns `<IP address, UDP port, Node ID>` triples for the `k` nodes it knows about closest to the target ID. These triples can come from a single k-bucket, or they may come from multiple k-buckets if the closest k-bucket is not full. In any case, the RPC recipient must return `k` items (unless there are fewer than `k` nodes in all its k-buckets combined, in which case it returns every node it knows about).

`FIND_VALUE` behaves like `FIND_NODE`, returning `<IP address, UDP port, Node ID>` triples--- with one exception. If the RPC recipient has received a `STORE` PRC for the key, it just returns the stored value.

the most important procedure a Kademlia participant must perform is to locate the `k` closest nodes to some given node ID, we call this procedure a *node lookup*.

### lookup流程详解

*node lookup*的流程梳理，主要通过几个图来体现。

我们的场景是：当前节点有自己的节点ID，有自己的k-bucket，有自己的并发发送窗口。在收到一个`FIND_NODE(ID)` 请求后，需要通过自己的处理，向请求方返回k个节点信息，这k个节点是当前节点尽力所能找到的距离目标节点最近的k个节点。见下图：

<img src="https://github.com/zwGithubStation/zwKade/blob/main/img-folder/find_node_1.png" width="1047px" height="682px">

首先，执行node lookup的节点从自己的k-bucket中选取出距离目标节点最近的 `α` 个节点，通过并行窗口向这些节点发送`FIND_NODE(ID)` 请求。见下图：

<img src="https://github.com/zwGithubStation/zwKade/blob/main/img-folder/find_node_2.png" width="1047px" height="719px">

发送会出现各种结果。

如果向节点`C`发送的`FIND_NODE(ID)`请求得到了响应(收到了k个节点信息)，我们会更新当前的FIND_NODE结果集合 `R` 。将最近的k个节点保留至 `R` 中，同时标记 `R` 中各个节点是否已成功访问过。这隐含着如下的设定：

- 已经标记访问成功的节点也可能因为不再排前k个而被踢出 `R`
- 在 `R` 中的未标记访问节点，可能因为对其访问失败(超时或返回失败) 而被踢出 `R`
- 超时后也可能收到迟来的`FIND_NODE(ID)` 响应，也正常处理
- 并不需要全部 `α` 个请求都返回才触发 `R` 的更新，每一个`FIND_NODE(ID)` 相应都可以触发

各种情况可以详见下图：

<img src="https://github.com/zwGithubStation/zwKade/blob/main/img-folder/find_node_3.png" width="1047px" height="719px">

最终这一递归式的查询最终成功的条件是：当前看到的前k近的节点，都已经成功得到了它们的 `FIND_NODE(ID)` 响应。这时我们将这k个节点的信息返回，针对本节点的 `FIND_NODE(ID)` 成功。详见下图： 

<img src="https://github.com/zwGithubStation/zwKade/blob/main/img-folder/find_node_4.png" width="1047px" height="719px">

其他RPC操作的内部流程都与 `FIND_NODE` 的流程有着类似的思想：

- 在节点上执行`STORE(key, value)` 操作，节点也会先找到k个距离key值最近的节点，向它们发送`STORE(key, value)` RPC。此外，这些节点还会按照同样的方式再次发出 `STORE(key, value)` RPC, 来保证数据在网络中的持久化。
- `FIND_VALUE(key)` 是与 `FIND_NODE` 近似的操作，如果当前节点不保有key所对应数据，它就会找距离key前k近的节点。但这一过程并不使用`FIND_NODE(key)` 而是 `FIND_VALUE(key)`，该操作在某一个节点返回key值所对应的数据时立即结束并返回数据。

### 优化点

#### 数据缓存策略

为了有效**缓存**的目的，当一个`FIND_VALUE(key)` 成功后，`FIND_VALUE(key)` 的发起节点会请求把`<key,value>` 数据项存储在距离key值最近的节点上(该节点显然不是value所在的节点)。因为Kademlia的拓扑结构具有线路上的单向性(unidirectionality)，使用这种cache可以加速 `FIND_VALUE(key)`。但如果一个key被访问太过频繁，最终可能导致太多的节点存有一份该数据。为了避免这种"过量缓存"，我们对于任意节点 `N` 上存储的 `<key,value>` 有一个过期值设定：距离key值最近的节点为 `K` ，**`N` 上存储的 `<key,value>` 的过期值 应该与 `K` 和 `N` 之间的节点数目 存在指数级反比例的关系**(距离越近，过期值越长，反之越短)。

鉴于Kademlia多用于**文件共享**，协议要求数据项的发布者应该每24小时重发一次数据，因为数据集超过24小时就会过期，来维持网络上没有太多冗余陈旧数据。对于诸如数字证书或者加密哈希的场景，则可以设置更长的数据过期阈值。

节点间频繁的请求流转通常可以让bucket保持有效的更新，但如果某一ID区间的节点没有接收过什么访问，我们也要有定期的bucket信息有效性检测：如果某一bucket一小时内都没有发生过*Node lookup*，就需要对该bucket进行刷新(refresh)操作，该操作会在bucket中随机选取一个节点ID，**执行FIND_NODE(ID)操作**。

#### 加入网路

如果一个节点`u`要**加入网络**，那它至少要知道一个已经存在于网络中的节点`w`。`u` 将 `w` 的信息加入自己相应的某一层bucket中，然后执行`FIND_NODE(u.ID)` 。最终`u` 将会更新自己的各级bucket。在 `u` 进行信息扩充时， 其他节点的k-bucket也在必要的情形下记录了节点 `u` 的存在。

#### 高效的数据项重发策略

为了确保`<key, value>` 数据项的持久化(persistence), 节点必须周期性的重新发布数据(republish keys)。否则，在两种情况下，可能会发生*lookup*操作的失败：

- 节点发布数据时，作为其`STORE` 指令目标的最初`k` 个节点，它们都存在离开网络的可能
- 新进入网络的节点，其节点ID与一些已经持久化的数据项key值的距离，可能相较于之前发布这些数据项的节点更加近

这都要求数据项的再次发布，以最大化满足**当前距离该数据项最近的k个节点上都保有该数据项。**

##### 策略1：周期性FIND_NODE(key)然后重新发布

针对可能的节点离网情况，Kademlia要求每一小时重新发布一次`<key, value>` 。最简单的实现可能会造成大量不必要的转发：

- `<key, value>` 可能存在于k个节点上， 每个节点周期性发送 `STORE(key, value)` 操作给当前其测算的离 `<key, value>` 最近的k-1个节点，形成k<sup>2</sup>规模的消息量，但实际要维护的最近的节点规模量级应该总是k个(没有大范围新节点入网的情况下)

- 下一小时要进行重新发布的节点规模可能大于k，k<sup>2</sup> 转发规模会相应的平方级增大

针对此的一个优化是，收到`STORE(key, value)` 的节点默认周围的k-1个节点也收到了重新发布的STORE消息，其在下一个小时内不会再做重发。

##### 策略2：周期性更新k-bucket，直接使用k-bucket进行重新发布

将策略1所需的针对各个key的`FIND_NODE` 操作，分摊至周期性的k-bucket更新上，每次的重新发布，都直接信任当前的k-bucket信息，直接在k-bucket种选取离key值前k近的节点进行`STORE` 发布。

##### 其他优化

为避免大量冗余的 `STORE(key, value)` 操作，`STORE` 操作可以有趋近性，也即一个节点 `X` 只有在如下的情况下，才会将 `STORE(key, value)` 操作发给节点 `Y` :

Distance(Y, key) > Distance(X, key)

### 算法证明

略，见论文文本。

### 系统优化

#### buckt更新策略优化

因为每个bucket容量有限，我们希望有效的LRU检查和淘汰策略，保证bucket信息的有效性。

针对bucket已满，有新的候选节点出现时的替换策略，Kademlia倾向于在需要给这些新节点发送消息时再进行替换核对。

当bucket已满且有新的节点 `Y` 向当前节点 `X` 发送消息时，`Y` 的信息先暂存在一个**缓存**中；当 `X` 针对bucket中的节点发起诸如 `FIND_NODE` 的请求时，**无响应(unresponsive)**的节点就可以被缓存中的新节点替换。缓存中的节点按照存储时间由近至远的顺序给与优先级，优先级高的先可以被替换至bucket中。

因为Kademlia使用UDP传输数据，**无响应**很可能意味着网络阻塞丢包，因此为避免网络**指数级增加退让间隔(exponentially increasing backoff interval)**, 不向无响应的节点再次发送其他的RPC请求(鉴于*lookup*操作只要有一个节点返回结果就可以继续进行，这样的停止发送也是可以接受的)。

当bucket中的节点连续5次无响应，可以被视为**失效(stale)**。当k-bucket没有满或者缓存为空时，Kademlia仅仅标记该节点为失效，而不是直接移除出bucket。

#### lookup操作加速

为了减少 *lookup* 操作可能的“**跳数**”(ID为160-bit 范围的整数值，每次lookup至少确认1bit的信息，期望跳数规模为**log<sub>2</sub>n**)规模，需要增加每次查找路由表的大小。我们期望通过将原有的k-bucket进行**每b个bucket合并为一个bucket**，将跳数规模由**log<sub>2</sub>n**缩减至**log<sub>2<sup>b</sup></sub>n**。

基于XOR操作的k-bucket划分，**在b>1时仍保有原来的划分策略，不引入额外的复杂度**，这是Kademlia相较于其他路由算法的优势之处。

## S/Kademlia

Baumgart, I., & Mies, S. **S/Kademlia : A practicable approach towards secure key-based routing S / Kademlia : A Practicable Approach Towards Secure Key-Based Routing.** June, 2014 



### 攻击类型

#### 针对底层网路(underlying network)的攻击

一般假设底层网络对于运行在其上的数据传输(overlay layer)不提供任何的安全防护措施。

攻击者可以获取并修改(overhear or modify)数据包(arbitrary data packages)。此外某些节点可以假造IP地址(spoof IP address)，数据包本身的可信度也存疑.

针对底层网络的攻击可以造成运行在其上的服务失效(denial of service)。

#### 针对信息路由的攻击

##### 日蚀攻击(Eclipse attack)

通过在网络中放置恶意节点，导致若干节点被其劫持，无法正常运行。在Kademlia中路由寻址可能频繁经过一些节点，这些节点如果被攻击，攻击者就相当于控制了网络中的一部分区域。两种途径避免之：

1. 限制节点随机生成nodeId的权限
2. 限制节点影响其他节点路由表的能力

因为Kademlia的路由表更新策略倾向于保留在网络中留存时间更长的节点，所以第2点天然就有较强的支持。

##### 女巫攻击(Sybil attack)

攻击者可以控制一大批节点进入网络，由此控制网络的一部分区域。在去中心化的P2P网络中，这种攻击不能完全消除，只能尽力避免(can not be prevented but only impeded)。一般需要节点“支付”一定的资源(带宽/CPU算力等)来认证进入网络。

##### 抖动攻击(Churn attack)

拥有一批节点的攻击者可以通过在网络中产生较大的波动来造成网络不稳定。仍是因为Kademlia的路由表更新策略倾向于保留在网络中留存时间更长的节点的缘故，这种攻击并不能造成很大影响。

##### 恶性路由(Adversarial routing)

Since a node is simply removed from a routing table when it neither responds with routing information nor routes any packet, the only way of influencing the networks’ routing is to return adversarial routing information. For example an adversarial node might just return other collaborating nodes which are closer to the queried key. This way an adversarial node routes a packet into its subnet of collaborators and neither the queried nor the closest node for a given key would be found. **This can be prevented by using a lookup algorithm that considers multiple disjoint paths**. The lookup succeeds when one path is free of adversarial nodes.  

With regard to a moderate number of adversarial nodes, lookup algorithms can significantly benefit from **disjoint paths **as well as **a low average path length**.

#### 其他类型攻击

##### Denial-of-Service

A adversarial may try to suborn(唆使) a victim to consume all its resources, i.e. memory, bandwidth, computational power. Thus the protocol needs to have mechanisms to allocate resources in a secure way. Physical attacks, such as jamming or side-band attacks are not considered in this paper.

##### Attacks on data storage

Key-based routing protocols are commonly used as building blocks to realize a distributed hash table (DHT) for data storage. To make it more difficult for adversarial nodes to modify stored data items, the same data item is **replicated** on a number of neighboring nodes. Although attacks on data storage are not regarded in this paper, the key-based routing layer has to provide a secure neighborhood to a given key.

### 主要设计

#### Secure nodeId assignment(节点ID的生成)

It should be hard for an attacker to generate a large number of nodeIds (Sybil attack) nor choose the nodeId freely (Eclipse attack). Furthermore **the nodeId should authenticate a node, i.e. no other node should be able to steal or fake the nodeId**. 

The latter[fake] can be achieved with one of the two methods: **Using a hash value over IP address and port** or by **hashing a public key**. 

The first solution has a significant **drawback** because with dynamically allocated IP addresses the nodeId will change subsequently. It is also not suitable to limit the number of generated nodeIds if you want to support networks with **NAT** in which several nodes appear to have the same public IP address. Finally there is no way of ensuring **integrity** of exchanged messages with those kind of nodeIds. 

This is why we advocate to use the hash over a public key to generate the nodeId. With this public key it is possible to sign messages exchanged by nodes. **Due to computational overhead** we differentiate between two **signature types**:

- ***Weak signature***: The weak signature does not sign the whole message. It is limited to **IP address, port and a timestamp**. The timestamp specifies how long the signature is valid. This prevents replay attacks if dynamic IP addresses are used. For synchronization issues the timestamp may be chosen in a very coarsegrained way. **The weak signature is primarily used in `FIND NODE` and `PING` messages where the integrity of the whole message is dispensable**.
- ***Strong signature***: The strong signature signs the full content of a message. This ensures integrity of the message and resilience against Man-in-the-Middle attacks(中间人攻击). Replay attacks(重放攻击) can be prevented with **nonces**(新鲜值) inside the RPC messages.

Those two signature types can authenticate nodes and ensure integrity of messages. 

We now need to impede the Sybil and Eclipse attack. This is done by either using a crypto puzzle or a signature from a central certificate authority, so we need to combine the signature types above with one of the following:

- ***Supervised signature***: If a signature’s public key additionally is signed by a trustworthy certificate authority, this signature is called **supervised signature**. This signature is needed to impede a Sybil attack in the network’s bootstrapping phase where only a few nodes exist in the network. A network size estimation can be used to decide if this signature is needed.

- ***Crypto puzzle signature***: In the absence of a trustworthy authority we need to impede the Eclipse and Sybil attack with a **crypto puzzle**. In [3]\(*Secure routing for structured peer-to-peer overlay networks.*\) the use of crypto puzzles for nodeId generation is rejected because they cannot be used to entirely prevent an attack. But in our opinion they are the most effective approach for distributed nodeId generation in an completely decentralised environment without trustworthy authority and should therefore be used to make an attack as hard as possible in such networks.

  For this reason we introduce two crypto puzzles as shown below. A static puzzle that impedes that the nodeId can be chosen freely and a dynamic puzzle that ensures that it is complex to generate a huge amount of nodeIds. H denotes a cryptographically secure hash function, ⊕ is the XOR operation, and the **c<sub>1</sub>** denote **crypto puzzle complexity**. **It is obvious that an increase of c<sub>1 </sub>decreases the space of possible public keys therefore the size of the public key must be increased subsequently**. **c<sub>1</sub>** is considered as a **constant**. Once the nodeId has been generated it stays fixed for a certain value. **c<sub>2</sub>** can be modified or extended over time when computational resources become cheaper.

  ![Image text](https://github.com/zwGithubStation/zwKade/blob/main/img-folder/SK_cryptoPuzzle.png)

  

#### Reliable sibling broadcast

#### Routing table maintainance

#### Lookup over disjoint paths



## Coral

Freedman, M. J., & Mazières, D. **Sloppy Hashing and Self-Organizing Clusters.** 2003



## BitTorrent

Bram Cohen. **Incentives Build Robustness in BitTorrent.** 2003



bittorrent spec：

- Index of BitTorrent Enhancement Proposals

  http://bittorrent.org/beps/bep_0000.html

- The BitTorrent Protocol Specification

  http://bittorrent.org/beps/bep_0003.html

- DHT Protocol

  http://bittorrent.org/beps/bep_0005.html



