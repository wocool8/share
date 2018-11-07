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
        abstract static class Sync extends AbstractQueuedSynchronizer {
            abstract void lock();
            // 由实现类实现获取锁方法
            protected boolean tryAcquire(int arg) {
                throw new UnsupportedOperationException();
            }    
            ...   
        }       
    }
### 1.1  NonfairSync

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
### 1.2  NonfairSync   
 根据线程
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

### 1.3 基于CAS实现锁
无论公平锁与非公平锁都是基于[CAS](/markdown/java/cas.md)实现 调用公共方法compareAndSetState

    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }