## 目录



## Kademlia

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

![Image text](https://github.com/zwGithubStation/zwKade/blob/main/img-folder/find_node_1.png)



<img src="https://github.com/zwGithubStation/zwKade/blob/main/img-folder/find_node_1.png" width="1047px" height="682px">

<img src="https://github.com/zwGithubStation/zwKade/blob/main/img-folder/find_node_1.png" style="zoom:100%" />

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


