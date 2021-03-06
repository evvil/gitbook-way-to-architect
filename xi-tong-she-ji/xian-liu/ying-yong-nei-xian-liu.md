# 应用内限流

应用级限流主要可以分为以下几种：

1. 限制整个应用的并发请求数
2. 限制总资源数
3. 限制某个接口的总请求数
4. 限制某个接口的时间窗请求数
5. 平滑某个接口的请求

## 限制整个应用的并发请求数

对于一个应用而言，一定会有极限并发请求数，即总有一个TPS/QPS阈值。如果并发请求数量超过了阈值，应用就会不响应请求或响应地很慢，因此我们要对应用进行过载保护，防止大并发请求突然涌入击垮应用。

如果使用Tomcat作为应用服务器，它的Connector中有几下几个配置项：

* **acceptCount**：如果tomcat线程都忙于响应，则新来的连接会进入队列排队；如果超过队列大小，则拒绝连接；该参数就是指允许的队列的大小。
* **maxConnections**：瞬时最大连接数，超出的请求会排队等候。
* **maxThreads**：tomcat能启动用来处理请求的最大线程数，其实就是Tomcat允许的最大并发请求数。如果请求处理量一直大于该值，则会引起响应变慢甚至僵死。

## 限制总资源数

对于数据库连接、线程等稀有资源，可以使用池化技术来限制总资源数，如连接池、线程池。

## 限制某个接口的总请求数

如果某个接口可能有突发访问情况，但又担心并发请求太高造成崩溃，那么这个时候可以对这个接口进行总请求数的限制。

这种场景主要是：

* 抢购：超过限额，则让用户等待或告知用户无货
* 开放平台：限制用户某个接口的试用请求量

可以使用计数器算法来解决该问题，参考代码如下：

```java
public class Counter {

    /**
     * 请求数量计数器
     */
    private AtomicLong counter = new AtomicLong(0);
    /**
     * 请求数量限制量
     */
    private final long limit;

    public Counter(long limit){
        this.limit = limit;
    }

    public boolean grant(){
        return counter.incrementAndGet() <= limit;
    }
}
```

测试代码

```java
@Test
void grant() throws InterruptedException {
    Counter counter = new Counter(10);
    for (int i = 0; i < 30; i++) {
        if(counter.grant()){
            System.out.println("permit:::request:::" + i);
            Thread.sleep(100);
        }else {
            System.out.println("reject:::request:::" + i);
        }
    }
}

@Test
void grant2() throws InterruptedException {
    Counter counter = new Counter(10);
    ExecutorService service = Executors.newFixedThreadPool(3);
    for (int i = 0; i < 30; i++) {
        int finalI = i;
        service.submit(()->{
            if(counter.grant()){
                System.out.println("permit:::request:::" + finalI);
                try {
                    Thread.sleep(new Random().nextInt(500));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }else {
                System.out.println("reject:::request:::" + finalI);
            }
        });
    }

    Thread.sleep(1000);
}
```

 

## 限制某个接口的时间窗请求数

某个接口的时间窗请求数，是指某个接口在每秒或每分钟或每天的请求量。主要场景是：一些基础服务会被其他系统调用，但是调用量过大会压垮该基础服务，此时就要对某一时间窗口内的请求量进行限制。

参考代码如下：

```java
public class PeriodCounter {

    /**
     * 每秒限制的请求数
     */
    private final long limit;

    /**
     * 键：当前时间，秒
     * 值：该秒内的累计请求量
     */
    LoadingCache<Long, AtomicLong> secondCounter = CacheBuilder.newBuilder()
            //写入2秒后删除
            .expireAfterWrite(2, TimeUnit.MINUTES)
            .build(new CacheLoader<Long, AtomicLong>() {
                @Override
                public AtomicLong load(Long aLong) throws Exception {
                    //重新获取初始值为0
                    return new AtomicLong(0);
                }
            });

    public PeriodCounter(long limit){
        this.limit = limit;
    }

    public boolean permit() throws ExecutionException {
        while (true){
            //获取当前秒
            long currentSecond = System.currentTimeMillis() / 1000;
            System.out.println(currentSecond);
            if(secondCounter.get(currentSecond).incrementAndGet() > limit){
                return false;
            }else {
                return true;
            }
        }
    }
}
```

 测试代码

