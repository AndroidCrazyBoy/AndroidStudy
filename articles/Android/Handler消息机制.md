## Handler消息机制问答

**1. 内存泄漏引用链**

主线程 —> threadlocal —> Looper —> MessageQueue —> Message —> Handler —> Activity

**2. 一个线程有多少个Handler？ **

一个线程对应多个handler，因为可以无限制new。

**3. 子线程维护的Looper，消息队列无消息的时候的处理方案是什么？有什么用？**

Looper是一个死循环，主线程中有epoll阻塞机制，而子线程需要调用quit方法才能退出，要让loop退出，需要msg==null,这个时候就需要返回一个空的消息。主线程中不能调用quit方法，会抛出异常。

**4. 为什么主线程可以new Handler? 如果在子线程中new Handler需要怎么做？**

Handler的使用需要调用，Looper.preapre()和Looper.loop()，主线程中ActivityThread 中有主动调用Looper.prepare()和Looper.loop()。所以在主线程中可以直接问new Handler。而子线程中需要手动调用Looper.prepare()和Looper.loop()，才能够使用new Handler。需要注意的一点是，子线程中的Looper循环并不会随着线程的销毁而停止，从而会导致内存泄漏，所以在MessageQueue没有消息的时候需要手动调用Looper.quit()方法，使Looper循环退出。

**5. 多个Handler往MessageQueue中添加数据（发消息时各个Handler可能处于不同线程）那么它内部是如何保证线程安全的？取消息呢？**

enqueueMessage方法中使用 synchronized(this) 同步锁(内置锁),锁定了该对象 不同的线程发消息的时候，插入的数据进行排序操作 取消息的时候也需要一个加锁的操作,以防在取的时候有插入数据的操作导致数据错乱。

**6. Looper 死循环为什么不会导致应用卡死，会消耗大量资源吗？**

首先 Android 所有的 UI 刷新和生命周期的回调都是由 Handler消息机制完成的，就是说 UI 刷新和生命周期的回调都是依赖 Looper 里面的死循环完成的。

其次Looper 不是一直拼命干活的傻小子，而是一个有活就干没活睡觉的老司机，所以主线程的死循环并不是一直占据着 CPU 的资源不释放，不会造成过度消耗资源的问题。这里涉及到了Linux pipe/epoll机制，简单说就是在主线程的 MessageQueue 没有消息时，便在 loop 的 queue.next() 中的 nativePollOnce() 方法里让线程进入休眠状态，此时主线程会释放CPU资源，直到下个消息到达或者有事务发生才会再次被唤醒。所以 Looper 里的死循环，没有一直空轮询着瞎忙，也不是进入阻塞状态占据着 CPU 不释放，而是进入了会释放资源的等待状态，等待着被唤醒。

**7. 子线程中Toast，showDialog，的方法。（和子线程不能更新UI有关吗）**

子线程确实不能直接更新UI，但是Toast 和 Dialog在子线程中直接使用报错并不是线程问题，而是Handler报错，因为两者内部都是调用了Handler导致在子线程中直接使用报错。

**8. 如何处理Handler 使用不当导致的内存泄露？**

1.  将Handler实现为静态内部类，而不是匿名内部类或非静态内部类。在静态内部类中，通过弱引用（WeakReference）持有对Activity或其他可能引发内存泄露的对象的引用。

2.  在Activity的onDestroy()方法中，确保移除所有的回调和。

   ```java
      @Override
      protected void onDestroy() {
          super.onDestroy();
          handler.removeCallbacksAndMessages(null); //移除所有的回调和
      }
   ```

**9. MessageQueue的数据结构是什么样的？为什么要用这个数据结构？**

虽然MessageQueue叫消息队列，但是它的内部实现并不是用的队列。实际上它是通过一个单链表的数据结构来维护消息列表，单链表在插入和删除上比较有优势。

1. 动态大小：由于Android设备资源有限，MessageQueue采用单链表可以允许其动态调整大小，不需要在数量变化很大的情况下重新分配固定的内存空间。

2. 高效的插入和移除：在Handler机制中，经常被插入和移除。单链表结构能够在O(1)的时间复杂度内完成对头部的插入或删除（当然，这通常是在已经有指向要插入或删除节点前后的引用的情况下）。也可以在任意位置插入延迟，并可以维持其原本的排序。

3. 保持顺序：由于处理UI更新和后台任务常常需要特定的执行顺序，单链表确保可以按照投递的顺序依次处理。

**10. Handler是如何实现同步屏障的？**

可以调用MessageQueue的postSyncBarrier()方法。请注意，这个特性是内部的API并不对公共开发者API暴露，因此你不能在常规应用中直接使用它来同步队列。

屏障消息的特点是msg.target = null。

**11. Handler用了什么设计模式？用它的好处是什么？**

生产者-消费者模式 内存共享的设计。

 好处：可以保证数据生产消费的顺序(MessageQueue先进先出)不管是生产者（子线程）还是消费者（主线程）都只依赖缓冲区(Handler),不会相互持有，没有任何耦合。