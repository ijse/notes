> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/nL7GR-HGG4JQjWzW8A1wAA)

**1. 分布式锁**
-----------

### **1.1 分布式锁介绍**

分布式锁是控制不同系统之间访问共享资源的一种锁实现，如果不同的系统或同一个系统的不同主机之间共享了某个资源时，往往需要互斥来防止彼此干扰来保证一致性。

### **1.2 为什么需要分布式锁**

在单机部署的系统中，使用线程锁来解决高并发的问题，多线程访问共享变量的问题达到数据一致性，如使用 synchornized、ReentrantLock 等。但是在后端集群部署的系统中，程序在不同的 JVM 虚拟机中运行，且因为 synchronized 或 ReentrantLock 都只能保证同一个 JVM 进程中保证有效，所以这时就需要使用分布式锁了。这里就不再赘述 synchornized 锁的原理，想了解可以读这篇文章[《深入理解 synchronzied 底层原理》](https://mp.weixin.qq.com/s?__biz=MzkxNTE3ODU0NA==&mid=2247486419&idx=1&sn=74a364f15280f0b65f2ea6bba960b318&scene=21#wechat_redirect)。

### **1.3 分布式锁需要具备的条件**

分布式锁需要具备互斥性、不会死锁和容错等。互斥性，在于不管任何时候，应该只能有一个线程持有一把锁；不会死锁在于即使是持有锁的客户端意外宕机或发生进程被 kill 等情况时也能释放锁，不至于导致整个服务死锁。容错性指的是只要大多数节点正常工作，客户端应该都能获取和释放锁。

**2. 分布式锁的实现方式**
----------------

目前主流的分布式锁的实现方式，基于数据库实现分布式锁、基于 Redis 实现分布式锁、基于 ZooKeeper 实现分布式锁，本篇文章主要介绍了 Redis 实现的分布式锁。

### **2.1 由单机部署到集群部署锁的演变**

一开始在 redis 设置一个默认值 key：ticket 对应的值为 20，并搭建一个 Spring Boot 服务，用来模拟多窗口卖票现象，配置类的代码就不一一列出了。

#### **2.1.1 单机模式解决并发问题**

一开始的时候在 redis 预设置的门票值 ticket=20，那么当一个请求进来之后，会判断是否余票是否是大于 0，若大于 0 那么就将余票减一，再重新写入 Redis 中，倘若库存小于 0，那么就会打印错误日志。

```
@RestController
@Slf4j
public class RedisLockController {
    
    @Resource
    private Redisson redisson;
    
    @Resource
    private StringRedisTemplate stringRedisTemplate;
    
    @RequestMapping("/lock")
    public String deductTicket() throws InterruptedException {
        String lockKey = "ticket";
        int ticketCount = Integer.parseInt(stringRedisTemplate.opsForValue().get(lockKey));
        if (ticketCount > 0) {
            int realTicketCount = ticketCount - 1;
            log.info("扣减成功，剩余票数：" + realTicketCount + "");
            stringRedisTemplate.opsForValue().set(lockKey, realTicketCount + "");
        } else {
            log.error("扣减失败，余票不足");
        }
        return "end";
    }
    
}
```

**代码运行分析：**这里明显有一个问题，就是当前若有两个线程同时请求进来，那么两个线程同时请求这段代码时，如图 thread 1 和 thread 2 同时，两个线程从 Redis 拿到的数据都是 20，那么执行完成后 thread 1 和 thread 2 又将减完后的库存 ticket=19 重新写入 Redis，那么数据就会产生问题，实际上两个线程各减去了一张票数，然而实际写进就减了一次票数，就出现了数据不一致的现象。

![](https://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74B6eN0ZqvvMibcS9HYWaZiaUSERWxRzJK8h7icthDVgYwKRrDmQvicIDRNboX9lhHhU9N7h6UoqjE4GNw/640?wx_fmt=png)

这种问题很好解决，上述问题的产生其实就是从 Redis 中拿数据和减余票不是原子操作，那么此时只需要将按下图代码给这俩操作加上 synchronized 同步代码快就能解决这个问题。

```
@RestController
@Slf4j
public class RedisLockController {

    @Resource
    private Redisson redisson;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @RequestMapping("/lock")
    public String deductTicket() throws InterruptedException {
        String lockKey = "ticket";
        synchronized (this) {
            int ticketCount = Integer.parseInt(stringRedisTemplate.opsForValue().get(lockKey));
            if (ticketCount > 0) {
                int realTicketCount = ticketCount - 1;
                log.info("扣减成功，剩余票数：" + realTicketCount + "");
                stringRedisTemplate.opsForValue().set(lockKey, realTicketCount + "");
            } else {
                log.error("扣减失败，余票不足");
            }
        }
        return "end";
    }

}
```

**代码运行分析：**此时当多个线程执行到第 14 行的位置时，只会有一个线程能够获取锁，进入 synchronized 代码块中执行，当该线程执行完成后才会释放锁，等下个线程进来之后就会重新给这段代码上锁再执行。说简单些就是让每个线程排队执行代码块中的代码，从而保证了线程的安全。

上述的这种做法如果后端服务只有一台机器，那毫无疑问是没问题的，但是现在互联网公司或者是一般软件公司，后端服务都不可能只用一台机器，最少都是 2 台服务器组成的后端服务集群架构，那么 synchronized 加锁就显然没有任何作用了。

如下图所示，若后端是两个微服务构成的服务集群，由 nginx 将多个的请求负载均衡转发到不同的后端服务上，由于 synchronize 代码块只能在同一个 JVM 进程中生效，两个请求能够同时进两个服务，所以上面代码中的 synchronized 就一点作用没有了。

![](https://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74B6eN0ZqvvMibcS9HYWaZiaUSMoiaybFXoxCdk9FXMLl0a67vEEJ6BKBc3eTOpjwW4Sia1Valgez1Qb6g/640?wx_fmt=png)

用 JMeter 工具随便测试一下，就很简单能发现上述代码的 bug。实际上 synchronized 和 juc 包下个那些锁都是只能用于 JVM 进程维度的锁，并不能运用在集群或分布式部署的环境中。

#### **2.1.2 集群模式解决并发问题**

通过上面的实验很容易就发现了 synchronized 等 JVM 进程级别的锁并不能解决分布式场景中的并发问题，就是为了应对这种场景产生了分布式锁。

本篇文章介绍了 Redis 实现的分布式锁，可以通过 Redis 的 setnx（只在键 key 不存在的情况下, 将键 key 的值设置为 value。若键 key 已经存在, 则 SETNX 命令不做任何动作。）的指令来解决的，这样就可以解决上面集群环境的锁不唯一的情况。

```
@RestController
@Slf4j
public class RedisLockController {

    @Resource
    private Redisson redisson;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @RequestMapping("/lock")
    public String deductTicket() throws InterruptedException {

        String lockKey = "ticket";
        // redis setnx 操作
        Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "dewu");
        if (Boolean.FALSE.equals(result)) {
            return "error";
        }

        int ticketCount = Integer.parseInt(stringRedisTemplate.opsForValue().get(lockKey));
        if (ticketCount > 0) {
            int realTicketCount = ticketCount - 1;
            log.info("扣减成功，剩余票数：" + realTicketCount + "");
            stringRedisTemplate.opsForValue().set(lockKey, realTicketCount + "");
        } else {
            log.error("扣减失败，余票不足");
        }

        stringRedisTemplate.delete(lockKey);
        return "end";
    }

}
```

**代码运行分析：**代码是有问题的，就是当执行扣减余票操作时，若业务代码报了异常，那么就会导致后面的删除 Redis 的 key 代码没有执行到，就会使 Redis 的 key 没有删掉的情况，那么 Redis 的这个 key 就会一直存在 Redis 中，后面的线程再进来执行下面这行代码都是执行不成功的，就会导致线程死锁，那么问题就会很严重了。

为了解决上述问题其实很简单，只要加上一个 try...finally 即可，这样业务代码即使抛了异常也可以正常的释放锁。**setnx + try ... finally 解决**，具体代码如下：

```
@RestController
@Slf4j
public class RedisLockController {

    @Resource
    private Redisson redisson;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @RequestMapping("/lock")
    public String deductTicket() throws InterruptedException {

        String lockKey = "ticket";
        // redis setnx 操作
        try {
            Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "dewu");
            if (Boolean.FALSE.equals(result)) {
                return "error";
            }
      
            int ticketCount = Integer.parseInt(stringRedisTemplate.opsForValue().get(lockKey));
          if (ticketCount > 0) {
              int realTicketCount = ticketCount - 1;
              log.info("扣减成功，剩余票数：" + realTicketCount + "");
              stringRedisTemplate.opsForValue().set(lockKey, realTicketCount + "");
          } else {
              log.error("扣减失败，余票不足");
          }
        } finally {
            stringRedisTemplate.delete(lockKey);
        }
        return "end";
    }

}
```

**代码运行分析**：上述问题解决了，但是又会有新的问题，当程序执行到 try 代码块中某个位置服务宕机或者服务重新发布，这样就还是会有上述的 Redis 的 key 没有删掉导致死锁的情况。这样可以使用 Redis 的过期时间来进行设置 key，**setnx + 过期时间解决**，如下代码所示：

```
@RestController
@Slf4j
public class RedisLockController {

    @Resource
    private Redisson redisson;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @RequestMapping("/lock")
    public String deductTicket() throws InterruptedException {

        String lockKey = "ticket";
        // redis setnx 操作
        try {
            Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "dewu");
            //程序执行到这
            stringRedisTemplate.expire(lockKey, 10, TimeUnit.SECONDS);
            if (Boolean.FALSE.equals(result)) {
                return "error";
            }

            int ticketCount = Integer.parseInt(stringRedisTemplate.opsForValue().get(lockKey));
          if (ticketCount > 0) {
              int realTicketCount = ticketCount - 1;
              log.info("扣减成功，剩余票数：" + realTicketCount + "");
              stringRedisTemplate.opsForValue().set(lockKey, realTicketCount + "");
          } else {
              log.error("扣减失败，余票不足");
          }
        } finally {
            stringRedisTemplate.delete(lockKey);
        }
        return "end";
    }

}
```

**代码运行分析**：上述代码解决了因为程序执行过程中宕机导致的锁没有释放导致的死锁问题，但是如果代码像上述的这种写法仍然还是会有问题，当程序执行到第 18 行时，程序宕机了，此时 Redis 的过期时间并没有设置，也会导致线程死锁的现象。可以用了 Redis 设置的原子命设置过期时间的命令，**原子性过期时间的 setnx 命令**，如下代码所示：

```
@RestController
@Slf4j
public class RedisLockController {

    @Resource
    private Redisson redisson;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @RequestMapping("/lock")
    public String deductTicket() throws InterruptedException {

        String lockKey = "ticket";
        // redis setnx 操作
        try {
            Boolean result = stringRedisTemplate.opsForValue().setIfPresent(lockKey, "dewu", 10, TimeUnit.SECONDS);
            if (Boolean.FALSE.equals(result)) {
                return "error";
            }

            int ticketCount = Integer.parseInt(stringRedisTemplate.opsForValue().get(lockKey));
          if (ticketCount > 0) {
              int realTicketCount = ticketCount - 1;
              log.info("扣减成功，剩余票数：" + realTicketCount + "");
              stringRedisTemplate.opsForValue().set(lockKey, realTicketCount + "");
          } else {
              log.error("扣减失败，余票不足");
          }

        } finally {
            stringRedisTemplate.delete(lockKey);
        }
        return "end";
    }

}
```

**代码运行分析**：通过设置原子性过期时间命令可以很好的解决上述这种程序执行过程中突然宕机的情况。这种 Redis 分布式锁的实现看似已经没有问题了，但在高并发场景下任会存在问题，一般软件公司并发量不是很高的情况下，这种实现分布式锁的方式已经够用了，即使出了些小的数据不一致的问题，也是能够接受的，但是如果是在高并发的场景下，上述的这种实现方式还是会存在很大问题。

![](https://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74B6eN0ZqvvMibcS9HYWaZiaUSZmWP3JH2B6wwIkdHBCvqw4rTXrjYWfTglN3wdK3FTiaag9AwDchicicKw/640?wx_fmt=png)

如上面代码所示，该分布式锁的过期时间是 10s，假如 thread 1 执行完成时间需要 15s，且当 thread 1 线程执行到 10s 时，Redis 的 key 恰好就是过期就直接释放锁了，此时 thread 2 就可以获得锁执行代码了，假如 thread 2 线程执行完成时间需要 8s，那么当 thread 2 线程执行到第 5s 时，恰好 thread 1 线程执行了释放锁的代码————**stringRedisTemplate**.delete(lockKey);  此时，就会发现 thread 1 线程删除的锁并不是其自己的加锁，而是 thread 2 加的锁；那么 thread 3 就又可以进来了，那么假如一共执行 5s，那么当 thread 3 执行到第 3s 时，thread 2 又会恰好执行到释放锁的代码，那么 thread 2 又删除了 thread 3 加的锁。

在高并发场景下，倘若遇到上述问题，那将是灾难性的 bug，只要高并发存在，那么这个分布式锁就会时而加锁成功时而加锁失败。

解决上述问题其实也很简单，让每个线程加的锁时给 Redis 设置一个唯一 id 的 value，每次释放锁的时候先判断一下线程的唯一 id 与 Redis 存的值是否相同，若相同即可释放锁。**设置线程 id 的原子性过期时间的 setnx 命令，**具体代码如下：

```
@RestController
@Slf4j
public class RedisLockController {

    @Resource
    private Redisson redisson;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @RequestMapping("/lock")
    public String deductTicket() throws InterruptedException {

        String lockKey = "ticket";
        String threadUniqueKey = UUID.randomUUID().toString();
        // redis setnx 操作
        try {
            Boolean result = stringRedisTemplate.opsForValue().setIfPresent(lockKey, threadUniqueKey, 10, TimeUnit.SECONDS);
            if (Boolean.FALSE.equals(result)) {
                return "error";
            }

            int ticketCount = Integer.parseInt(stringRedisTemplate.opsForValue().get(lockKey));
          if (ticketCount > 0) {
              int realTicketCount = ticketCount - 1;
              log.info("扣减成功，剩余票数：" + realTicketCount + "");
              stringRedisTemplate.opsForValue().set(lockKey, realTicketCount + "");
          } else {
              log.error("扣减失败，余票不足");
          }
        } finally {
            if (Objects.equals(stringRedisTemplate.opsForValue().get(lockKey), threadUniqueKey)) {
                stringRedisTemplate.delete(lockKey);
            }
        }
        return "end";
    }

}
```

**代码运行分析**：上述实现的 Redis 分布式锁已经能够满足大部分应用场景了，但是还是略有不足，比如当线程进来需要的执行时间超过了 Redis key 的过期时间，那么此时已经释放了，你其他线程就可以立马获得锁执行代码，就又会产生 bug 了。

分布式锁 Redis key 的过期时间不管设置成多少都不合适，比如将过期时间设置为 30s，那么如果业务代码出现了类似慢 SQL、查询数据量很大那么过期时间就不好设置了。那么这里有没有什么更好的方案呢？答案是有的——锁续命。

那么锁续命方案的原来就在于当线程加锁成功时，会开一个分线程，取锁过期时间的 1/3 时间点定时执行任务，如上图的锁为例，每 10s 判断一次锁是否存在 (即 Redis 的 key)，若锁还存在那么就直接重新设置锁的过期时间，若锁已经不存在了那么就直接结束当前的分线程。  

### **2.2 Redison 框架实现 Redis 分布式锁  
**

上述 “锁续命” 方案说起来简单，但是实现起来还是挺复杂的，于是市面上有很多开源框架已经帮我们实现好了，所以就不需要自己再去重复造轮子再去写一个分布式锁了，所以本次就拿 Redison 框架来举例，主要是可以学习这种设计分布式锁的思想。

#### **2.2.1 Redison 分布式锁的使用**

Redison 实现的分布式锁，使用起来还是非常简单的，具体代码如下：

```
@RestController
@Slf4j
public class RedisLockController {

    @Resource
    private Redisson redisson;

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @RequestMapping("/lock")
    public String deductTicket() throws InterruptedException {

        //传入Redis的key
        String lockKey = "ticket";
        // redis setnx 操作
        RLock lock = redisson.getLock(lockKey);
        try {
            //加锁并且实现锁续命
            lock.lock();
            int ticketCount = Integer.parseInt(stringRedisTemplate.opsForValue().get(lockKey));
          if (ticketCount > 0) {
              int realTicketCount = ticketCount - 1;
              log.info("扣减成功，剩余票数：" + realTicketCount + "");
              stringRedisTemplate.opsForValue().set(lockKey, realTicketCount + "");
          } else {
              log.error("扣减失败，余票不足");
          }

        } finally {
            //释放锁
            lock.unlock();
        }
        return "end";
    }

}
```

#### **2.2.2 Redison 分布式锁的原理**

Redison 实现分布式锁的原理流程如下图所示，当线程 1 加锁成功，并开始执行业务代码时，Redison 框架会开启一个后台线程，每隔锁过期时间的 1/3 时间定时判断一次是否还持有锁 (Redis 中的 key 是否还存在)，若不持有那么就直接结束当前的后台线程，若还持有锁，那么就重新设置锁的过期时间。当线程 1 加锁成功后，那么线程 2 就会加锁失败，此时线程 2 就会就会做类似于 CAS 的自旋操作，一直等待线程 1 释放了之后线程 2 才能加锁成功。

![](https://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74B6eN0ZqvvMibcS9HYWaZiaUSSney9J4VJ6loppgSJOT2wGjc46iaYQlf0GJ0HhuIvAnVsib5JMlwK4Gg/640?wx_fmt=png)

#### **2.2.3 Redison 分布式锁的源码分析**

Redison 底层实现分布式锁时使用了大量的 lua 脚本保证了其加锁操作的各种原子性。Redison 实现分布式锁使用 lua 脚本的好处主要是能保证 Redis 的操作是原子性的，Redis 会将整个脚本作为一个整体执行，中间不会被其他命令插入。

**Redisson 核心使用 lua 脚本加锁源码分析：**

方法名为 tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command)：

```
//使用lua脚本加锁方法
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
     internalLockLeaseTime = unit.toMillis(leaseTime);

     return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
           //当第一个线程进来会直接执行这段逻辑                            
           //判断传入的Redis的key是否存在，即String lockKey = "ticket";
           "if (redis.call('exists', KEYS[1]) == 0) then " +  
           //如果不存在那么就设置这个key为传入值、当前线程id 即参数ARGV[2]值(即getLockName(threadId)),并且将线程id的value值设置为1
             "redis.call('hset', KEYS[1], ARGV[2], 1); " +  
          //再给这个key设置超时时间，超时时间即参数ARGV[1](即internalLockLeaseTime的值)的时间
             "redis.call('pexpire', KEYS[1], ARGV[1]); " +    
             "return nil; " +
             "end; " +
          //当第二个线程进来，Redis中的key已经存在(锁已经存在)，那么直接进这段逻辑
          //判断这个Redis key是否存在且当前的这个key是否是当前线程设置的
           "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
          //如果是的话，那么就进入重入锁的逻辑，利用hincrby指令将第一个线程进来将线程id的value值设置为1再加1 
          //然后每次释放锁的时候就会减1，直到这个值为0，这把锁就释放了，这点与juc的可重锁类似           
          //“hincrby”指令为Redis hash结构的加法
             "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
             "redis.call('pexpire', KEYS[1], ARGV[1]); " +
             "return nil; " +
             "end; " +
          //倘若不是本线程加的锁，而是其他线程加的锁,由于上述lua脚本都是有线程id的校验，那么上面的两段lua脚本都不会执行
      //那么此时这里就会将当前这个key的过期时间返回 
             "return redis.call('pttl', KEYS[1]);",
             Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));   // KEYS[1])  ARGV[1]   ARGV[2]
}
// getName()传入KEYS[1]，表示传入解锁的keyName，这里是 String lockKey = "ticket";
// internalLockLeaseTime传入ARGV[1]，表示锁的超时时间，默认是30秒
// getLockName(threadId)传入ARGV[2]，表示锁的唯一标识线程id
```

设置监听器方法：方法名 tryAcquireOnceAsync(long leaseTime, TimeUnit unit, final long threadId)。

```
//设置监听器方法：
  private RFuture<Boolean> tryAcquireOnceAsync(long leaseTime, TimeUnit unit, final long threadId) {
        if (leaseTime != -1) {
            return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
        }
   //加锁成功这里会返回一个null值，即ttlRemainingFuture为null
   //若线程没有加锁成功，那么这里返回的就是这个别的线程加过的锁的剩余的过期时间,即ttlRemainingFuture为过期时间
        RFuture<Boolean> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
        //如果还持有这个锁，则开启定时任务不断刷新该锁的过期时间
        //这里给当前业务加了个监听器
        ttlRemainingFuture.addListener(new FutureListener<Boolean>() {
            @Override
            public void operationComplete(Future<Boolean> future) throws Exception {
                if (!future.isSuccess()) {
                    return;
                }

                Boolean ttlRemaining = future.getNow();
                // lock acquired
                if (ttlRemaining) {
                    //定时任务执行方法
                    scheduleExpirationRenewal(threadId);
                }
            }
        });
        return ttlRemainingFuture;
    }
```

定时任务执行方法:  方法名 scheduleExpirationRenewal(final long threadId)：

```
//定时任务执行方法
  private void scheduleExpirationRenewal(final long threadId) {
        if (expirationRenewalMap.containsKey(getEntryName())) {
            return;
        }

      //这里new了一个TimerTask()定时任务器
      //这里定时任务会推迟执行，推迟的时间是设置的锁过期时间的1/3,
      //很容易就能发现是一开始锁的过期时间默认值30s，具体可见private long lockWatchdogTimeout = 30 * 1000;
      //过期时间单位是秒
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                
                RFuture<Boolean> future = commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
             //这里又是一个lua脚本
             //这里lua脚本先判断了一下，Redis的key是否存在且设置key的线程id是否是参数ARGV[2]值
                        "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " + 
             //如果这个线程创建的Redis的key即锁仍然存在，那么久给锁的过期时间重新设值为internalLockLeaseTime，也就是初始值30s
                            "redis.call('pexpire', KEYS[1], ARGV[1]); " +
             //Redis的key过期时间重新设置成功后，这里的lua脚本返回的就是1
                            "return 1; " +
                        "end; " +
             //如果主线程已经释放了这个锁，那么这里的lua脚本就会返回0，直接结束“看门狗”的程序
                        "return 0;",
                          Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
                
                future.addListener(new FutureListener<Boolean>() {
                    @Override
                    public void operationComplete(Future<Boolean> future) throws Exception {
                        expirationRenewalMap.remove(getEntryName());
                        if (!future.isSuccess()) {
                            log.error("Can't update lock " + getName() + " expiration", future.cause());
                            return;
                        }
                        
                        if (future.getNow()) {
                            // reschedule itself
                            scheduleExpirationRenewal(threadId);
                        }
                    }
                });
            }
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);  
      

        if (expirationRenewalMap.putIfAbsent(getEntryName(), task) != null) {
            task.cancel();
        }
    }
```

```
//上面源码分析过了，当加锁成功后tryAcquireAsync()返回的值为null， 那么这个方法的返回值也为null
private Long tryAcquire(long leaseTime, TimeUnit unit, long threadId) {
   return get(tryAcquireAsync(leaseTime, unit, threadId));
}
```

```
public void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException {
        //获得当前线程id
       long threadId = Thread.currentThread().getId();
       //由上面的源码分析可以得出，当加锁成功后，这个ttl就是null
       //若线程没有加锁成功，那么这里返回的就是这个别的线程加过的锁的剩余的过期时间
        Long ttl = tryAcquire(leaseTime, unit, threadId);
        // lock acquired
       //如果加锁成功后，这个ttl就是null，那么这个方法后续就不需要做任何逻辑
       //若没有加锁成功这里ttl的值不为null，为别的线程加过锁的剩余的过期时间，就会继续往下执行
        if (ttl == null) {
            return;
        }

        RFuture<RedissonLockEntry> future = subscribe(threadId);
        commandExecutor.syncSubscription(future);

        try {
        //若没有加锁成功的线程，会在这里做一个死循环，即自旋
            while (true) {
                //一直死循环尝试加锁，这里又是上面的加锁逻辑了
                ttl = tryAcquire(leaseTime, unit, threadId);
                // lock acquired
                if (ttl == null) {
                    break;
                }
    //这里不会疯狂自旋，这里会判断锁失效之后才会继续进行自旋，这样可以节省一点CPU资源
                // waiting for message
                if (ttl >= 0) {
                    getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } else {
                    getEntry(threadId).getLatch().acquire();
                }
            }
        } finally {
            unsubscribe(future, threadId);
        }
  //        get(lockAsync(leaseTime, unit));
    }
```

**Redison 底层解锁源码分析：**

```
@Override
    public void unlock() {
      // 调用异步解锁方法
        Boolean opStatus = get(unlockInnerAsync(Thread.currentThread().getId()));
        //当释放锁的线程和已存在锁的线程不是同一个线程，返回null
        if (opStatus == null) {
            throw new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                    + id + " thread-id: " + Thread.currentThread().getId());
        }
        //根据执行lua脚本返回值判断是否取消续命订阅
        if (opStatus) {
          // 取消续命订阅
            cancelExpirationRenewal();
        }
    }
```

```
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                //如果锁已经不存在， 发布锁释放的消息，返回1
        "if (redis.call('exists', KEYS[1]) == 0) then " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; " +
                "end;" +
                //如果释放锁的线程和已存在锁的线程不是同一个线程，返回null
                "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                    "return nil;" +
                "end; " +
                //当前线程持有锁，用hincrby命令将锁的可重入次数-1，即线程id的value值-1
                "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                //若线程id的value值即可重入锁的次数大于0 ，就更新过期时间，返回0
                "if (counter > 0) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                    "return 0; " +
                //否则证明锁已经释放，删除key并发布锁释放的消息，返回1
                "else " +
                    "redis.call('del', KEYS[1]); " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; "+
                "end; " +
                "return nil;",
                Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(threadId));

    }
  // getName()传入KEYS[1]，表示传入解锁的keyName
  // getChannelName()传入KEYS[2]，表示redis内部的消息订阅channel
  // LockPubSub.unlockMessage传入ARGV[1]，表示向其他redis客户端线程发送解锁消息
  // internalLockLeaseTime传入ARGV[2]，表示锁的超时时间，默认是30秒
  // getLockName(threadId)传入ARGV[3]，表示锁的唯一标识线程id
```

```
void cancelExpirationRenewal() {
      // 将该线程从定时任务中删除
        Timeout task = expirationRenewalMap.remove(getEntryName());
        if (task != null) {
            task.cancel();
        }
    }
```

上述情况如果是单台 Redis，那么利用 Redison 开源框架实现 Redis 的分布式锁已经很完美了，但是往往生产环境的的 Redis 一般都是哨兵主从架构，Redis 的主从架构有别与 Zookeeper 的主从，客户端只能请求 Redis 主从架构的 Master 节点，Slave 节点只能做数据备份，Redis 从 Master 同步数据到 Slave 并不需要同步完成后才能继续接收新的请求，那么就会存在一个主从同步的问题。

当 Redis 的锁设置成功，正在执行业务代码，当 Redis 向从服务器同步时，Redis 的 Maste 节点宕机了，Redis 刚刚设置成功的锁还没来得及同步到 Slave 节点，那么此时 Redis 的主从哨兵模式就会重新选举出新的 Master 节点，那么这个新的 Master 节点其实就是原来的 Slave 节点，此时后面请求进来的线程都会请求这个新的 Master 节点，然而选举后产生的新 Master 节点实际上是没有那把锁的，那么从而导致了锁的失效。

![](https://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74B6eN0ZqvvMibcS9HYWaZiaUSmka3ics0rkZIvYq7iasHeN1AYIU7woZJh84jqVz5KtHRqibbbPyoUaSMA/640?wx_fmt=png)

上述问题用 Redis 主从哨兵架构实现的分布式锁在这种极端情况下是无法避免的，但是一般情况下生产上这种故障的概率极低，即使偶尔有问题也是可以接受的。

如果想使分布式锁变的百分百可靠，那可以选用 Zookeeper 作为分布式锁，就能完美的解决这个问题。由于 zk 的主从数据同步有别与 Redis 主从同步，zk 的强一致性使得当客户端请求 zk 的 Leader 节点加锁时，当 Leader 将这个锁同步到了 zk 集群的大部分节点时，Leader 节点才会返回客户端加锁成功，此时当 Leader 节点宕机之后，zk 内部选举产生新的 Leader 节点，那么新的客户款访问新的 Leader 节点时，这个锁也会存在，所以 zk 集群能够完美解决上述 Redis 集群的问题。

由于 Redis 和 Zookeeper 的设计思路不一样，任何分布式架构都需要满足 CAP 理论，“鱼和熊掌不可兼得”，要么选择 AP 要么选择 CP，很显然 Redis 是 AP 结构，而 zk 是属于 CP 架构，也导致了两者的数据同步本质上的区别。

其实设计 Redis 分布式锁有种 RedLock 的思想就是借鉴 zk 实现分布式锁的这个特点，这种 Redis 的加锁方式在 Redison 框架中也有提供 api，具体使用也很简单，这里就不一一赘述了。其主要思想如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74B6eN0ZqvvMibcS9HYWaZiaUSF5QS0HdR9HMsfBsodTTyIebjGruUp4U6LSgp2QE1EVn2WicicCDEzVMA/640?wx_fmt=png)

这种实现方式，我认为生产上并不推荐使用。很简单原本只需要对一个 Redis 加锁，设置成功返回即可，但是现在需要对多个 Redis 进行加锁，无形之中增加了好几次网络 IO，万一第一个 Redis 加锁成功后，后面几个 Redis 在加锁过程中出现了类似网络异常的这种情况，那第一个 Redis 的数据可能就需要做数据回滚操作了，那为了解决一个极低概率发生的问题又引入了多个可能产生的新问题，很显然得不偿失。并且这里还有可能出现更多乱七八糟的问题，所以我认为这种 Redis 分布式锁的实现方式极其不推荐生产使用。

退一万说如果真的需要这种强一致性的分布式锁的话，那为什么不直接用 zk 实现的分布式锁呢，性能肯定也比这个 RedLock 的性能要好。

**3. 分布式锁使用场景**
---------------

这里着重讲一下分布式锁的两种以下使用场景：

### **3.1 热点缓存 key 重建优化**

一般情况下互联网公司基本都是使用 “缓存” 加过期时间的策略，这样不仅加快数据读写， 而且还能保证数据的定期更新，这种策略能够满足大部分需求，但是也会有一种特殊情况会有问题：原本就存在一个冷门的 key，因为某个热点新闻的出现，突然这个冷门的 key 请求量暴增成了使其称为了一个热点 key，此时缓存失效，并且又无法在很短时间内重新设置缓存，那么缓存失效的瞬间，就会有大量线程来访问到后端，造成数据库负载加大，从而可能会让应用崩溃。

例如：“Air Force one”原本就是一个冷门的 key 存在于缓存中，微博突然有个明星穿着 “Air Force one” 上了热搜，那么就会有很多明星的粉丝来得物 app 购买 “Air Force one”，此时的“Air Force one” 就直接成为了一个热点 key，那么此时 “Air Force one” 这个 key 如果缓存恰好失效了之后，就会有大量的请求同时访问到 db，会给后端造成很大的压力，甚至会让系统宕机。

要解决这个问题只需要用一个简单的分布式锁即可解决这个问题，只允许一个线程去重建缓存，其他线程等待重建缓存的线程执行完， 重新从缓存获取数据即可。可见下面的实例伪代码：

// 分布式锁解决热点缓存，代码如下：

```
public String getCache(String key) {
        //从缓存获取数据
        String value = stringRedisTemplate.opsForValue().get(key);
        //传入Redis的key
        try {
            if (Objects.isNull(value)) {
                //这里只允许一个线程进入，重新设置缓存
                String mutexKey = key;
                //从db 获取数据
                value = mysql.getDataFromMySQL();
                //写回缓存
                stringRedisTemplate.opsForValue().setIfPresent(mutexKey, "poizon", 60, TimeUnit.SECONDS);
                //删除key
                stringRedisTemplate.delete(mutexKey);
            } else {
                Thread.sleep(100);
                getCache(key);
            }

        } catch (InterruptedException e) {
            log.error("getCache is error", e);
        }
        return value;
    }
```

### **3.2 解决缓存与数据库数据不一致问题**

如果业务对数据的缓存与数据库需要强一致时，且并发量不是很高的情况下的情况下时，就可以直接加一个分布式读写锁就可以直接解决这个问题了。可以直接利用可以加分布式读写锁保证并发读写或写写的时候按顺序排好队，读读的时候相当于无锁。

并发量不是很高且业务对缓存与数据库有着强一致对要求时，通过这种方式实现最简单，且效果立竿见影。倘若在这种场景下，如果还监听 binlog 通过消息的方式延迟双删的方式去保证数据一致性的话，引入了新的中间件增加了系统的复杂度，得不偿失。

### **3.3 超高并发场景下的分布式锁设计理论**

与 ConcurrentHashMap 的设计思想有点类似，用分段锁来实现，这个是之前在网上看到的实现思路，本人并没有实际使用过，不知道水深不深，但是可以学习一下实现思路。

假如 A 商品的库存是 2000 个，现在可以将该 A 商品的 2000 个库存利用类似 ConcurrentHashMap 的原理将不同数量段位的库存的利用取模或者是 hash 算法让其扩容到不同的节点上去，这样这 2000 的库存就水平扩容到了多个 Redis 节点上，然后请求 Redis 拿库存的时候请求原本只能从一个 Redis 上取数据，现在可以从五个 Redis 上取数据，从而可以大大提高并发效率。

![](https://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74B6eN0ZqvvMibcS9HYWaZiaUSeKj1wibtG5DyNpFK6lfVrvv8k0FU0erejpqmMMeUAwSiaOcibDl3wIqnQ/640?wx_fmt=png)

**4. 总结与思考**
------------

综上可知，Redis 分布式锁并不是绝对安全，Redis 分布式锁在某种极端情况下是无法避免的，但是一般情况下生产上这种故障的概率极低，即使偶尔有问题也是可以接受。

CAP 原则指的是在一个分布式系统中，一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）这三个要素最多只能同时实现两点，不可能三者兼顾。鱼和熊掌不可兼得”，要么选择 AP 要么选择 CP，选择 Redis 作为分布式锁的组件在于其单线程内存操作效率很高，且在高并发场景下也可以保持很好的性能。

如果一定要要求分布式锁百分百可靠，那可以选用 Zookeeper 或者 MySQL 作为分布式锁，就能完美的解决锁安全的问题，但是选择了一致性那就要失去可用性，所以 Zookeeper 或者 MySQL 实现的分布式锁的性能远不如 Redis 实现的分布式锁。

最后着重感谢组内同学对本篇 blog 的建议与支持，若有不足的地方烦请指出，大家可以一起探讨学习，共同进步。😁😁

![](https://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74B6eN0ZqvvMibcS9HYWaZiaUS0bjUXeXE5xJhN4ldJDSPcmnRDUWDGneH2aBxP7a8qTtSTVr2gredOQ/640?wx_fmt=png)

 *** 文** **/harmony**

 关注得物技术，每周一三五晚 18:30 更新技术干货  
要是觉得文章对你有帮助的话，欢迎评论转发点赞～

 ![](http://mmbiz.qpic.cn/mmbiz_png/AAQtmjCc74Diclic7XyegA4UmC0SqoT4jDEpdzibjkibmAvz8svbJwfsufiaBqOWx9sskIrickxzGfCwkjuMBiaNLDxNA/0?wx_fmt=png) ** 得物技术 ** 技术知识分享交流平台，与你一同走向技术的云端。 158 篇原创内容  公众号

  
上期（0422）活动中奖者名单公示：  
得物文化 T 恤：.w  
随行杯：树 * 静  
得物渔夫帽：纳 * 斯  
（注意：由于上海目前快递不支持发货，预计发出时间为 5 月底，如有疑问，可以微信公众号后台留言）