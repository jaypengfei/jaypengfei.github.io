---
layout: post
title:  一致性哈希算法
date:   2018-04-25 16:19:00 +0800
categories: 算法
tag: Hash
---

* content
{:toc}
假设现在有三台缓存服务器，用于缓存图片，服务器分别编号为0、1、2，现在有3万张图片需要缓存，我们希望这些图片可以被均匀的缓存到这3台服务器上，以便能够分摊缓存的压力。那么要怎么做？我们可以将3万张图片无规律的平均缓存在3台服务器上，这样的话满足了均匀分布的要求。但是如果这样做的话，当我们需要访问某张缓存图片时，则需要遍历3台服务器，从3万个缓存项中找到需要访问的图片，遍历的效率太低，会带来严重的性能问题，这也就失去了缓存的意义，缓存的目的就是提高速度，减轻后端服务器的压力，如果每次访问一个缓存项都需要遍历所有的缓存服务器，想想就不太现实。那么，有没有更好的办法呢？

原始的做法是对缓存项的键进行hash，将hash后的结果对缓存服务器的数量进行取模操作，通过取模后的结果，决定缓存项将会缓存在哪一台服务器上。

但是上述的余数分布式算法存在一定的缺陷，试想一下，如果3台缓存服务器不能满足我们的需求，这时我们就需要增加缓存服务器的数量。假设，我们增加了一台缓存服务器，此时服务器的数量由3台变为了4台，此时如果还是用同样的算法对图片进行缓存，那么这张图片所在的服务器编号必定与之前有所不同，因为取模数有3变成了4。这样带来的后果就是当服务器数量变动时（增加或者减少），所有缓存的位置都可能会发生改变，也就是说，此时所有缓存在一定时间内是失效的。为了解决这个问题就诞生了一致性哈希算法。

1.算法背景
------------------------------------

一致性哈希算法在1997年由麻省理工学院的Karger等人在解决分布式Cache中提出的，设计目标是为了解决因特网中的热点(Hot spot)问题，初衷和CARP十分类似。一致性哈希修正了CARP使用的简单哈希算法带来的问题，使得DHT可以在P2P环境中真正得到应用。

但现在一致性hash算法在分布式系统中也得到了广泛应用，研究过memcached缓存数据库的人都知道，memcached服务器端本身不提供分布式cache的一致性，而是由客户端来提供，具体在计算一致性hash时采用如下步骤：

1. 首先求出memcached服务器（节点）的哈希值，并将其配置到0～2^32的圆（continuum）上。
2. 然后采用同样的方法求出存储数据的键的哈希值，并映射到相同的圆上。
3. 然后从数据映射到的位置开始顺时针查找，将数据保存到找到的第一个服务器上。如果超过2^32仍然找不到服务器，就会保存到第一台memcached服务器上。

![chashing]({{ '/styles/images/java/consistentHashing/pic1.png' | prepend: site.baseurl }})

<center>图1 一致性哈希映射图</center>

![chashing]({{ '/styles/images/java/consistentHashing/pic2.png' | prepend: site.baseurl }})

<center>图2 新增节点</center>

从上图中可以看出新增加一台服务器节点，如果使用余数分布式算法则会导致大量的缓存项失效（缓存项对应的服务器节点改变），但是如果采用了一致性哈希算法，那么只会影响到其中的一部分数据（图2）。

## 2.一致性哈希性质

考虑到分布式系统每个节点都有可能失效，并且新的节点很可能动态的增加进来，如何保证当系统的节点数目发生变化时仍然能够对外提供良好的服务，这是值得考虑的，尤其是在设计分布式缓存系统时，如果某台服务器失效，对于整个系统来说如果不采用合适的算法来保证一致性，那么缓存于系统中的所有数据都可能会失效（即由于系统节点数目变少，客户端在请求某一对象时需要重新计算其hash值（通常与系统中的节点数目有关），由于hash值已经改变，所以很可能找不到保存该对象的服务器节点），因此一致性哈希就显得至关重要，良好的分布式缓存系统中的一致性哈希算法应该满足以下几个方面：

