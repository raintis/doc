__简介__

Google Guava library中提供了RateLimiter类，它经常用于限制对一些物理资源或者逻辑资源的访问速率。与Semaphore 相比，Semaphore 限制了并发访问的数量而不是使用速率。

RateLimiter类定义如下：
```java
com.google.common.util.concurrent.RateLimiter

@ThreadSafe
@Beta
public abstract class RateLimiter
extends Object
```

Guava Docs关于RateLimiter介绍如下：
```java
A rate limiter. Conceptually, a rate limiter distributes permits at a configurable rate. Each acquire() blocks if necessary until 
a permit is available, and then takes it. Once acquired, permits need not be released.

Rate limiters are often used to restrict the rate at which some physical or logical resource is accessed. This is in contrast to 
Semaphore which restricts the number of concurrent accesses instead of the rate (note though that concurrency and rate are closely 
related, e.g. see Little’s Law).

A RateLimiter is defined primarily by the rate at which permits are issued. Absent additional configuration, permits will be 
distributed at a fixed rate, defined in terms of permits per second. Permits will be distributed smoothly, with the delay between 
individual permits being adjusted to ensure that the configured rate is maintained.

It is possible to configure a RateLimiter to have a warmup period during which time the permits issued each second steadily 
increases until it hits the stable rate.
```

通过设置许可证的速率来定义RateLimiter。在默认配置下，许可证会在固定的速率下被分配，速率单位是每秒多少个许可证。为了确保维护配置的速率，许可会被平稳地分配，许可之间的延迟会做调整。
可能存在配置一个拥有预热期的RateLimiter 的情况，在这段时间内，每秒分配的许可数会稳定地增长直到达到稳定的速率。

原理 
RateLimiter使用的是一种叫令牌桶的流控算法，RateLimiter会按照一定的频率往桶里扔令牌，线程拿到令牌才能执行，比如你希望自己的应用程序QPS不要超过1000，那么RateLimiter设置1000的速率后，就会每秒往桶里扔1000个令牌。

用法
举例来说明如何使用RateLimiter，想象下我们需要处理一个任务列表，但我们不希望每秒的任务提交超过两个：
```java
final RateLimiter rateLimiter = RateLimiter.create(2.0); // rate is "2 permits per second"
  void submitTasks(List<Runnable> tasks, Executor executor) {
    for (Runnable task : tasks) {
      rateLimiter.acquire(); // may wait
      executor.execute(task);
    }
  }
```
再举另外一个例子，想象下我们制造了一个数据流，并希望以每秒5kb的速率处理它。可以通过要求每个字节代表一个许可，然后指定每秒5000个许可来完成：
```java
  final RateLimiter rateLimiter = RateLimiter.create(5000.0); // rate = 5000 permits per second
  void submitPacket(byte[] packet) {
    rateLimiter.acquire(packet.length);
    networkService.send(packet);
  }
```
最后，在RateLimiter的doc中有这么一句话
```java
It is important to note that the number of permits requested never affect the throttling of the request 
itself (an invocation to acquire(1) and an invocation to acquire(1000) will result in exactly the same throttling, if any), 
but it affects the throttling of the next request. I.e., if an expensive task arrives at an idle RateLimiter, it will be granted 
immediately, but it is the next request that will experience extra throttling, thus paying for the cost of the expensive task.

Note: RateLimiter does not provide fairness guarantees.
```
翻译如下： 
有一点很重要，那就是请求的许可数从来不会影响到请求本身的限制（调用acquire(1) 和调用acquire(1000) 将得到相同的限制效果，如果存在这样的调用的话），但会影响下一次请求的限制，也就是说，如果一个高开销的任务抵达一个空闲的RateLimiter，它会被马上许可，但是下一个请求会经历额外的限制，从而来偿付高开销任务。注意：RateLimiter 并不提供公平性的保证。

应用
我们可以轻松地借助 Guava 的 RateLimiter 来实现流量控制。 
例如，公司需要上线某个业务，合作方提供API接口来获取业务数据，不过考虑到服务器负载压力合作方对API接口调用做限制：要求QPS最高不能超过60。API调用代码实现如下：

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import com.google.common.util.concurrent.RateLimiter;

public class ApiCallDemo {

    private int permitsPerSecond = 55; //每秒55个许可
    private int threadNum = 10;

    public static void main(String[] args) {

        new ApiCallDemo().call();
    }

    private void call() {

        ExecutorService executor = Executors.newFixedThreadPool(threadNum);
        final RateLimiter rateLimiter = RateLimiter.create(permitsPerSecond); 
        for (int i=0; i<threadNum; i++) {
            executor.execute(new ApiCallTask(rateLimiter));
        }

        executor.shutdown();
    }
}

class ApiCallTask implements Runnable{
    private RateLimiter rateLimiter;
    private boolean runing = true;
    public ApiCallTask(RateLimiter rateLimiter) {
        this.rateLimiter = rateLimiter;
    }

    @Override
    public void run() {

        while(runing){
            rateLimiter.acquire(); // may wait
            getData();
        }
    }

    /**模拟调用合作伙伴API接口*/
    private void getData(){
        System.out.println(Thread.currentThread().getName()+" runing!");
        try {
            TimeUnit.MILLISECONDS.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
