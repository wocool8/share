# ReentrantLock
---
## 一 公平锁与非公平锁 
ReentrantLock的构造方法默认使用非公平模式，也可以ReentrantLock(true)创建公平模式

    public class ReentrantLock implements Lock, java.io.Serializable {
        private final Sync sync;
        public ReentrantLock() {
            sync = new NonfairSync();
        }
        public ReentrantLock(boolean fair) {
            sync = fair ? new FairSync() : new NonfairSync();
        }     
    }
    
### 1.1  AbstractQueuedSynchronizer
   
AbstractQueuedSynchronizer是一个链表结构用于实现公平锁的线程访问先进先出，ReentrantLock使用state作为锁的状态字段，state == 0表示锁没有被持有，state ！= 0 表示锁被持有。
  
    abstract static class Sync extends AbstractQueuedSynchronizer {
        
        /**
         * The synchronization state.
         */
        private volatile int state;
            
        abstract void lock();
        // 由实现类实现获取锁方法
        protected boolean tryAcquire(int arg) {
            throw new UnsupportedOperationException();
        }    
        ...   
    }      
### 1.2  FairSync   
就是线程按照执行顺序排成一排，依次获取锁，但是这种方式在高并发的场景下极其损耗性能
 
    static final class FairSync extends Sync {
            private static final long serialVersionUID = -3000897897090466540L;
    
            final void lock() {
                acquire(1);
            }
    
            protected final boolean tryAcquire(int acquires) {
                final Thread current = Thread.currentThread();
                int c = getState();
                if (c == 0) {
                    if (!hasQueuedPredecessors() &&
                        compareAndSetState(0, acquires)) {
                        setExclusiveOwnerThread(current);
                        return true;
                    }
                }
                else if (current == getExclusiveOwnerThread()) {
                    int nextc = c + acquires;
                    if (nextc < 0)
                        throw new Error("Maximum lock count exceeded");
                    setState(nextc);
                    return true;
                }
                return false;
            }
        }
 公平锁 基于hasQueuedPredecessors实现有序
 
     public final boolean hasQueuedPredecessors() {
         // The correctness of this depends on head being initialized
         // before tail and on head.next being accurate if the current
         // thread is first in queue.
         Node t = tail; // Read fields in reverse initialization order
         Node h = head;
         Node s;
         return h != t &&
             ((s = h.next) == null || s.thread != Thread.currentThread());
     }    
### 1.3  NonfairSync

     static final class NonfairSync extends Sync {
            final void lock() {
                if (compareAndSetState(0, 1))
                    setExclusiveOwnerThread(Thread.currentThread());
                else
                    acquire(1);
            }
            
            protected final boolean tryAcquire(int acquires) {
                return nonfairTryAcquire(acquires);
            }
            
            final boolean nonfairTryAcquire(int acquires) {
                final Thread current = Thread.currentThread();
                int c = getState();
                if (c == 0) {
                    if (compareAndSetState(0, acquires)) {
                        setExclusiveOwnerThread(current);
                        return true;
                    }
                }
                else if (current == getExclusiveOwnerThread()) {
                    int nextc = c + acquires;
                    if (nextc < 0) // overflow
                        throw new Error("Maximum lock count exceeded");
                    setState(nextc);
                    return true;
                }
                return false;
            }            
        }

### 1.4 基于CAS实现状态变更
无论公平锁与非公平锁都是基于[CAS](/markdown/java/cas.md)实现AbstractQueuedSynchronizer的state状态变更，实现方法为compareAndSetState

    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }