# ReentrantLock
---
## 公平模式和非公平模式
ReentrantLock的构造方法默认使用非公平模式，也可以ReentrantLock(true)创建公平模式

    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
    
代码

    public class TestReentrantLock {
        
        public static void main(String[] args) {
            final ReentrantLock lock = new ReentrantLock(true);
            ExecutorService pool = Executors.newSingleThreadExecutor();
            pool.execute(new Runnable() {
                public void run() {
                    lock.lock();
                    try {
                        Thread.sleep(100000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        lock.unlock();
                    }
                }
            });
        }
    }
