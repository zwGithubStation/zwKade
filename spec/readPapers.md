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

#### Efficient key re-publishing

### Prove

### improve preformance

## S/Kademlia

Baumgart, I., & Mies, S. **S/Kademlia : A practicable approach towards secure key-based routing S / Kademlia : A Practicable Approach Towards Secure Key-Based Routing.** June, 2014 



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