```java
@Test
void permit() throws ExecutionException, InterruptedException {
    //每秒只允许访问两次
    PeriodCounter counter = new PeriodCounter(2);
    for (int i = 0; i < 100; i++) {
        Thread.sleep(new Random().nextInt(1000));
        if(counter.permit()){
            System.out.println("permit:::request:::" + i);
        }else {
            System.out.println("reject:::request:::" + i);
        }
    }
}

@Test
void permit2() throws InterruptedException {
    //每秒只允许访问两次
    PeriodCounter counter = new PeriodCounter(2);
    ExecutorService service = Executors.newFixedThreadPool(3);
    for (int i = 0; i < 100; i++) {
        Thread.sleep(new Random().nextInt(1000));
        int finalI = i;
        service.submit(()->{
            try {
                if(counter.permit()){
                    System.out.println("permit:::request:::" + finalI);
                    try {
                        Thread.sleep(new Random().nextInt(500));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }else {
                    System.out.println("reject:::request:::" + finalI);
                }
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
    }

    Thread.sleep(1000);
}
```

 TODO：多时间窗口限制的代码实现

## 平滑某个接口的请求

上述的几种限流方式都不能很好地应对突发请求：高并发请求可能被允许，从而导致一些问题。在一些场景中，需要对突发流量进行整形，整形为平均速率请求进行处理（比如5r/s，每秒钟处理5次请求，则每隔200毫秒处理一个请求，让速率平滑）。漏桶算法和令牌桶算法都可以满足此类场景。Google的Guava框架中提供了令牌桶算法（RateLimiter），可以直接拿来使用。

RateLimiter提供的令牌桶算法可用于平滑突发流量（SmoothBursty）和平滑预热流量（SmoothWarmingUp）。

### 平滑突发流量

```java
/**
 * 平滑突发流量
 *
 * RateLimiter.create(5)：create(double permitsPerSecond)，表示桶的大小为5，每隔200毫秒新增一个令牌；
 * limiter.acquire()：表示消费1个令牌。
 *  如果当前桶中有足够令牌，则成功（返回值为0）；
 *  如果当前桶中没有令牌，则等待，直到产生足够令牌，才从该方法中返回（返回值为等待时间）；
 *      比如，当令牌发放间隔是200毫秒，则等待200毫秒才能拿到令牌，。
 */
@Test
public void test1() {
    RateLimiter limiter = RateLimiter.create(5);
    for (int i = 0; i < 8; i++) {
        double acquire = limiter.acquire();
        System.out.println("wait-time-in-second:::" + acquire);
    }
}
//输出如下
wait-time-in-second:::0.0 
wait-time-in-second:::0.195944 
wait-time-in-second:::0.195802 
wait-time-in-second:::0.197988 
wait-time-in-second:::0.195698 
wait-time-in-second:::0.196579 
wait-time-in-second:::0.195072 
wait-time-in-second:::0.196727
```

RateLimiter允许有一定程度地突发流量，即可以一次性消费多个令牌；也允许消费未来的令牌，但接下来的下一次消费，则要等待相应时间才能获取到令牌，演示代码如下：

