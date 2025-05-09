# 第 29 章 无锁的原子操作：Redis如何应对并发访问？

为了保证并发访问的正确性，Redis 提供了两种方法，分别是加锁和原子操作。

加锁是一种常用的方法，在读取数据前，客户端需要先获得锁，否则就无法进行操作。当一个客户端获得锁后，就会一直持有这把锁，直到客户端完成数据更新，才释放这把锁。

看上去好像是一种很好的方案，但是，其实这里会有两个问题：一个是，如果加锁操作多，会降低系统的并发访问性能；第二个是，Redis 客户端要加锁时，需要用到分布式锁，而分布式锁实现复杂，需要用额外的存储系统来提供加解锁操作，我会在下节课向你介绍。

原子操作是另一种提供并发访问控制的方法。原子操作是指执行过程保持原子性的操作，而且原子操作执行时并不需要再加锁，实现了无锁操作。这样一来，既能保证并发控制，还能减少对系统并发性能的影响。

## 1. 并发访问中需要对什么进行控制？

我们说的并发访问控制，是指对多个客户端访问操作同一份数据的过程进行控制，以保证任何一个客户端发送的操作在 Redis 实例上执行时具有互斥性。例如，客户端 A 的访问操作在执行时，客户端 B 的操作不能执行，需要等到 A 的操作结束后，才能执行。

并发访问控制对应的操作主要是数据修改操作。当客户端需要修改数据时，基本流程分成两步：

- 客户端先把数据读取到本地，在本地进行修改；
- 客户端修改完数据后，再写回 Redis。

我们把这个流程叫做“读取 - 修改 - 写回”操作（Read-Modify-Write，简称为 RMW 操作）。当有多个客户端对同一份数据执行 RMW 操作的话，我们就需要让 RMW 操作涉及的代码以原子性方式执行。访问同一份数据的 RMW 操作代码，就叫做临界区代码。

## 2. Redis 的两种原子操作方法

为了实现并发控制要求的临界区代码互斥执行，Redis 的原子操作采用了两种方法：

1. 把多个操作在 Redis 中实现成一个操作，也就是单命令操作；
2. 把多个操作写到一个 Lua 脚本中，以原子性方式执行单个 Lua 脚本。

虽然 Redis 的单个命令操作可以原子性地执行，但是在实际应用中，数据修改时可能包含多个操作，至少包括读数据、数据增减、写回数据三个操作，这显然就不是单个命令操作了，那该怎么办呢？

别担心，Redis 提供了 `INCR/DECR` 命令，把这三个操作转变为一个原子操作了。`INCR/DECR` 命令可以对数据进行增值 / 减值操作，而且它们本身就是单个命令操作，Redis 在执行它们时，本身就具有互斥性。

但是，如果我们要执行的操作不是简单地增减数据，而是有更加复杂的判断逻辑或者是其他操作，那么，Redis 的单命令操作已经无法保证多个操作的互斥执行了。所以，这个时候，我们需要使用第二个方法，也就是 Lua 脚本。Redis 会把整个 Lua 脚本作为一个整体执行，在执行的过程中不会被其他命令打断，从而保证了 Lua 脚本中操作的原子性。

我再给你举个例子，来具体解释下 Lua 的使用。当一个业务应用的访问用户增加时，我们有时需要限制某个客户端在一定时间范围内的访问次数，比如爆款商品的购买限流、社交网络中的每分钟点赞次数限制等。

那该怎么限制呢？我们可以把客户端 IP 作为 key，把客户端的访问次数作为 value，保存到 Redis 中。客户端每访问一次后，我们就用 INCR 增加访问次数。

不过，在这种场景下，客户端限流其实同时包含了对访问次数和时间范围的限制，例如每分钟的访问次数不能超过 20。所以，我们可以在客户端第一次访问时，给对应键值对设置过期时间，例如设置为 60s 后过期。同时，在客户端每次访问时，我们读取客户端当前的访问次数，如果次数超过阈值，就报错，限制客户端再次访问。你可以看下下面的这段伪代码，它实现了对客户端每分钟访问次数不超过 20 次的限制。

```basic
//获取ip对应的访问次数
current = GET(ip)
//如果超过访问次数超过20次，则报错
IF current != NULL AND current > 20 THEN
    ERROR "exceed 20 accesses per second"
ELSE
    //如果访问次数不足20次，增加一次访问计数
    value = INCR(ip)
    //如果是第一次访问，将键值对的过期时间设置为60s后
    IF value == 1 THEN
        EXPIRE(ip,60)
    END
    //执行其他操作
    DO THINGS
END
```

此时，我们就可以使用 Lua 脚本来保证并发控制。我们可以把访问次数加 1、判断访问次数是否为 1，以及设置过期时间这三个操作写入一个 Lua 脚本，如下所示：

```shell
local current
current = redis.call("incr",KEYS[1])
if tonumber(current) == 1 then
    redis.call("expire",KEYS[1],60)
end
```

假设我们编写的脚本名称为 lua.script，我们接着就可以使用 Redis 客户端，带上 eval 选项，来执行该脚本。脚本所需的参数将通过以下命令中的 keys 和 args 进行传递。

```shell
$ redis-cli  --eval lua.script  keys , args
```

## 3. 每课一问

是否需要把读取客户端 ip 的访问次数 GET(ip)，以及判断访问次数是否超过 20 的判断逻辑，也加到 Lua 脚本中？

文章中的限流问题，如果业务并发不是很大，可以不把读取客户端 ip 访问次数和判断次逻辑放在 lua 脚本中。如果客户端并发量很大，为了保证逻辑正确性，是需要把逻辑放到 lua 脚本中的。

有一种情况会导致客户端访问次数超过 20，例如超过 20 个线程并发访问，都走到了步骤1，然后客户端都认为次数没有超过限制，之后都执行了步骤2，进入后续业务流程。在这种情况下，访问次数会超过 20，但下次再有线程请求进来时，会直接走到步骤3后返回。