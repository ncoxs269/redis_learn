# 第 12 章 有一亿个keys要统计，应该用哪种集合？

在 Web 和移动应用的业务场景中，我们经常需要保存这样一种信息：一个 key 对应了一个数据集合。我举几个例子。

- 手机 App 中的每天的用户登录信息：一天对应一系列用户 ID 或移动设备 ID；
- 电商网站上商品的用户评论列表：一个商品对应了一系列的评论；
- 用户在手机 App 上的签到打卡信息：一天对应一系列用户的签到记录；
- 应用网站上的网页访问信息：一个网页对应一系列的访问点击。

我们知道，Redis 集合类型的特点就是一个键对应一系列的数据，所以非常适合用来存取这些数据。但是，在这些场景中，除了记录信息，我们往往还需要对集合中的数据进行统计，例如：

- 在移动应用中，需要统计每天的新增用户数和第二天的留存用户数；
- 在电商网站的商品评论中，需要统计评论列表中的最新评论；
- 在签到打卡中，需要统计一个月内连续打卡的用户数；
- 在网页访问记录中，需要统计独立访客（Unique Visitor，UV）量。

要想选择合适的集合，我们就得了解常用的集合统计模式。这节课，我就给你介绍集合类型常见的四种统计模式，包括**聚合统计、排序统计、二值状态统计和基数统计**。

## 1. 聚合统计

所谓的聚合统计，就是指统计多个集合元素的聚合结果，包括：统计多个集合的共有元素（交集统计）；把两个集合相比，统计其中一个集合独有的元素（差集统计）；统计多个集合的所有元素（并集统计）。

在刚才提到的场景中，统计手机 App 每天的新增用户数和第二天的留存用户数，正好对应了聚合统计。要完成这个统计任务，我们可以用一个集合记录所有登录过 App 的用户 ID，同时，用另一个集合记录每一天登录过 App 的用户 ID。然后，再对这两个集合做聚合统计。我们来看下具体的操作。

记录所有登录过 App 的用户 ID 还是比较简单的，我们可以直接使用 `Set` 类型，把 key 设置为 `user:id`，表示记录的是用户 ID，value 就是一个 `Set` 集合，里面是所有登录过 App 的用户 ID，我们可以把这个 `Set` 叫作累计用户 `Set`。

累计用户 Set 中没有日期信息，我们是不能直接统计每天的新增用户的。所以，我们还需要把每一天登录的用户 ID，记录到一个新集合中，我们把这个集合叫作每日用户 Set，它有两个特点：

1. key 是 `user:id` 以及当天日期，例如 `user:id:20200803`；
2. value 是 Set 集合，记录当天登录的用户 ID。

在统计每天的新增用户时，我们只用计算每日用户 Set 和累计用户 Set 的差集就行。统计第二天的留存用户数只要计算两天的 Set 的交集即可。

Set 的差集、并集和交集的计算复杂度较高，在数据量较大的情况下，如果直接执行这些计算，会导致 Redis 实例阻塞。所以，我给你分享一个小建议：**你可以从主从集群中选择一个从库，让它专门负责聚合计算，或者是把数据读取到客户端，在客户端来完成聚合统计，这样就可以规避阻塞主库实例和其他从库实例的风险了**。

## 2. 排序统计

我以在电商网站上提供最新评论列表的场景为例，进行讲解。最新评论列表包含了所有评论中的最新留言，这就要求集合类型能对元素保序，也就是说，集合中的元素可以按序排列，这种对元素保序的集合类型叫作有序集合。

在 Redis 常用的 4 个集合类型中（`List`、`Hash`、`Set`、`Sorted Set`），`List` 和 `Sorted Set` 就属于有序集合。`List` 是按照元素进入 `List` 的顺序进行排序的，而 `Sorted Set` 可以根据元素的权重来排序，我们可以自己来决定每个元素的权重值。比如说，我们可以根据元素插入 `Sorted Set` 的时间确定权重值，先插入的元素权重小，后插入的元素权重大。

我先说说用 `List` 的情况。每个商品对应一个 `List`，这个 `List` 包含了对这个商品的所有评论，而且会按照评论时间保存这些评论，每来一个新评论，就用 `LPUSH` 命令把它插入 `List` 的队头。在只有一页评论的时候，我们可以很清晰地看到最新的评论，但是，在实际应用中，网站一般会分页显示最新的评论列表，一旦涉及到分页操作，`List` 就可能会出现问题了。

