DelayQueue延迟队列理解： 
1、DelayQueue队列中的元素必须是Delayed接口的实现类，该类内部实现了getDelay()和compareTo()方法，第一个方法是比较两个任务的延迟时间进行排序，第二个方法用来获取延迟时间。 
2、DelayQueue队列没有大小限制，因此向队列插数据不会阻塞 
3、DelayQueue中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素。否则线程阻塞。 
4、DelayQueue中的元素不能为null 
5、DelayQueue内部是使用PriorityQueue实现的。compareTo()比较后越小的越先取出来。

我们用DelayQueue们模拟一个Session实现的场景。 
Session有以下特点： 
1、以唯一键key来插入和获取对象 
2、Session有自动过期时间，到期后系统会自动清理。 
3、每次获取session对象，该key值所在的对象生命周期重置，过期时间从当前时间开始重新计算。

__实现思路： __

1、对于特点1，采用hashmap来保存session存储对象 
2、对于特点2，3，利用DelayQueue延迟队列来实现： 
创建一个延迟队列ptrqueue，每当有session插入hashmap时，就同步往ptrqueue队列插入一个与session的key同名的指针对象（该指针实现了Delayed接口，通过key值指向hashmap中对应元素）；每当读取session操作时，就更新ptrqueue队列中对应指针的到期时间；专门开启一个守护线程(阻塞式)从ptrqueue队列中获取过期的指针，再根据指针删除hashmap中对应元素。 
这里写图片描述
![baidu](https://img-blog.csdn.net/20170228152614399?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc29vbmZseQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast "百度logo")
```java
public class DelayedDemo {

    public static void main(String[] args) throws InterruptedException {
        TSession sessionService=new TSession();
        sessionService.ConnectionAndStart();
        /*模拟客户端调用*/
        sessionService.put("userIdentity", "tangwenming");
        Thread.sleep(4000);
        sessionService.put("userGroup", "super");

        sessionService.get("userIdentity");

        sessionService.get("userGroup");
        Thread.sleep(2000);
        sessionService.get("userGroup");
        Thread.sleep(2000);
        sessionService.get("userGroup");
        Thread.sleep(2000);
        sessionService.get("userGroup");
        Thread.sleep(5500);
        sessionService.get("userGroup");
        sessionService.get("userIdentity");
    }

}
class TSession{
    /*从conf中获取session自动过期时间,单位：秒*/
    private static int liveTime=Integer.valueOf(getConfig("livetime"));
    /*指针保存队列*/
    DelayQueue<Itemptr> ptrqueue=new DelayQueue<Itemptr>();
    /*Session数据存储map*/
    public ConcurrentHashMap<String, Object> datapool = new ConcurrentHashMap<String, Object>();

    public void put(String key,Object value){
        /*插入session数据池*/
        datapool.put(key, value);
        /*插入对应key值的指针*/
        Itemptr ptr=new Itemptr(key,liveTime);
        ptrqueue.remove(ptr);/*更新过期时间step1*/
        ptrqueue.put(ptr);/*更新过期时间step2*/
        System.out.println("插入"+key+"："+value+",生命周期初始化："+liveTime+"秒");
    }
    public Object get(String key){
        Object resultObject= datapool.get(key);
        if(resultObject!=null){
            /*刷新对应key值的指针*/
            Itemptr ptr=new Itemptr(key,liveTime);
            ptrqueue.remove(ptr);
            ptrqueue.put(ptr);
            System.out.println("获取"+key+"成功："+resultObject+",生命周期重新计算");
        }else{
            /*从session池中返回对象*/
            System.out.println("获取"+key+"失败："+resultObject+"。对象已过期");
        }
        return resultObject;
    }
    private void sesseion_gc(){
        Itemptr ptr;
        while (true){
            try {
                /*阻塞线程等待直到获取超时的元素指针
                 *获取成功后从队列中删除节点
                  在while true循环块中确实比非阻塞式的poll节省资源*/
                ptr = ptrqueue.take();
                /*根据指针删除session对象*/
                datapool.remove(ptr.getKey());
                System.out.println("删除过期key="+ptr.getKey()+"的元素");
                /*降低cpu负担，根据业务需要和硬件调整*/
                Thread.sleep(300);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    private static String getConfig(String key){
        return "5";/*单位：秒*/
    }
    /*以守护进程运行gc回收方法*/
    public void ConnectionAndStart(){
        Thread sessionThread=new Thread(){
            @Override
            public void run(){
                sesseion_gc();
                }
        };
        sessionThread.setDaemon(true);
        sessionThread.start();
    }
}
class Itemptr implements Delayed{

    private String key;
    public String getKey() {
        return key;
    }

    private long liveTime ;
    private long removeTime;

    public long getRemoveTime() {
        return removeTime;
    }
    public Itemptr(String key,long liveTime){
        this.key=key;
        this.liveTime = liveTime;
        this.removeTime = TimeUnit.NANOSECONDS.convert(liveTime, TimeUnit.SECONDS) + System.nanoTime();
    }
    @Override
    public int compareTo(Delayed o) {
        if (o == null) return 1;
        if (o == this) return  0;
        if (o instanceof Itemptr){
            Itemptr ptr = (Itemptr)o;
            /*用过期时间排序，确定优先级。
             * DelayQueue按照升序（由小到大）排序的，也就是临近当前时间的优先出队*/
            if (removeTime > ptr.getRemoveTime() ) {
                return 1;
            }else if (removeTime == ptr.getRemoveTime()) {
                return 0;
            }else {
                return -1;
            }
        }
        return 1;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(removeTime - System.nanoTime(), TimeUnit.NANOSECONDS);
    }

    /*
     * 队列remove()判断时使用equals比较：指针队列只需要判断key字符相同即可
     * remove(Object o)
       * Removes a single instance of the specified element from this queue, if it is present, whether or not it has expired.
     */
    @Override
    public boolean equals(Object obj) {
        if (obj instanceof Itemptr) {
            if (obj==this)
                return true;
            return ((Itemptr)obj).getKey() == this.getKey() ?true:false;
        }
        return false;
    }
}
```
__输出：__

插入userIdentity：tangwenming,生命周期初始化：5秒 
插入userGroup：super,生命周期初始化：5秒 
获取userIdentity成功：tangwenming,生命周期重新计算 
获取userGroup成功：super,生命周期重新计算 
获取userGroup成功：super,生命周期重新计算 
获取userGroup成功：super,生命周期重新计算 
删除过期key=userIdentity的元素 
获取userGroup成功：super,生命周期重新计算 
删除过期key=userGroup的元素 
获取userGroup失败：null。对象已过期 
获取userIdentity失败：null。对象已过期

session依靠sessionID辨别客户端连接，每个sessionID创建的保存数据的hashmap和指针queue，都应该是独立的。本文主要阐述DelyaQueue的使用，为了降低程序复杂度，没有去实现该功能。有兴趣的可以去实现。