```java
//一次性消费消费多个令牌示例
@Test
public void test2() {
    RateLimiter limiter = RateLimiter.create(5);
    System.out.println("wait-time-in-second:::" + limiter.acquire(5));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
}
//输出如下：
wait-time-in-second:::0.0 
wait-time-in-second:::0.997478 //这里需要等到一秒
wait-time-in-second:::0.193398 
wait-time-in-second:::0.196968 
wait-time-in-second:::0.198204 
wait-time-in-second:::0.196326 
wait-time-in-second:::0.197762

//消费未来的令牌
@Test
public void test3() {
    RateLimiter limiter = RateLimiter.create(5);
    System.out.println("wait-time-in-second:::" + limiter.acquire(10));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
    System.out.println("wait-time-in-second:::" + limiter.acquire(1));
}
//输出如下
wait-time-in-second:::0.0 
wait-time-in-second:::1.996424 //这里需要等待两秒
wait-time-in-second:::0.190973 
wait-time-in-second:::0.196195 
wait-time-in-second:::0.196406 
wait-time-in-second:::0.197421 
wait-time-in-second:::0.198815
```

 此外，RateLimiter中还有一个最大突发秒数（`maxBurstSeconds`）的参数，默认值为1，它的含义是：如果RateLimiter在一段时间内没有使用，当流量到来时，允许使用前`maxBurstSeconds`时间内产生的令牌。原文释义如下：

> The work \(permits\) of how many seconds can be saved up if this RateLimiter is unused

 即RateLimiter允许将一段时间内没有消费的令牌暂存起来，保留待未来使用。

举个例子：

```java
@Test
public void test4() throws InterruptedException {
    RateLimiter limiter = RateLimiter.create(2);
    System.out.println("wait-time-in-second:::" + limiter.acquire());//①此时消费1个令牌，无需等待，返回值为0
    Thread.sleep(2000);//暂停2s，
    // 因为maxBurstSeconds默认为1s，所以，②处获取令牌时，桶中实际有3个令牌，②③④处均无需等待，返回值为0
    System.out.println("wait-time-in-second:::" + limiter.acquire());//②
    System.out.println("wait-time-in-second:::" + limiter.acquire());//③
    System.out.println("wait-time-in-second:::" + limiter.acquire());//④
    System.out.println("wait-time-in-second:::" + limiter.acquire());//等待大约500毫秒
    System.out.println("wait-time-in-second:::" + limiter.acquire());//等待大约500毫秒
    System.out.println("wait-time-in-second:::" + limiter.acquire());//等待大约500毫秒
}
//输出如下：
wait-time-in-second:::0.0 
wait-time-in-second:::0.0 
wait-time-in-second:::0.0 
wait-time-in-second:::0.0 
wait-time-in-second:::0.499828 
wait-time-in-second:::0.492934 
wait-time-in-second:::0.497183
```

 此外，RateLimiter还提供了`tryAcquire()`这种无阻塞或可超时的令牌消费方式。

### 平滑预热流量

因为SmoothBurst允许一定程度的突发流量，假如突然来了太多的流量，系统可能扛不住。因此，RateLimiter也提供了相应的应付工具：SmoothWarmingUp，它会控制流量从0流量和最大流量上升的速率，即刚开始速率小一些，然后慢慢趋于我们设定的允许的最大速率。

```text
public static RateLimiter create(double permitsPerSecond, 
            long warmupPeriod, //预热期时间
            TimeUnit unit)
```

 举个例子：

```java
@Test
public void test5() throws InterruptedException {
    RateLimiter limiter = RateLimiter.create(5,1, TimeUnit.SECONDS);
    for (int i = 0; i < 10; i++) {
        double acquire = limiter.acquire();
        System.out.println("wait-time-in-second:::" + acquire);
    }
}
//输出如下
wait-time-in-second:::0.0 
wait-time-in-second:::0.516669 
wait-time-in-second:::0.352466 
wait-time-in-second:::0.216223 
wait-time-in-second:::0.197775 
wait-time-in-second:::0.195669 
wait-time-in-second:::0.194383 
wait-time-in-second:::0.197728 
wait-time-in-second:::0.195479 
wait-time-in-second:::0.195817
```

##  总结

应用级限流只是单应用内的请求限流，假如将应用部署到多台机器，就不能使用这种方式进行先限流了。因此，我们需要使用分布式限流和接入层限流来解决该问题。

## 参考

《亿级流量网站架构核心技术》：限流详解