假设当前的评论 `List` 是`{A, B, C, D, E, F}`（其中，A 是最新的评论，以此类推，F 是最早的评论），在展示第一页的 3 个评论时，我们可以用下面的命令，得到最新的三条评论 A、B、C：

```shell
LRANGE product1 0 2
1) "A"
2) "B"
3) "C"
```

然后，再用下面的命令获取第二页的 3 个评论，也就是 D、E、F。

```shell
LRANGE product1 3 5
1) "D"
2) "E"
3) "F"
```

但是，如果在展示第二页前，又产生了一个新评论 G，评论 G 就会被 `LPUSH` 命令插入到评论 `List` 的队头，评论 `List` 就变成了`{G, A, B, C, D, E, F}`。此时，再用刚才的命令获取第二页评论时，就会发现，评论 C 又被展示出来了，也就是 C、D、E。

```shell
LRANGE product1 3 5
1) "C"
2) "D"
3) "E"
```

`List` 是通过元素在 `List` 中的位置来排序的，当有一个新元素插入时，原先的元素在 `List` 中的位置都后移了一位。对比新元素插入前后，`List` 相同位置上的元素就会发生变化，用 `LRANGE` 读取时，就会读到旧元素。

其实读到旧数据也是可以符合预期场景的，典型的就是大多数论坛回复按时间排序，新发布的内容在最上面，当我翻页时发现读到旧数据说明有新回复了。

和 `List` 相比，`Sorted Set` 就不存在这个问题，因为它是根据元素的实际权重来排序和获取数据的。我们可以按评论时间的先后给每条评论设置一个权重值，然后再把评论保存到 `Sorted Set` 中。`Sorted Set` 的 `ZRANGEBYSCORE` 命令就可以按权重排序后返回元素。这样的话，即使集合中的元素频繁更新，`Sorted Set` 也能通过 `ZRANGEBYSCORE` 命令准确地获取到按序排列的数据。

假设越新的评论权重越大，目前最新评论的权重是 N，我们执行下面的命令时，就可以获得最新的 10 条评论：

```shell
ZRANGEBYSCORE comments N-9 N
```

所以，在面对需要展示最新列表、排行榜等场景时，如果数据更新频繁或者需要分页显示，建议你优先考虑使用 Sorted Set。

## 3. 二值状态统计

这里的二值状态就是指集合元素的取值就只有 0 和 1 两种。在签到打卡的场景中，我们只用记录签到（1）或未签到（0），所以它就是非常典型的二值状态。

在签到统计时，每个用户一天的签到用 1 个 bit 位就能表示，一个月（假设是 31 天）的签到情况用 31 个 bit 位就可以，而一年的签到也只需要用 365 个 bit 位，根本不用太复杂的集合类型。这个时候，我们就可以选择 `Bitmap`。这是 Redis 提供的扩展数据类型。我来给你解释一下它的实现原理。

`Bitmap` 本身是用 `String` 类型作为底层数据结构实现的一种统计二值状态的数据类型。`String` 类型是会保存为二进制的字节数组，所以，Redis 就把字节数组的每个 bit 位利用起来，用来表示一个元素的二值状态。你可以把 `Bitmap` 看作是一个 bit 数组。

`Bitmap` 提供了 `GETBIT/SETBIT` 操作，使用一个偏移值 `offset` 对 bit 数组的某一个 bit 位进行读和写。不过，需要注意的是，`Bitmap` 的偏移量是从 0 开始算的，也就是说 `offset` 的最小值是 0。当使用 `SETBIT` 对一个 bit 位进行写操作时，这个 bit 位会被设置为 1。`Bitmap` 还提供了 `BITCOUNT` 操作，用来统计这个 bit 数组中所有“1”的个数。

假设我们要统计 ID 3000 的用户在 2020 年 8 月份的签到情况，就可以按照下面的步骤进行操作。

第一步，执行下面的命令，记录该用户 8 月 3 号已签到。

```shell
SETBIT uid:sign:3000:202008 2 1 
```

第二步，检查该用户 8 月 3 日是否签到。

```shell
GETBIT uid:sign:3000:202008 2
```

第三步，统计该用户在 8 月份的签到次数。

```shell
BITCOUNT uid:sign:3000:202008
```

这样，我们就知道该用户在 8 月份的签到情况了，是不是很简单呢？接下来，你可以再思考一个问题：如果记录了 1 亿个用户 10 天的签到情况，你有办法统计出这 10 天连续签到的用户总数吗？

在介绍具体的方法之前，我们要先知道，`Bitmap` 支持用 `BITOP` 命令对多个 `Bitmap` 按位做“与”“或”“异或”的操作，操作的结果会保存到一个新的 `Bitmap` 中。

我以按位“与”操作为例来具体解释一下。从下图中，可以看到，三个 `Bitmap` `bm1`、`bm2` 和 `bm3`，对应 bit 位做“与”操作，结果保存到了一个新的 `Bitmap` 中（示例中，这个结果 `Bitmap` 的 key 被设为“resmap”）。

![img](第12章 有一亿个keys要统计，应该用哪种集合.assets/4151af42513cf5f7996fe86c6064f97a.jpg)

回到刚刚的问题，在统计 1 亿个用户连续 10 天的签到情况时，你可以把每天的日期作为 key，每个 key 对应一个 1 亿位的 Bitmap，每一个 bit 对应一个用户当天的签到情况。这要求同一个用户都在相同的`offset`上，所以如何排序是一个问题

接下来，我们对 10 个 Bitmap 做“与”操作，得到的结果也是一个 Bitmap。在这个 Bitmap 中，只有 10 天都签到的用户对应的 bit 位上的值才会是 1。最后，我们可以用 BITCOUNT 统计下 Bitmap 中的 1 的个数，这就是连续签到 10 天的用户总数了。

## 4. 基数统计

基数统计就是指统计一个集合中不重复的元素个数。对应到我们刚才介绍的场景中，就是统计网页的 UV。网页 UV 的统计有个独特的地方，就是需要去重，一个用户一天内的多次访问只能算作一次。在 Redis 的集合类型中，Set 类型默认支持去重，所以看到有去重需求时，我们可能第一时间就会想到用 Set 类型。当你需要统计 UV 时，可以直接用 `SCARD` 命令，这个命令会返回一个集合中的元素个数。

有一个用户 user1 访问 page1 时，你把这个信息加到 Set 中：

```go
SADD page1:uv user1
```

但是，如果 page1 非常火爆，UV 达到了千万，这个时候，一个 Set 就要记录千万个用户 ID。对于一个搞大促的电商网站而言，这样的页面可能有成千上万个，如果每个页面都用这样的一个 Set，就会消耗很大的内存空间。

有什么办法既能完成统计，还能节省内存吗？这时候，就要用到 Redis 提供的 `HyperLogLog` 了。`HyperLogLog` 是一种用于统计基数的数据集合类型，它的最大优势就在于，当集合元素数量非常多时，它计算基数所需的空间总是固定的，而且还很小。

在 Redis 中，每个 HyperLogLog 只需要花费 12 KB 内存，就可以计算接近 2^64 个元素的基数。你看，和元素越多就越耗费内存的 Set 和 Hash 类型相比，HyperLogLog 就非常节省空间。

在统计 UV 时，你可以用 PFADD 命令（用于向 HyperLogLog 中添加新元素）把访问页面的每个用户都添加到 HyperLogLog 中。

```shell
PFADD page1:uv user1 user2 user3 user4 user5
```

接下来，就可以用 PFCOUNT 命令直接获得 page1 的 UV 值了，这个命令的作用就是返回 HyperLogLog 的统计结果。PFCOUNT page1:uv

```shell
PFCOUNT page1:uv
```

关于 HyperLogLog 的具体实现原理，你不需要重点掌握，不会影响到你的日常使用，我就不多讲了。如果你想了解一下，课下可以看看[这条链接](http://en.wikipedia.org/wiki/HyperLogLog)。

不过，有一点需要你注意一下，HyperLogLog 的统计规则是基于概率完成的，所以它给出的统计结果是有一定误差的，标准误算率是 0.81%。这也就意味着，你使用 HyperLogLog 统计的 UV 是 100 万，但实际的 UV 可能是 101 万。虽然误差率不算大，但是，如果你需要精确统计结果的话，最好还是继续用 Set 或 Hash 类型。

## 5. 每课一问

这节课，我们学习了 4 种典型的统计模式，以及各种集合类型的支持情况和优缺点，我想请你聊一聊，你还遇到过其他的统计场景吗？用的是怎样的集合类型呢？

现在大数据情况下都是通过实时流方式统计pvuv，不太会基于redis，做统计业界主流还是上数仓用hive等做报表。
