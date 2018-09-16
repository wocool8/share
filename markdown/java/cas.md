# Compare And Swap
---
## 一 并发编程为什么要用CAS
 
### 1.1 锁的代价 
Java的并发编程在JDK1.5之前都是靠synchronized关键字来加锁。 锁是用来做并发最简单的方式，其代价也是最高的，加锁机制会有如下几个问题：
(1)锁、释放锁会需要操作系统进行上下文切换和调度延时，在上下文切换的时候，cpu之前缓存的指令和数据都将失效，这个过程将增加系统开销<br>
(2)多个线程同时竞争锁，锁竞争机制本身需要消耗系统资源。没有获取到锁的线程会被挂起直至获取锁，在线程被挂起和恢复执行的过程中也存在很大开销，也有可能产生死锁<br>
(3)等待锁的线程会阻塞，影响实际的使用体验。如果被阻塞的线程优先级高，而持有锁的线程优先级低，将会导致优先级反转(Priority Inversion)<br>
### 1.2 乐观锁与悲观锁
### 1.2.1 悲观锁
悲观锁：是认为别的线程会修改值。独占锁是一种悲观锁，synchronized就是一种独占锁。synchronized加锁后就能够确保程序执行时不会被其它线程干扰，得到正确的结果。
### 1.2.2 乐观锁
乐观锁：本质上是乐观的，认为别的线程不会去修改值。如果发现值被修改了，可以再次重试。
### 1.3 CAS是什么 
CAS的机制就相当于乐观锁（非阻塞算法），CAS是由CPU硬件实现，所以执行相当快.<br>
CAS有三个操作参数：内存地址，期望值，要修改的新值，<br>
当期望值和内存当中的值进行比较不相等的时候，表示内存中的值已经被别线程改动过，这时候失败返回，<br>
当相等的时候，将内存中的值改为新的值，并返回成功

## 二 JVM中的CAS
在java1.8之前 java.util.concurrent.atomic包中可以查看源代码的实现，这里参java.util.concurrent.atomic.AtomicLong源代码  
        
        //+1操作
        public final long getAndIncrement() {
            while (true) {
                long current = get();
                long next = current + 1;
                //当+1操作成功的时候直接返回，退出此循环
                if (compareAndSet(current, next))
                    return current;
            }
        }
    
        //调用JNI实现CAS
        public final boolean compareAndSet(long expect, long update) {
            return unsafe.compareAndSwapLong(this, valueOffset, expect, update);
        }
        
JNI:Java Native Interface为JAVA本地调用，允许java调用其他语言。在jdk1.8后getAndIncrement()方法已经看不到具体代码了，而是封装在unsafe类里面。

        // setup to use Unsafe.compareAndSwapLong for updates
        private static final Unsafe unsafe = Unsafe.getUnsafe();
        private static final long valueOffset;
        
        public final long getAndIncrement() {
            return unsafe.getAndAddLong(this, valueOffset, 1L);
        }
        // unsafe中被封装的实现
        public final long getAndAddLong(Object var1, long var2, long var4) {
            long var6;
            do {
                var6 = this.getLongVolatile(var1, var2);
            } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));
    
            return var6;
        }
        
## 三 CAS的缺点
CAS虽然很高效的解决原子操作，但是CAS仍然存在三大问题。ABA问题，循环时间长开销大和只能保证一个共享变量的原子操作。
### 3.1 ABA问题
因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。
ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。
从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。
    
    public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            ((newReference == current.reference &&
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }


compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

### 3.2 循环时间长开销大
自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。
### 3.3 只能保证一个共享变量的原子操作
当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行CAS操作。
