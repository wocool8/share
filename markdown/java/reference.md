# 引用
---
## 一 强引用(Strong Reference) 
强引用指的时程序代码中普遍存在的，只要强引用还存在，垃圾回收机制永远不会回收掉被引用的对象（）

    Object obj = new Object()
java虚拟机会使用可达性分析算法对强应用进行回收
## 二 软引用(Soft Reference)
软引用就是用来描述一些还有但是并非必须的的对象，对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象进列进回收范围之中进行二次回收。如果这次回收还没有足够的内存才会抛出内存溢出异常
    
    String value = new String(“mumu”);
    SoftReference softReference = new SoftReference(value);
    //获得引用对象值
    sfRefer.get();
    
我们将使用一个Java语言实现的雇员信息查询系统查询存储在磁盘文件或者数据库中的雇员人事档案信息，我们在浏览WEB页面的时候也常常会使用“后退”button。这时我们一般会有两种程序实现方式:一种是把过去查看过的雇员信息保存在内存中，每个存储了雇员档案信息的Java对象的生命周期贯穿整个应用程序始终;还有一种是当用户開始查看其它雇员的档案信息的时候，把存储了当前所查看的雇员档案信息的Java对象结束引用，使得垃圾收集线程能够回收其所占用的内存空间，当用户再次须要浏览该雇员的档案信息的时候，又一次构建该雇员的信息。非常显然，第一种实现方法将造成大量的内存浪费，而另外一种实现的缺陷在于即使垃圾收集线程还没有进行垃圾收集，包括雇员档案信息的对象仍然完善地保存在内存中，应用程序也要又一次构建一个对象。我们知道，訪问磁盘文件、訪问网络资源、查询数据库等操作都是影响应用程序执行性能的重要因素，假设能又一次获取那些尚未被回收的Java对象的引用，必将降低不必要的訪问，大大提高程序的执行速度
## 三 弱引用(Weak Reference)
弱引用也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用的对象只能生存到下一次垃圾回收之前。当垃圾回收的时候无论内存是否足够，都会回收掉只被弱引用的关联的对象
    
    static class ThreadLocalMap {
    
         static class Entry extends WeakReference<ThreadLocal<?>> {
         /** The value associated with this ThreadLocal. */
         Object value;
    
         Entry(ThreadLocal<?> k, Object v) {
              uper(k);
              value = v;
         }
    }
    
ThreadLocal的内部类ThreadLocalMap使用软引用，来避免被ThreadLocal应用而不能被回收造成的内存泄露
## 四 虚引用(Phantom Reference)
许引用也被称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存周期构成影响，也无法通过虚引用来获得一个对象的实例。为一个对象设置虚引用的唯一目的就是能在这个对象被收集器回收时收到一个系统通知
   
    String value = new String(“mumu”);
    ReferenceQueue<Object> refQueue = new ReferenceQueue<Object>();
    PhantomReference<Object> phantom = new PhantomReference<Object>(obj, refQueue);
         
调用Object的finalize() 并不一定会立刻被回收，有可能还被唤醒，但是使用PhantomReference 会被回收，所以在显示调用Object的finalize()方法时可以考虑使用PhantomReference