## 目录

- [初见](#初见)
  - [P2P是什么](#p2p是什么)
  - [种子.torrent文件](#种子torrent文件)
  - [去中心化网络（DHT）](#去中心化网络dht)
    - [工作模式](#工作模式)
    - [DHT 网络中”朋友圈“的按距分桶](#dht-网络中朋友圈的按距分桶)
    - [DHT 网络寻址”朋友“的方式](#dht-网络寻址朋友的方式)
    - [DHT 网络中“朋友圈”的维护方式](#dht-网络中朋友圈的维护方式)
- [Kademlia](#kademlia)
  - [缘起：以相互借书为例](#缘起以相互借书为例)
  - [P2P文件共享系统的一种实现: IPFS](#p2p文件共享系统的一种实现-ipfs)
  - [Distributed Hash Tables](#distributed-hash-tables)
  - [Map数据的去中心化及有组织(Organized)分区](#map数据的去中心化及有组织organized分区)
    - [使用hash表实现分区(Partitions): Chord](#使用hash表实现分区partitions-chord)
    - [使用搜索树实现分区(Partitions): Kademlia](#使用搜索树实现分区partitions-kademlia)
  - [(完全)二叉前缀树中叶子节点(IDs或者keys)的距离定义](#完全二叉前缀树中叶子节点ids或者keys的距离定义)
  - [寻址数据项的节点ID路由表](#寻址数据项的节点id路由表)
  - [k-bucket逐级寻址](#k-bucket逐级寻址)
  - [支持节点动态离线与上线(Dynamic Leaves and Joins)](#支持节点动态离线与上线dynamic-leaves-and-joins)
    - [k值的理解](#k值的理解)
    - [最近节点更新(k-nearest update)](#最近节点更新k-nearest-update)
    - [k-bucket更新](#k-bucket更新)
    - [将新的节点信息加入k-bucket的策略](#将新的节点信息加入k-bucket的策略)
    - [k-bucket更新(bucket refresh)](#k-bucket更新bucket-refresh)
    - [新节点上线的流程](#新节点上线的流程)
    - [数据项流向就近节点](#数据项流向就近节点)
  - [RPCs](#rpcs)
  - [优化及扩展](#优化及扩展)


## 初见

### P2P是什么

想要下载一个文件的时候，你只要得到那些已经存在了文件的 peer，并和这些 peer 之间，建立点对点的连接，而不需要到中心服务器上，就可以就近下载文件。

一旦下载了文件，你也就成为 peer 中的一员，你旁边的那些机器，也可能会选择从你这里下载文件，所以当你使用 P2P 软件的时候，例如 BitTorrent，往往能够看到，既有下载流量，也有上传的流量，也即你自己也加入了这个 P2P 的网络，自己从别人那里下载，同时也提供给其他人下载。

可以想象，这种方式，参与的人越多，下载速度越快，一切完美。

### 种子.torrent文件

原始的p2p有一个问题，当你想下载一个文件的时候，怎么知道哪些 peer 有这个文件呢？这就用到种子啦，也即咱们比较熟悉的.torrent 文件。

.torrent 文件由两部分组成，分别是：**文件信息**和**announce（tracker URL）**。

文件信息里面有这些内容：

- info 区：

  这里指定的是该种子有几个文件、文件有多长、目录结构，以及目录和文件的名字。

- Name 字段：

  指定顶层目录名字。

- 每个段的大小：

  BitTorrent（简称 BT）协议把一个文件分成很多个小段，然后分段下载。

- 段哈希值：

  将整个种子中，每个段的 SHA-1 哈希值拼在一起。

下载时，BT 客户端首先解析.torrent 文件，得到 tracker 地址，然后连接 **tracker 服务器**。

tracker 服务器回应下载者的请求，将其他下载者（包括发布者）的 IP 提供给下载者。

下载者再连接其他下载者，根据.torrent 文件，两者分别向对方告知自己已经有的块，然后交换对方没有的数据。

此时不需要其他服务器参与，并分散了单个线路上的数据流量，因此减轻了服务器的负担。

下载者每得到一个块，需要算出下载块的 Hash 验证码，并与.torrent 文件中的对比。如果一样，则说明块正确，不一样则需要重新下载这个块。这种规定是为了解决下载内容的准确性问题。

从这个过程也可以看出，这种方式特别依赖 tracker。tracker 需要收集下载者信息的服务器，并将此信息提供给其他下载者，使下载者们相互连接起来，传输数据。虽然下载的过程是非中心化的，但是加入这个 P2P 网络的时候，都需要借助 tracker 中心服务器，**这个服务器是用来登记有哪些用户在请求哪些资源**。所以，**这种工作方式有一个弊端，一旦 tracker 服务器出现故障或者线路遭到屏蔽，BT 工具就无法正常工作了**。

### 去中心化网络（DHT）

那能不能彻底非中心化呢？

于是，后来就有了一种叫作 DHT（Distributed Hash Table）的去中心化网络。

每个加入这个 DHT 网络的人，都要负责存储这个网络里的资源信息和其他成员的联系信息，相当于所有人一起构成了一个庞大的分布式存储数据库。

有一种著名的 DHT 协议，叫 Kademlia 协议。这个和区块链的概念一样，很抽象，我来详细讲一下这个协议。

Kademlia协议源于美国纽约大学 P. Maymounkov 和 D. Mazieres 在2002年发布的研究论文 《Kademlia: A peerto-peer information system based on the XOR metric》。与其他DHT实现技术比较，Kad通过独特的以异或算法（XOR）为距离度量基础，建立了一种全新的DHT拓扑结构，相比于其他算法，大大提高了路由查询速度。在BiTtorrent、emule以及BitComet 和 BitSpirit等下载软件中，都用到Kademlia核心算法。

在P2P网络中，任何一个节点启动之后，它都有两个角色：

- **peer角色**：

  监听一个 TCP 端口，用来上传和下载文件，这个角色表明，我这里有某个文件。

- **DHT node角色**：

  监听一个 UDP 的端口，通过这个角色，这个节点加入了一个 DHT 的网络。

在 DHT 网络里面，每一个 DHT node 都有一个 ID。这个 ID 是一个很长的串。每个 DHT node 都有责任掌握一些知识，**也就是文件索引，也即它应该知道某些文件是保存在哪些节点上**。它只需要有这些知识就可以了，**而它自己本身不一定就是保存这个文件的节点**。

#### 工作模式

当然，每个 DHT node 不会有全局的知识，也即不知道所有的文件保存在哪里，它只需要知道一部分。

那应该知道哪一部分呢？这就需要用哈希算法计算出来。

每个文件可以计算出一个哈希值，而 DHT node 的 ID 是和哈希值相同长度的串。

DHT 算法是这样规定的：如果一个文件计算出一个哈希值，则ID和这个哈希值一样的那个 DHT node，就有责任知道从哪里下载这个文件，即便它自己没保存这个文件。

当然不一定这么巧，总能找到和哈希值一模一样的，有可能一模一样的 DHT node 也下线了，所以 DHT 算法还规定：除了一模一样的那个 DHT node 应该知道，ID 和这个哈希值非常接近的 N 个 DHT node 也应该知道。

什么叫和哈希值接近呢？例如只修改了最后一位，就很接近；修改了倒数 2 位，也不远；修改了倒数 3 位，也可以接受。总之，凑齐了规定的 N 这个数就行。

![Image text](https://github.com/zwGithubStation/zwKade/blob/main/pic/DHTNet.webp)

如上图，文件 1 通过哈希运算，得到匹配 ID 的 DHT node 为 node C，当然还会有其他的，我这里没有画出来。所以，node C 有责任知道文件 1 的存放地址，虽然 node C 本身没有存放文件 1。同理，文件 2 通过哈希运算，得到匹配 ID 的 DHT node 为 node E，但是 node D 和 E 的 ID 值很近，所以 node D 也知道。当然，文件 2 本身没有必要一定在 node D 和 E 里，但是碰巧这里就在 E 那有一份。

接下来一个新的节点 node new 上线了。如果想下载文件 1，它首先要加入 DHT 网络，如何加入呢？在这种模式下，**种子.torrent 文件里面就不再是 tracker 的地址了，而是一个 list 的 node 的地址，而所有这些 node 都是已经在 DHT 网络里面的**。

当然随着时间的推移，很可能有退出的，有下线的，但是我们假设，不会所有的都联系不上，总有一个能联系上。node new 只要在种子里面找到一个 DHT node，就加入了网络。node new 会计算文件 1 的哈希值，并根据这个哈希值了解到，和这个哈希值匹配，或者很接近的 node 上知道如何下载这个文件，例如计算出来的哈希值就是 node C。

但是 node new 不知道怎么联系上 node C，因为种子里面的 node 列表里面很可能没有 node C，但是它可以问，DHT 网络特别像一个社交网络，node new 只有去它能联系上的 node 问，你们知道不知道 node C 的联系方式呀？在 DHT 网络中，每个 node 都保存了一定的联系方式，但是肯定没有 node 的所有联系方式。DHT 网络中，节点之间通过互相通信，也会交流联系方式，也会删除联系方式。和人们的方式一样，你有你的朋友圈，你的朋友有它的朋友圈，你们互相加微信，就互相认识了，过一段时间不联系，就删除朋友关系。

有个理论是，社交网络中，任何两个人直接的距离不超过六度，也即你想联系比尔盖茨，也就六个人就能够联系到了。所以，node new 想联系 node C，就去万能的朋友圈去问，并且求转发，朋友再问朋友，很快就能找到。如果找不到 C，也能找到和 C 的 ID 很像的节点，它们也知道如何下载文件 1。在 node C 上，告诉 node new，下载文件 1，要去 B、D、 F，于是 node new 选择和 node B 进行 peer 连接，开始下载，它一旦开始下载，自己本地也有文件 1 了，于是 node new 告诉 node C 以及和 node C 的 ID 很像的那些节点，我也有文件 1 了，可以加入那个文件拥有者列表了。

但是你会发现 node new 上没有文件索引，但是根据哈希算法，一定会有某些文件的哈希值是和 node new 的 ID 匹配上的。在 DHT 网络中，会有节点告诉它，你既然加入了咱们这个网络，你也有责任知道某些文件的下载地址。好了，一切都分布式了。

这里面遗留几个细节的问题：

- **DHT node ID 以及文件哈希是个什么东西？**

  节点 ID 是一个随机选择的 160bits（20 字节）空间，文件的哈希也使用这样的 160bits 空间。

- **所谓 ID 相似，具体到什么程度算相似？**

  在 Kademlia 网络中，距离是通过**异或（XOR）**计算的。我们就不以 160bits 举例了。我们以 5 位来举例。01010 与 01000 的距离，就是两个 ID 之间的异或值，为 00010，也即为 2。 01010 与 00010 的距离为 01000，也即为 8,。**01010 与 00011 的距离为 01001，也即 8+1=9** 。

  以此类推，**高位不同的，表示距离更远一些；低位不同的，表示距离更近一些，总的距离为所有的不同的位的距离之和**。

  这个距离不能比喻为地理位置，**因为在 Kademlia 网络中，位置近不算近，ID 近才算近，所以我把这个距离比喻为社交距离**，也即在朋友圈中的距离，或者社交网络中的距离。这个和你住的位置没有关系，和人的经历关系比较大。

  更清晰的一种表述来说，整个网络空间可以被表示成一颗二叉树(**节点路由树**)，而网络中的节点都被当作二叉树的叶子，并且每一个节点的位置都由其 ID 值的最短前缀唯一的确定，如下图：

  ![Image text](https://github.com/zwGithubStation/zwKade/blob/main/pic/binaryTree.png)

  - 节点路由树的构造：
    1. 节点ID由二进制表示，从高到低每一位依次进行第2步操作
    2. 当前位于第n位的二进制位，位于路由树的第n层(从上到下)，如值为1进入右子树，否则进入左子树
    3. 节点ID的各个位处理完毕后，新生成的最底层叶子节点，即代表该节点ID

  

  

#### DHT 网络中”朋友圈“的按距分桶

就像人一样，虽然我们常联系人的只有少数，但是朋友圈里肯定是远近都有。

DHT 网络的朋友圈也是一样，远近都有，并且按距离分层。假设某个节点的 ID 为 01010，如果一个节点的 ID，前面所有位数都与它相同，只有最后 1 位不同。这样的节点只有 1 个，为 01011。

与基础节点的异或值为 00001，即距离为 1；对于 01010 而言，这样的节点归为“k-bucket 1”。

如果一个节点的 ID，前面所有位数都相同，从倒数第 2 位开始不同，这样的节点只有 2 个，即 01000 和 01001，与基础节点的异或值为 00010 和 00011，即距离范围为 2 和 3；对于 01010 而言，这样的节点归为“k-bucket 2”。

如果一个节点的 ID，前面所有位数相同，从倒数第 i 位开始不同，这样的节点只有 2^(i-1) 个，与基础节点的距离范围为[2^(i-1), 2^i)；对于 01010 而言，这样的节点归为“k-bucket i”。

最终到从倒数 160 位就开始都不同。你会发现，**差距越大，陌生人越多**，但是朋友圈不能都放下，所以**每一层都只放 K 个，这是参数可以配置**。

#### DHT 网络寻址”朋友“的方式

假设，node A 的 ID 为 00110，要找 node B ID 为 10000，异或距离为 10110，距离范围在[2^4, 2^5)，所以这个目标节点可能在“k-bucket 5”中，这就说明 B 的 ID 与 A 的 ID 从第 5 位开始不同，所以 B 可能在“k-bucket 5”中。

然后，A 看看自己的 k-bucket 5 有没有 B。如果有，太好了，找到你了；如果没有，在 k-bucket 5 里随便找一个 C。因为是二进制，C、B 都和 A 的第 5 位不同，那么 C 的 ID 第 5 位肯定与 B 相同，即它与 B 的距离会小于 2^4，**相当于比 A、B 之间的距离缩短了一半以上**。

再请求 C，在它自己的通讯录里，按同样的查找方式找一下 B。如果 C 知道 B，就告诉 A；如果 C 也不知道 B，那 C 按同样的搜索方法，可以在自己的通讯录里找到一个离 B 更近的 D 朋友（D、B 之间距离小于 2^3），把 D 推荐给 A，A 请求 D 进行下一步查找。

![Image text](https://github.com/zwGithubStation/zwKade/blob/main/pic/binarySearchInKademlia.webp)

Kademlia 的这种查询机制，是通过**折半查找**的方式来收缩范围，**对于总的节点数目为 N，最多只需要查询 log2(N) 次，就能够找到**。

例如，上图这个最差的情况。A 和 B 每一位都不一样，所以相差 31，A 找到的朋友 C，不巧正好在中间。和 A 的距离是 16，和 B 距离为 15，于是 C 去自己朋友圈找的时候，不巧找到 D，正好又在中间，距离 C 为 8，距离 B 为 7。于是 D 去自己朋友圈找的时候，不巧找到 E，正好又在中间，距离 D 为 4，距离 B 为 3，E 在朋友圈找到 F，距离 E 为 2，距离 B 为 1，最终在 F 的朋友圈距离 1 的地方找到 B。

当然这是最最不巧的情况，每次找到的朋友都不远不近，正好在中间。如果碰巧了，在 A 的朋友圈里面有 G，距离 B 只有 3，然后在 G 的朋友圈里面一下子就找到了 B，两次就找到了。

#### DHT 网络中“朋友圈”的维护方式

Kademlia 算法中，每个节点只有 4 个指令:

- **PING**：测试一个节点是否在线，还活着没，相当于打个电话，看还能打通不。

- **STORE**：要求一个节点存储一份数据，既然加入了组织，有义务保存一份数据。

- **FIND_NODE**：根据节点 ID 查找一个节点，就是给一个 160 位的 ID，通过上面朋友圈的方式找到那个节点。

- **FIND_VALUE**：根据 KEY 查找一个数据，实则上跟 FIND_NODE 非常类似。KEY 就是文件对应的 160 位的 ID，就是要找到保存了文件的节点。

每个 bucket 里的节点，都按最后一次接触的时间倒序排列。这就相当于，朋友圈里面最近联系过的人往往是最熟的。

**每次执行四个指令中的任意一个都会触发更新**。当一个节点与自己接触时，检查它是否已经在 k-bucket 中，也就是说是否已经在朋友圈。如果在，**那么将它挪到 k-bucket 列表的最底，也就是最新的位置**，刚联系过，就置顶一下，方便以后多联系；如果不在，新的联系人要不要加到通讯录里面呢？假设通讯录已满的情况，PING 一下列表最上面，也即最旧的一个节点。如果 PING 通了，将旧节点挪到列表最底，并丢弃新节点，老朋友还是留一下；如果 PING 不通，删除旧节点，并将新节点加入列表，这人联系不上了，删了吧。这个机制保证了任意节点加入和离开都不影响整体网络。

## Kademlia

### 缘起：以相互借书为例

图书馆里书籍堆集，一般可以使用了Dewey Decimal System的图书馆可以让借书人方便的获取想要看的书籍。同时，与朋友们互相分享自己拥有的图书也是一件快事。

相互借书会存在一些问题。比如如何确定谁拥有哪本书？一般人可不会费力维护这样的完整信息表单(维护起来会很痛苦)。那假设我们不关心谁拥有哪本书，也不关心会有新的朋友参与/老朋友离开，我们应该如何设计一个可以简单操作的互借系统呢？

具体的说，我们面临以下难题：

- 终究要确认谁拥有哪本书
- 如何与周围的朋友沟通以获取期望的图书
- 如何处理这样的场景：你与一个朋友发生了争吵，他决定立即退出你的互借系统
- 当你有一个惊人的朋友圈(比如一百万个朋友)时，你的互借系统还能正常运行么

这样一个不太恰当的例子，帮我们引出了P2P文件共享系统的概念。致力于实现P2P文件共享系统的数据结构一般称之为分布式哈希表(DHT, Distributed Hash Table)。

### P2P文件共享系统的一种实现: IPFS

Inter-Planetary File System([**IPFS**][IPFS Website])自然是底层基于DHTs的明星产品，但这里不做涉及，我们关注于DHT的实现方式。

### Distributed Hash Tables

对于数据存储来说，传统的Map可以满足我们的一般需求。每个数据项以<key, value>的形式存储，Map一般会提供两个接口：`PUT(key, value)` 和 `GET(key)`，前者存储一条数据，后者根据key值获取数据。

当数据量激增时，很难支撑所有Map数据存在单个计算机内。一种解决方案是将数据分布在多个计算机上存储，然后通过访问一个中心调度计算机，决策哪条数据应该存往/取自哪台计算机。显然，当调度计算机下线时，整个系统就变得不可用了。

DHT就是一个存储在多计算机上的系统，其目标就是解决这种中心化带来的问题。我们仍希望使用我们的PUT/GET接口，但希望在某些计算机下线时，系统仍然可用，也即DHT设计时需要考虑的两个特性：

- **可扩展性 Scalable** 当有更多的计算机参与到系统中，计算机集群规模不断增长时，我们希望DHT可以稳定的保证数据较为均匀的存储在集群内的各个计算机上
- **容错性 Fault-Tolerant** 当一个计算机脱离集群时，我们应该仍然可以访问到该计算机之前存储的数据

当然，如果所有计算机同时下线，我们就无处存储任何数据。因此一般系统假设计算机会以一个较低的速率离开集群。我们需要在计算机之间进行key-value数据的复制来保证当计算机下线时数据可恢复。

### Map数据的去中心化及有组织(Organized)分区

我们的目标是在没有中心化调度节点的前提下，设计一种协议使得我们的map数据能够相对均匀的存储在集群的每一个节点上。集群中的每一个节点都应该有一个ID(identifier)，对于一条数据来说，我们将其存储在ID与该数据的key值**最为相近**的节点上。**节点的ID和数据的key值可以在同一个取值范围中取值，比如在160位的整数(160-bit integers)中选取**。这一思路有以下几个优势：

1. ID的选取不依赖中心调度节点
2. 依赖于内含的**距离**概念，ID的统一随机选取可以更加有效的实现数据在各个节点的均匀分布
3. 新增一个节点，只需要对一小部分的数据进行与新增节点距离上的设定即可(key值进行简单的移位操作以表征与节点距离的变化)

显然很多人好奇160位这个数字是如何敲定的。简单来说，对一个可以处理hash碰撞的hash函数来说，任何形式的key值都可以被计算为一个160-bit的值。由于历史原因(**Kademlia论文于2002年10月10号发表**)，最初的哈希算法设定为**SHA-1**。

在如上的设定下，PUT/GET操作显得简单许多，只需要找到距离对应key值最近的节点，在该节点上进行存取操作即可。这让我们的问题变为：**给定一个key值，如何找到距离最近的节点？**

实现map所用的底层数据结构通常有两种：**哈希表(a hash table)或者是搜索树(a search tree)**。我们首选了解一下hash表如何在map的key值空间上实现距离测算。

#### 使用hash表实现分区(Partitions): Chord

可参考该链接的介绍：https://zhuanlan.zhihu.com/p/53711866

暂不做涉及。

#### 使用搜索树实现分区(Partitions): Kademlia

我们关注的是一个二叉前缀树(Trie,前缀树/字典树)上的分区。

![Image text](https://github.com/zwGithubStation/zwKade/blob/main/pic/binaryTree1.png)

在上面一棵高度为3的二叉树中，画圈的叶子节点用来代表节点ID。**我们可以说所有的节点ID和数据项的key值都要从[0,1,...2<sup>3</sup>-1]这个范围内选取(二进制形式)。使用完全二叉树的形式可以在叶子节点看到所有的可选值**。上图中，000,110,111代表三个节点ID。

**我们需要有确定的算法，将其他的叶子节点(代表数据项)划分给其中某一特定节点。**

### (完全)二叉前缀树中叶子节点(IDs或者keys)的距离定义

**数据项key值与节点ID之间的亲密度(closeness)的精确定义:**

1. 将数据项key值与节点ID转化为二进制表示

2. 所有节点ID囊括在一个候选集合中

3. 对于所有尚存于候选集合中的所有节点ID，从最高位开始，在集合只剩下最后一个ID之前，逐位进行第4步
4. 对于数据项key的第N位值x，如果至少有一个候选节点在第N位的值也等于x，将所有第N位值非x的节点踢出候选集合，该轮结束，返回第3步
5. 最后剩下的一个ID即为与数据项key值最为接近的节点ID

显然，当有`i-1`个ID节点被踢出候选集合时，下一个被踢出的就是距离数据项key值`第i近`的节点ID。

另一个角度，也可以这样描述这一问题：

1. 起初，某一节点ID与数据项key的距离值D为0

将某一节点ID的二进制值与数据项key的二进制值，逐位从高到低比较：

- 在第N位时，如果该位上的值相同，则D值不变，否则D加上一个距离值D<sub>N</sub> 
- 对于任意的D<sub>N</sub> 来说，应该永远存在D<sub>N</sub> > (D<sub>N-1</sub> +D<sub>N-2</sub> + ... + D<sub>0</sub>) ，用来表示高位不同所产生的距离值大于所有高位相同的场景
- 最终所有的候选节点ID都能产生一个距离值D，**其排序严格意义上表示了每个候选节点与数据项key值的距离排序**

**很容易想到这个流程与XOR的计算十分类似！** 最终，我们有了：

数据项key值与节点ID之间的距离(Distance)的合理定义:

**数据项key值与节点ID之间的距离可以表示为两者XOR运算的整数值。**

由此，基于XOR运算的正整数值，即可得到key/ID之间的距离；与数据项key值**最近**的节点ID就是与之XOR运算值最小的节点ID。

最后要说的是，**XOR可以用来进行距离度量(XOR is a distance metric)。但显然并不能用于常理上的十进制数间的距离度量，**例如：

> 388 XOR 403 = 23(无意义的运算)	110000100 XOR 110010100 = 000010000(有意义的运算)
>
> 388 XOR 404 = 16(无意义的运算)	110000100 XOR 110010011 = 000010111(有意义的运算)

388与{403, 404}的距离大小，并不能用XOR体现，但110000100<sub>2</sub> 与 {110010100<sub>2</sub> , 110010011<sub>2</sub>} 的距离大小，就可以用XOR体现。

### 寻址数据项的节点ID路由表

在确认了哪些数据项key应该存放在哪些节点上后，接下来的任务是：

**节点如何通过其他节点信息，找到指定key值的数据项。**

我们有如下的**<u>目标</u>**：

- **尽可能的找当前能找到的距离目标数据项最近的节点ID**

- **应该在维护尽可能少的节点ID信息前提下，就能相对较快的找到数据所在的节点**
- 每一步获取一个中间节点信息，都应该至少在定位节点的路上**“前进了一个bit”**，这样可将算法的运行复杂度控制在`O(log(n))`内[**因为单个ID是160bit表示的一个整数，即相当于将2<sup>160</sup> 个候选，剪枝为160个候选**]

因此我们可以考虑有一个**按照某种方式组织的，包含指数级数目的节点ID信息路由表**。

### k-bucket逐级寻址

在确定路由表的具体形式之前，我们先定义我们的**寻址策略**：

1. 在路由表中找到距离目标数据集最近的节点C(XOR操作)

2. 向C确认是否维护目标数据集信息；如果不存在，走a，b；如果存在则返回

   a. 请求C在其路由表中找到距离目标数据集最近的节点D

   b. 将D的信息更新至自己的路由表中，对D发起步骤2一样的请求，回到步骤2

我们有如下的**<u>假设</u>**：

1. **路由表中的寻址信息都是正确合法的**
2. **如果当前节点的每层k-bucket的范围内真实存在节点，该层k-bucket至少包含一个节点信息(k-bucket的构造如下图所示)**

![Image text](https://github.com/zwGithubStation/zwKade/blob/main/pic/k-bucketExamples.png)

我们将最终应该获取的节点ID记为id<sub>global</sub> ，我们可以证明：

在“寻址策略”中的任何一次查询节点过程(假设向节点c发起了查询，请求c查询离目标数据项距离最近的节点ID)中，节点c返回的下一层查询节点ID(标记为id<sub>close</sub>)**满足如下性质**：

**对于节点c来说，如果id<sub>global</sub>也在c的某层k-bucket内，则该k-bucket就是id<sub>close</sub>所在的k-bucket**。

![Image text](https://github.com/zwGithubStation/zwKade/blob/main/pic/k-bucket-prove.png)

证明很简单，**鉴于我们的算法设定：**

1. 我们设定了每一层bucket内的ID的若干位前缀相同，且每一层bucket之间的前缀不同

2. 我们设定了在bucket内的搜索都力图实现找到bucket中距离目标最近的节点ID

所以如果id<sub>close</sub>所在bucket之外的某一bucket中存在某一节点ID(标记为id<sub>k</sub>)与id<sub>global</sub>的距离相较于id<sub>global</sub>与id<sub>close</sub>之间更近(**更为严格的描述是：id<sub>close</sub> 相较于其他bucket内的任意节点ID信息id<sub>other</sub>，都存在 len(LCP(id<sub>global</sub>, id<sub>close</sub>)) > len(LCP(id<sub>global</sub>, id<sub>other</sub>)) **)，那就与1，2设定相矛盾了(id<sub>k</sub> 与 id<sub>global</sub> 有更为接近的前缀----->设定1成立，但并没有返回id<sub>k</sub> 所在的bucket内的节点ID -----> 与设定2矛盾)。

通过上图的图示也可以很好的理解。

通过引入LCP(id<sub>global</sub>, id<sub>close</sub>)的概念，在理想情况下(也即在上面假设1，2成立的条件下)，我们针对查询目标id<sub>global</sub>向中间节点ID<sub>pre</sub>进行的查询所获得新的节点ID<sub>suc</sub>，都应满足：

> **len(LCP(id<sub>global</sub>, ID<sub>suc</sub>)) - len(LCP(id<sub>global</sub>, ID<sub>pre</sub>)) &ge; 1**

直白的说，每次都应该在与id<sub>global</sub>前缀一致的路上，至少前进一步。**在整个节点规模为N的网络中，可以保证规模在log(N)的查询次数下获取到id<sub>global</sub>。**

从**“自上而下遍历前缀树”**的角度考虑也许更加易于理解。每次的查询保证遍历可以至少下沉一层。在ID的空间为160bits的设定下，不超过160次查询即可找到id<sub>global(当然我们希望这棵树能够尽可能的达到完全二叉树那样的平衡，这依赖节点ID的随机化生成)。

### 支持节点动态离线与上线(Dynamic Leaves and Joins)

#### k值的理解

对于零星下线的场景(sporadically leaving),显然k-bucket中的参数**k**可以帮助我们。通常可以这样理解k值：**可能会在同一小时内同时下线的节点数目的最小值**，换言之，几乎不可能有超过k个节点在同一小时内同时下线。Kademlia最初论文中实验性的将k设置为20。

在设定了k值时，为了有效应对节点下线，**每个节点的每个bucket内应该存放k个节点信息而非1个，这样就在理论上保证了每个bucket中至少有一个节点属于在线状态**。

#### 最近节点更新(k-nearest update)

为了使节点感知“周围”的节点变化，我们可以使用下面的步骤更新当前节点的最近k个节点信息：

1. 通过XOR找到当前路由表中的最近k个节点ID
2. 询问每一个节点他们自身最近的k个节点ID(很可能就包含他们自己)
3. 根据各个节点获取到的结果信息，更新最近k节点信息

#### k-bucket更新

理论上我们应该尽可能保证维护的k-bucket中的每一个节点信息是最新的(up-to-date)。所以当k-bucket中的某一节点C在过去的一小时内没有被当前节点查询过的话，当前节点应该向C发送是否可达的ping消息，如果没有应答，则C节点应从当前节点的k-bucket中移除。

这样的逻辑使得任意节点在离线是无需知会其他节点：因为其他节点如果存有该节点信息，总会被移除出k-bucket。

#### 将新的节点信息加入k-bucket的策略

在如下的情况下我们将新的节点信息加入k-bucket中：

- 当前节点在进行信息更新时获取到了一个新的节点信息(ID, IP)
- 该(ID, IP)对应应该存入的bucket中，还有多余的位置(容量<k)

第二条规则表明我们的新增/抛弃策略对当前已经在维护的节点**更加友好**，这样做的原因是基于对P2P网络中节点的观察：**在网络中活跃越久的节点，有更高的可能性继续留在网络中。**因此在可以维护的信息有限的前提下，更倾向于维护旧有节点信息。

#### k-bucket更新(bucket refresh)

对于还没有填充满的bucket来说，我们可以主动找到一些新节点将其加入bucket中维护。因为bucket中可以维护的节点ID具有范围性(相同前缀)，我们可以随机构造可能的ID信息并发起查询，如果存在则将其新增至bucket中维护。

#### 新节点上线的流程

一个新节点`j`能够进入网络的前提：`j`必须知道至少一个其他节点的(ID, IP)信息。在此基础上进行自己的信息填充：

- `j`根据最初所知的节点信息，查询获取到一批距离自己最近的节点信息。
- 进行k-bucket更新操作，填充自己各层的bucket。
- 被`j`访问过的节点都可以感知到`j`的存在，也都有机会将`j`的信息更新至自己的k-bucket中。

#### 数据项流向就近节点

当某一节点`current`获取到一个新的节点`new`的信息时，可以对自己所存储的数据项`data`进行测算，如果`XOR(data.keyID, current.ID) > XOR(data.keyID, new.ID)`，则`current`可以向`new`发起PUT(data)的操作。这样实现数据项往距离自己最近的节点上存储的初衷。

### RPCs

随着主要流程的梳理后，现在可以正式定义Kademlia中几个必要的RPC。

**PING**

一个节点能够收到`PING`调用，用来查询该节点是否仍然在线。

**FIND_COMP(id)**

一个节点能够收到`FIND_COMP(id)`调用，节点会返回自己路由表中距离自己**前k近**的的节点信息(ID, IP)

**FIND_VALUE(key)**

一个节点能够收到`FIND_VALUE(key)`调用，如果数据项`(key,value)`就在该节点上存储，数据value将被返回；如果数据项不在该节点上存储，节点想当于接受到了一个`FIND_COMP(key)`调用，继续问询自己路由表中维护的其他节点。

**STORE(key,value)**

一个节点能够收到`STORE(key,value)`调用，节点会将该数据项存在本地。

**注意1：冗余存储与自动删除**

为了确保数据项能够存留在网络中，因收到`STORE`调用或者发起`FIND_VALUE(key)`调用等操作而在本地保有数据项的节点，应该以一个固定的频率(例如每24小时)向外发出`STORE`调用。

此外，为了避免数据集存储因冗余的`STORE`操作无限膨胀，节点应该以某种速率删除本地存储的数据集。

**注意2：异步因子α**

为了增速查找流程，`FIND_COMP/FIND_VALUE`操作可以在异步因子的设定下做异步。在Kademlia原始论文中`α = 3`。也即`FIND_COMP`操作可以并行的同时发个三个不同的节点。

最终，我们构造了DHT来实现数据的存取：

- `GET(key)`

  被转化为`FIND_VALUE(key)`，节点向最近的**k**个节点请求获取key对应的数据项。

- `PUT(key,value)`

  首先`FIND_COMP(key)`确认当前的数据所在节点(如果存在)，然后通过`STORE`操作将数据项写入最近

### 优化及扩展



[IPFS Website]:https://ipfs.tech/