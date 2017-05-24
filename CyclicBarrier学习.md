**1、类说明：**<br>
一个同步辅助类，它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)。在涉及一组固定大小的线程的程序中，这些线程必须不时地互相等待，此时 CyclicBarrier 很有用。因为该 barrier 在释放等待线程后可以重用，所以称它为循环 的 barrier。
**2、使用场景：**<br>
需要所有的子任务都完成时，才执行主任务，这个时候就可以选择使用CyclicBarrier。
**3、常用方法：**<br>
```
await
public int await()
          throws InterruptedException,
                 BrokenBarrierException
在所有参与者都已经在此 barrier 上调用 await方法之前，将一直等待。如果当前线程不是将到达的最后一个线程，出于调度目的，将禁用它，且在发生以下情况之一前，该线程将一直处于休眠状态：
最后一个线程到达；或者
其他某个线程中断当前线程；或者
其他某个线程中断另一个等待线程；或者
其他某个线程在等待 barrier 时超时；或者
其他某个线程在此 barrier 上调用 reset()。
如果当前线程：
在进入此方法时已经设置了该线程的中断状态；或者
在等待时被中断
则抛出 InterruptedException，并且清除当前线程的已中断状态。如果在线程处于等待状态时 barrier 被 reset()，或者在调用 await 时 barrier 被损坏，抑或任意一个线程正处于等待状态，则抛出 BrokenBarrierException 异常。
如果任何线程在等待时被 中断，则其他所有等待线程都将抛出 BrokenBarrierException 异常，并将 barrier 置于损坏状态。
如果当前线程是最后一个将要到达的线程，并且构造方法中提供了一个非空的屏障操作，则在允许其他线程继续运行之前，当前线程将运行该操作。如果在执行屏障操作过程中发生异常，则该异常将传播到当前线程中，并将 barrier 置于损坏状态。
 
返回：
到达的当前线程的索引，其中，索引 getParties() - 1 指示将到达的第一个线程，零指示最后一个到达的线程
抛出：
InterruptedException - 如果当前线程在等待时被中断
BrokenBarrierException - 如果另一个 线程在当前线程等待时被中断或超时，或者重置了 barrier，或者在调用 await 时 barrier 被损坏，抑或由于异常而导致屏障操作（如果存在）失败。
```
**4、相关实例**<br>
赛跑时，等待所有人都准备好时，才起跑：
```java
public class CyclicBarrierTest {  
  
    public static void main(String[] args) throws IOException, InterruptedException {  
        //如果将参数改为4，但是下面只加入了3个选手，这永远等待下去  
        //Waits until all parties have invoked await on this barrier.   
        CyclicBarrier barrier = new CyclicBarrier(3);  
  
        ExecutorService executor = Executors.newFixedThreadPool(3);  
        executor.submit(new Thread(new Runner(barrier, "1号选手")));  
        executor.submit(new Thread(new Runner(barrier, "2号选手")));  
        executor.submit(new Thread(new Runner(barrier, "3号选手")));  
  
        executor.shutdown();  
    }  
}  

class Runner implements Runnable {  
    // 一个同步辅助类，它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)  
    private CyclicBarrier barrier;  
  
    private String name;  
  
    public Runner(CyclicBarrier barrier, String name) {  
        super();  
        this.barrier = barrier;  
        this.name = name;  
    }  
  
    @Override  
    public void run() {  
        try {  
            Thread.sleep(1000 * (new Random()).nextInt(8));  
            System.out.println(name + " 准备好了...");  
            // barrier的await方法，在所有参与者都已经在此 barrier 上调用 await 方法之前，将一直等待。  
            barrier.await();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        } catch (BrokenBarrierException e) {  
            e.printStackTrace();  
        }  
        System.out.println(name + " 起跑！");  
    }  
}  
```

**输出结果：**<br>
3号选手 准备好了...<br>
2号选手 准备好了...<br>
1号选手 准备好了...<br>
1号选手 起跑！<br>
2号选手 起跑！<br>
3号选手 起跑！<br>
