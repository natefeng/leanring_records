# SpringCloud

## 限流算法

###  计数器固定窗口算法

**优点：实现简单，容易理解**

**缺点：流量曲线不够平滑，会存在一些问题**

- **一段时间内(不超过时间窗口)服务不可用。**比如窗口大小为1s，限流大小为100，如果初始1ms内请求达到了100，那么剩下的2ms~999ms此时的服务都是不可用的。
- **窗口切换的时候可能会产生两倍于阈值流量的请求**。比如窗口大小为1s，限流大小为100，然后恰好在某个窗口的第999ms来了100个请求，并且再恰好，下一个窗口的第一个1ms来了100个请求，也全部通过。那么就是在2ms内通过了200个请求，而我们设置的阈值是100，通过的请求达到了阈值的两倍。

```java
package limit;


import java.util.concurrent.atomic.AtomicInteger;

public class CurrentLimit {

    private int windowSize;

    private int limit;

    private AtomicInteger count;

    public CurrentLimit(){

    }
    public CurrentLimit(int windowSize,int limit){
        this.windowSize = windowSize;
        this.limit = limit;
        count = new AtomicInteger();
        Thread thread = new Thread(() -> {
            while (true){
                count.set(0);
                try {
                    Thread.sleep(windowSize);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        thread.setDaemon(true);
        thread.start();
    }

    public boolean tryAcquire() throws InterruptedException {
            int newCount = count.addAndGet(1);
            if (newCount > limit){
               return false;
            }
            return true;
    }

    public static void main(String[] args) throws InterruptedException {
        CurrentLimit currentLimit = new CurrentLimit(1000,20);
        int count = 0;
        for (int i = 0; i < 50 ; i++) {
            if (currentLimit.tryAcquire()){
                count++;
            }
        }
        System.out.println("第一次50次请求中通过" + count +"限流" + (50 - count));
        Thread.sleep(1000);

        count = 0;
        for (int i = 0; i < 50 ; i++) {
            if (currentLimit.tryAcquire()){
                count++;
            }
        }
        System.out.println("第二次50次请求中通过" + count +"限流" + (50 - count));
    }


}

```

### 计数器滑动窗口算法

### 漏斗算法

1. **漏桶的漏出速率是固定的，可以起到整流的作用。**及时出现请求的流量随机性，忽大忽小，但是经过漏斗算法以后，有稳定的处理速度，对系统的稳定性提高。

2. **不能解决流量突发问题**，刚刚测试的例子，漏斗速率为2个/秒，那么突然来10个请求，限于漏斗的容量，只会有5个被接受，另外5个被拒绝。

```java
package limit;

import java.util.Date;
import java.util.LinkedList;

public class LeakyBucketLimiter {

    //漏斗容量
    private int capacity;

    //漏斗速率
    private int rate;
    //剩余容量
    private int left;
    private LinkedList<Request> requestLinkedList;

    private LeakyBucketLimiter(){
    }

    public static void main(String[] args) {
        LeakyBucketLimiter leakyBucketLimiter = new LeakyBucketLimiter(5,2);
        for (int i = 0; i <= 10; i++) {
            Request request = new Request(i, new Date());
            if (leakyBucketLimiter.tryAcquire(request)){
                System.out.println(i + "号请求被接受");
            }else {
                System.out.println(i + "号请求被拒绝");
            }
        }
    }

    private boolean tryAcquire(Request request) {
        if (left <= 0){
            return false;
        }
        left --;
        requestLinkedList.add(request);
        return true;
    }

    public LeakyBucketLimiter(int capacity,int rate){
       this.capacity = capacity;
       this.rate = rate;
       this.left = capacity;
       requestLinkedList = new LinkedList<>();
        Thread thread = new Thread(() -> {
            while (true){
               if (!requestLinkedList.isEmpty()){
                   Request request = requestLinkedList.removeFirst();
                   handleRequest(request);
                   left ++;
               }
               try {
                   Thread.sleep(1000 / rate);
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
            }
        });
        thread.start();
    }

    private void handleRequest(Request request) {
        request.setHandleTime(new Date());
        System.out.println(request.getCode() + "号请求被处理，请求发起时间："
                + request.getLaunchTime() + ",请求处理时间：" + request.getHandleTime() + ",处理耗时："
                + (request.getHandleTime().getTime()  - request.getLaunchTime().getTime()) + "ms");
    }


    static class Request{
        private int code;
        private Date launchTime;
        private Date handleTime;

        private Request(){

        }
        public Request(int code,Date launchTime){
            this.launchTime = launchTime;
            this.code = code;
        }
        public int getCode(){
            return code;
        }
        public void setCode(int code) {
            this.code = code;
        }
        public Date getLaunchTime() {
            return launchTime;
        }

        public void setLaunchTime(Date launchTime) {
            this.launchTime = launchTime;
        }

        public Date getHandleTime() {
            return handleTime;
        }

        public void setHandleTime(Date handleTime) {
            this.handleTime = handleTime;
        }

    }

}

```

### 令牌桶算法

**允许一定的流量并发**。令牌桶算法会马上处理请求，而漏斗则是不会立刻处理请求。即可以允许一定的并发

```java
package limit;

import java.util.Date;

public class TokenBucketLimiter {

    private int capacity;
    private int rate;
    private int tokenAmount;

    public TokenBucketLimiter(int capacity,int rate){
        this.capacity = capacity;
        this.rate = rate;
        tokenAmount = capacity;
        Thread thread = new Thread(() -> {
            while (true){
                synchronized (this){
                    tokenAmount++;
                    if (tokenAmount > capacity){
                        tokenAmount = capacity;
                    }
                }
                try {
                    Thread.sleep(1000 / rate);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        thread.setDaemon(true);
        thread.start();
    }

    public static void main(String[] args) {
        TokenBucketLimiter tokenBucketLimiter = new TokenBucketLimiter(5,1000);
        for (int i = 0; i <= 20 ; i++) {
            Request request = new Request(i, new Date());
            if (tokenBucketLimiter.tryAcquire(request)){
                System.out.println(i + "号请求被接受");
            }else {
                System.out.println(i+" 号请求被拒绝");
            }
        }
    }
    public synchronized boolean tryAcquire(Request request){
        if (tokenAmount > 0){
            tokenAmount--;
            handleRequest(request);
            return true;
        }
        return false;
    }
    private void handleRequest(Request request){
        request.setHandleTime(new Date());
        System.out.println(request.getCode() + "号请求被处理，请求发起时间："
                + request.getLaunchTime() + ",请求处理时间：" + request.getHandleTime() + ",处理耗时："
                + (request.getHandleTime().getTime()  - request.getLaunchTime().getTime()) + "ms");
    }


    static class Request{
        private int code;
        private Date launchTime;
        private Date handleTime;

        private Request(){

        }
        public Request(int code,Date launchTime){
            this.launchTime = launchTime;
            this.code = code;
        }
        public int getCode(){
            return code;
        }
        public void setCode(int code) {
            this.code = code;
        }
        public Date getLaunchTime() {
            return launchTime;
        }

        public void setLaunchTime(Date launchTime) {
            this.launchTime = launchTime;
        }

        public Date getHandleTime() {
            return handleTime;
        }

        public void setHandleTime(Date handleTime) {
            this.handleTime = handleTime;
        }

    }

}

```