> 平衡性(Balance)

平衡性是指哈希的结果能够尽可能均匀的分布到所有的缓存服务器中去，这样可以使得所有的缓存空间都得到利用。

> 单调性(Monotonicity)

单调性是指如果已经有一些内容通过哈希分派到了相应的缓存中，又有新的缓存服务器加入到系统中，那么哈希的结果应能够保证原有已分配的内容可以被映射到新的缓存服务器中去，而不会被映射到旧的缓存集合中的其他缓存区。简单的哈希算法往往不能满足单调性的要求，如最简单的线性哈希：x = (ax + b) mod (P)，在上式中，P表示全部缓冲的大小。不难看出，当缓冲大小发生变化时(从P1到P2)，原来所有的哈希结果均会发生变化，从而不满足单调性的要求。哈希结果的变化意味着当缓存空间发生变化时，所有的映射关系需要在系统内全部更新。而在P2P系统内，缓存的变化等价于Peer加入或退出系统，这一情况在P2P系统中会频繁发生，因此会带来极大计算和传输负荷。单调性就是要求哈希算法能够应对这种情况。

> 分散性(Spread)

在分布式环境中，终端有可能看不到所有的缓冲，而是只能看到其中的一部分。当终端希望通过哈希过程将内容映射到缓存上时，由于不同终端所见的缓存范围有可能不同，从而导致哈希的结果不一致，最终的结果是相同的内容被不同的终端映射到不同的缓存区中。这种情况显然是应该避免的，因为它导致相同内容被存储到不同缓存中去，降低了系统存储的效率。分散性的定义就是上述情况发生的严重程度。好的哈希算法应能够尽量避免不一致的情况发生，也就是尽量降低分散性。

> 负载(Load)

负载问题实际上是从另一个角度看待分散性问题。既然不同的终端可能将相同的内容映射到不同的缓冲区中，那么对于一个特定的缓存区而言，也可能被不同的用户映射为不同的内容。与分散性一样，这种情况也是应当避免的，因此好的哈希算法应能够尽量降低缓冲的负荷。

> 平滑性(Smoothness)

平滑性是指缓存服务器的数目平滑改变和缓存对象的平滑改变是一致的。

## 3.虚拟节点

另外，一致性哈希算法在服务器节点过少时，容易因为节点分布不均匀而造成数据倾斜的问题，例如系统中只有两台服务器，其环分布如下：

![chashing]({{ '/styles/images/java/consistentHashing/pic7.png' | prepend: site.baseurl }})

<center>图3 服务器环分布</center>

此时可能会造成大量数据集中到节点A上，而只有少量会定位到节点B上。为了解决这种数据倾斜问题，一致性哈希算法引入了虚拟节点机制，即对每一个服务器节点计算多个哈希值，每个计算结果位置都放置一个此服务器节点，称为虚拟节点。具体做法可以是在服务器IP或主机名的后面增加编号来实现。例如上面的情况，可以为每台服务器计算三个虚拟节点，于是可以分别计算：Node A #1, Node A #2, Node A #3, Node B #1, Node B #2, Node B #3的哈希值，于是形成了6个虚拟节点：

![chashing]({{ '/styles/images/java/consistentHashing/pic8.png' | prepend: site.baseurl }})

<center>图4 虚拟节点</center>

同时数据定位算法不变，只是多了一步虚拟节点到实际节点的映射，例如定位到“Node A#1”、“Node A#2”、“Node A#3”三个虚拟节点的数据均定位到Node A上。这样就解决了服务节点少时数据倾斜的问题。在实际应用中，通常将虚拟节点数设置为32甚至更大，因此即使很少的服务节点也能做到相对均匀的数据分布。

> 参考

1. https://www.cnblogs.com/lpfuture/p/5796398.html
2. http://www.zsythink.net/archives/1182


<hr>
​最后的最后，老婆我爱你。







