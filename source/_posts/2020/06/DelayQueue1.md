---
title: DelayQueue(jdk1.8)
date: 2020-06-30 10:46:17
tags:
- Java容器
categories:
- Java
- JUC
---

### DelayQueue简介

    DelayQueue(延迟队列)是java.util.concurrent包下的适用于一些非即时执行场景下的并发集合。
    数据以PriorityQueue的结构存储，借助ReentrantLock保证线程安全，使用Condition完成对线程
    的精确控制。
<!-- more -->

类定义如下：

{% codeblock lang:java %}
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> 
{% endcodeblock %}

  从类定义可以看到，队列中的元素对象都需要实现Delayed接口，通过实现Delayed的compareTo()和getDelay()方法实现元素的排序和取出消费的判断。
  而DelayQueue本身是BlockingQueue的一个实现，未到执行时间的元素对象不会被取出，而是阻塞当前线程让其等待至任务的执行时间。

### 属性信息

{% codeblock lang:java %}
// 可重入锁，用来保证集合操作的线程安全
private final transient ReentrantLock lock = new ReentrantLock();

// 队列数据用优先级队列存储
private final PriorityQueue<E> q = new PriorityQueue<E>();

// 当前线程
private Thread leader = null;

// Condition用来控制线程
private final Condition available = lock.newCondition();
{% endcodeblock %}

  这里定义的线程leader，参考的多线程的Leader/Follower模式设计。
  其思想是当有多个消费者线程去获取队列的元素对象时，同一个时刻只有一个线程成为leader等待队首对象，当取得队首对象时就通知其他的线程取代他成为leader等待下一个队首。

  Condition这里用来精确的控制线程，当等待的队首对象还未到执行时间时，会使用Condition的await()方法让当前线程等待。

### 核心方法

{% codeblock lang:java %}
// 入队方法
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        q.offer(e);// 调用PriorityQueue的入队方法
        if (q.peek() == e) {
            // 队首元素是新增的元素 唤醒等待线程来处理
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}


// 弹出队首元素   仍然是用ReenTrantLock保证线程安全
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E first = q.peek();
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return q.poll();
    } finally {
        lock.unlock();
    }
}


public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();// 加锁
    try {
        for (;;) {
            E first = q.peek();
            if (first == null)
                available.await();// 无队首表明队列为空 则让线程等待
            else {
                long delay = first.getDelay(NANOSECONDS);// 获取队首任务的剩余执行时间
                if (delay <= 0)
                    return q.poll();// 队首任务可以执行 弹出
                first = null; // don't retain ref while waiting
                // 任务还需等待，判断leader
                if (leader != null)
                    // leader不为空，则当前线程等待，由leader线程等待队首任务
                    available.await();
                else {
                	// leader为空，当前线程成为新的leader
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        // 设置leader线程的等待时间，确保队首任务执行的时间点就能唤醒继续处理
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            // 最后置空leader，避免线程处理任务的时候继续占用leader
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null)
        	// leader为空并且有待处理的任务，唤醒其他线程
            available.signal();
        lock.unlock();
    }
}
{% endcodeblock%}

  DelayQueue最重要的方法便是take()，消费者线程通过调用take()去依次取出队首任务进行处理。
  只要理解了Leader/Follower模式就不难理解take()的逻辑。

### 示例

  这里用一个单线程生产者/消费者的示例展示下DelayQueue的基本用法

{% codeblock lang:java %}
// DelayQueue队列元素类的定义
public class DelayTask implements Delayed {

    // 延迟时间
    private final Long delay;
    // 执行时间
    private final Long exprie;
    // 创建时间
    private final Long create;
    // 任务信息
    private final String msg;

    public DelayTask(Long delay, String msg) {
        this.delay = delay;
        this.create = System.currentTimeMillis();
        this.exprie = create + delay;
        this.msg = msg;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        long now = System.currentTimeMillis();
        return unit.convert(exprie - now, TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        long delay1 = getDelay(TimeUnit.SECONDS);
        long delay2 = o.getDelay(TimeUnit.SECONDS);
        return Long.compare(delay1, delay2);
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("delay:").append(getDelay(TimeUnit.SECONDS)).append("msg:").append(msg);
        return sb.toString();
    }
}

// Delayed接口的定义
public interface Delayed extends Comparable<Delayed> {

    /**
     * Returns the remaining delay associated with this object, in the
     * given time unit.
     *
     * @param unit the time unit
     * @return the remaining delay; zero or negative values indicate
     * that the delay has already elapsed
     */
    long getDelay(TimeUnit unit);
}
{% endcodeblock %}

  DelayQueue中的元素类需要实现Delayed，实现getDelay()计算任务的剩余执行时间。
  PriorityQueue中的元素都需要继承Comparable，否则无法排序，这里是通过让Delayed继承来实现，然后在子类中重写compareTo()。

{% codeblock lang:java %}
// 任务生产逻辑
public static void producer(DelayQueue<DelayTask> queue) {
    new Thread(() -> {
        for (int i = 0; i < 10; i++) {
            String msg = "aaaaa" + i;
            // 为了便于测试，这里每隔一段时间往队列中加任务
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            DelayTask task = new DelayTask(100000L, msg);
            queue.offer(task);
            logger.info("生产任务：{}", task.toString());
        }
    }).start();
}

// 任务消费逻辑
public static void consumer(DelayQueue<DelayTask> queue) {
    new Thread(() -> {
        while (true) {
            DelayTask task = null;
            try {
                // 调用take()取出任务
                task = queue.take();
            } catch (InterruptedException e) {
                logger.info("消费任务异常：{}", e.getLocalizedMessage());
            }
            if (null != task) {
                logger.info("消费任务:{}", task.toString());
            } else {
                logger.info("没有待消费的任务");
            }
        }
    }).start();
}
{% endcodeblock %}

### 小结

      DelayQueue可应用于一些执行时间较为灵活的场景，比如开课前30分钟发送通知，但是课程的时间并不
    固定，就可以动态获取课程上课时间后定义一个延迟任务等待执行。
      示例中为了方便采用的单线程，但实际开发中，应该用多个线程作为消费者去处理队列中的任务。特别
    是当任务的逻辑较为复杂时，单线程处理会导致后续任务超时，至于线程数可以根据实际测试去设置。
      在写示例的过程中，有想到一个问题，就是如果有较多的任务需要在同一个时间节点执行，这时一个
    DelayQueue就无法处理。肯定会有大量的任务超时。我的想法是，如果不考虑其他方案，可能需要采用多
    个DelayQueue，同一个队列中避免执行时间相同的任务。在实际开发中，我们也要注意是否会有大量任务
    的执行时间点一样。
