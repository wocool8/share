# synchronized
---
## 一 synchronized使用
### 1.1 修饰普通方法  
    public synchronized void method() {
        // do something
    }
    
### 1.2 synchronized 代码块
synchronized 代码块是对实例对象加锁

    synchronized (this) {
        // do something
    }      
### 1.3 修饰静态方法(类)  
对静态方法的同步本质上是对类的同步（静态方法本质上是属于类的方法，而不是对象上的方法）,对类对象加锁

    public static synchronized void method() {
        // do something
    }  
    
    synchronized (SyncTest.class) {
        // do something
    }      

## 二 synchronized实现原理
### 2.1 Synchronized的语义底层是通过一个monitor的对象来完成

    public static synchronized void method() {
        synchronized (SyncTest.class) {
            System.out.println("hello world");
        }
    }
使用 javap -v class SyncTest.class 反编译结果如下图
![规则匹配](../../picture/thread/syncronized.PNG)
执行同步代码首先要先执行monitorenter指令，退出时执行monitorexit指令，使用Synchronized进行同步，其关键就是对对象监视器monitor进行获取

#### 2.1.1 monitorenter
JVM规范中对monitorenter描述如下：

    monitorenter ：
    Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes 
    monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:
    • If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. 
      The thread is then the owner of the monitor.
    • If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.
    • If another thread already owns the monitor associated with objectref, the thread blocks until the monitor’s entry count is zero, 
       then tries again to gain ownership.
 
 #### 2.1.2 monitorexit
 JVM规范中对monitorexit描述如下：      

    monitorexit：　
    The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.
    The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, 
    the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.       
 
### 2.3 方法级synchronized
方法级的同步是隐式，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 [ACC_SYNCHRONIZED](/markdown/jvm/class.md) 访问标志区分一个方法是否同步方法
  
### 2.4 对象头
使用new创建一个对象的时候，JVM会创建一个instanceOopDesc对象，这个对象中包含了对象头以及实例数据。
synchronized在JVM层面实现了对临界资源的同步互斥访问，通过对对象的头文件来操作，从而达到加锁和释放锁的目的。

### 2.5 notify/notifyAll/wait
JDK文档对于notify/notifyAll/wait描述如下

    void notify() 
    Wakes up a single thread that is waiting on this object’s monitor. 
    
    void notifyAll() 
    Wakes up all threads that are waiting on this object’s monitor. 
    
    void wait( ) 
    Causes the current thread to wait until another thread invokes the notify() method or the notifyAll( ) method for this object. 

由于 需要获取对象的monitor所以 必须在同步方法中使用 否则会抛出java.lang.IllegalMonitorStateException



