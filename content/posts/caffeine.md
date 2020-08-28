---
title: "Caffeine内存缓存"
date: 2020-08-27T17:47:55+08:00
draft: false
---

# Eviction Policy  

在caffeine中缓存的管理使用的是Window TinyLFU(W-TinyLFU)的技术，结构如下图

![image](/image/caffeine/caffeine.png)

先来解释一下各个部分：

 window cache（绿色部分）: 新对象存入的区域；考虑到缓存的驱逐策略是基于类似LFU的粗略，这里的window cache是对新加入对象的一个保护策略，这些对象会现在window cache中呆一段时间，然后累积一定的frequency之后（LRU策略将其淘汰的时候）才会到TinyLFU中与其他缓存进行frequency的battle，决定去留；
main cache （红色和蓝色部分）：大部分缓存所在的区域；
 TinyLFU (紫色部分，一个admission filter）：当window cache空间存满之后，来自window cache区域的victim将会和来自main cache区域的victim进行比较，判断二者出现频率的大小，如果前者更大，那么将会将前者插入main cache，淘汰掉后者；反之，淘汰掉前者；

下面这个图可以更好的帮助了解Window TinyLFU的过程

![image](/image/caffeine/tiny-lfu.png)


在caffeine中，window cache所占大小约为总cache大小的1%，这是一个调参之后的结果，Window TinyLFU的作者做过大量实验证明在大多数的情况下1%能够实现缓存的最大命中率。

在TinyLFU中，使用count min sketch的方式来进行频率的估算，所谓count min sketch，就是利用多个哈希函数来对某个对象的频率进行计数，然后在计算这个对象的频率时，选择最小的哈希函数的计数值最为他的频率（这里可以考虑一种在单一哈希函数下的一个场景，如果有一个热键，他出现的频率很高，哈希之后的键位h1，然后又有一个很冷的键，出现的频率很低，然而由于哈希函数的碰撞其哈希之后的键也为h1，那么这就会造成很难淘汰这个冷键。这里采用多个哈希函数可以很好的规避这种情况）。

count min sketch的结构如下图：

![image](/image/caffeine/count-min-sketch.png)



在TinyLFU中，为了使缓存能够有一定的“鲜活性”，在每一次access的时候都会将缓存技术乘以一个aging factor（默认为2），这样的一个操作被称为“Reset”。这样做的好处是可以规避掉历史的“惯性”，举个例子，比如在视频网站上之前一个很火的剧，被点击了上亿次，如果不加以aging的措施的话，可能会一直霸榜，这不是我们希望看到的。



下面再来说一下main cache部分，采用的是Segmented LRU策略，即SLRU。其中会有两个部分：

protected，这部分的对象比较安全，但是如果protected队列满了之后会将lru淘汰入probation队列。
probation，这部分的对象较为危险，如果probation队列满了之后会淘汰lru的对象，但是如果这部分中的对象又再次被访问之后此对象会升级进入protected队列。


# Expiration Policy  

caffeine中的过期策略有两种，分别是fixed expiration policy和variable expiration policy，下面进行简单介绍。

对于定时过期策略（fixed expiration policy）而言，为每个队列设置一个固定的过期时间（expire after access，expire after write）如60s，同时可维护一个LRU队列，靠近head端的对象为更老的对象，靠近tail端的对象为更新的对象。当某个对象过期时，head朝向tail端移动，这样在进行过期对象清理的时候head左边的对象就是需要清理的，整个操作复杂度在O(1)级别。

![image](/image/caffeine/queue.png)



对于可变过期时间策略(variable expiration policy), 每个对象的过期时间可能不尽相同，caffeine引入时间轮来对此进行操作。关于时间轮的概念可参考kafka中时间轮的设计。

简单来说，就是利用一个定长数组，每个元素代表一定的时间范围，对应过期时间的缓存对象以双链表的形式存储在这个时间格上。当时间轮转过一圈时，上面一层时间轮每个元素代表的时间范围会是 原始的时间范围x数组元素个数。当时间推动时，时间指针指向的时间格中的对象会被进行过期操作，同时其上层时间轮对应的时间格上的对象会被重新hash填入下层时间轮中。原理其实很简单，可以和生活中的钟表的结构进行类比。

![image](/image/caffeine/time-wheel.png)



# 参考文献

https://dl.acm.org/doi/pdf/10.1145/3274808.3274816

http://highscalability.com/blog/2016/1/25/design-of-a-modern-cache.html

http://highscalability.com/blog/2019/2/25/design-of-a-modern-cachepart-deux.html





