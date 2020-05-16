# Introduction

StampedLock was introduced in Java 8 and has been one of the most important features of the concurrent family.

# Why Lock is needed?

## Problems without Lock

*   Inconsistency

Demonstrate by Example

**ExampleWithoutLock.java**
``` java
public class ExampleWithoutLock implements Runnable {
  @Getter
  private int count = 0;
 
  @Override
  public void run() {
    count = count + 1;
  }
}
```

**MainTask.java**
``` java
ExecutorService executor = Executors.newFixedThreadPool(3);
 
ExampleWithoutLock exampleWithoutLock = new ExampleWithoutLock();
 
IntStream.range(0, 10000)
    .forEach(i -> executor.submit(exampleWithoutLock::run));
 
// Wait for the job to finish
try {
    TimeUnit.SECONDS.sleep(10);
} catch (InterruptedException e) {
    e.printStackTrace();
}
 
System.out.println(exampleWithoutLock.getCount());  // 9900
```
## Problems with Lock

*   Deadlock
*   Starvation

# Comparing other mechanisms

## synchronized

**ExampleWithSynchronized.java**
``` java
@Override
public synchronized void run() {
  count = count + 1;
}
```
## ReentrantLock

**ExampleWithReentrantLock.java**
``` java
@Override
public void run() {
  reentrantLock.lock();
  try {
    count = count + 1;
  } finally{
    reentrantLock.unlock();
  }
}
```

The class `ReentrantLock` is a mutual exclusion lock with the same basic behavior as the implicit monitors accessed via the `synchronized` keyword but with extended capabilities.
This method is thread-safe just like the synchronized counterpart. If another thread has already acquired the lock subsequent calls to `lock()` pause the current thread until the lock has been unlocked. Only one thread can hold the lock at any given time.
The constructor for this class accepts an optional _fairness_ parameter. When set `true`, under contention, locks favor granting access to the longest-waiting thread. Otherwise this lock does not guarantee any particular access order. 
It supports various methods for fine grained control:

*   tryLock
*   isHeldByCurrentThread
*   isLocked 
*   newCondition 
*   getQueuedThreads 
*   getWaitingThreads(Condition condition)  

## ReentrantReadWriteLock

The idea behind read-write locks is that it's usually safe to read mutable variables concurrently as long as nobody is writing to this variable. So the read-lock can be held simultaneously by multiple threads as long as no threads hold the write-lock. This can improve performance and throughput in case that reads are more frequent than writes.

**ExampleWithReentrantReadWriteLock.java**
``` java
@Override
public void run() {
  reentrantReadWriteLock.writeLock().lock();
  try {
    count = count + 1;
  } finally{
    reentrantReadWriteLock.writeLock().unlock();
  }
}
```

Example: 1 writer and 2 readers

**MultipleReaders.java**
``` java 
Map<String, String> map = new HashMap<>();
ReadWriteLock lock = new ReentrantReadWriteLock();
 
executor.submit(() -> {
  lock.writeLock().lock();
  try {
    System.out.println(currentThread().getName() + ": start");
    sleep(100);
    map.put("foo", "bar");
    System.out.println(currentThread().getName() + ": end");
  } catch (InterruptedException e) {
    e.printStackTrace();
  } finally {
    lock.writeLock().unlock();
  }
});
 
Runnable readTask = () -> {
  lock.readLock().lock();
  try {
    System.out.println(currentThread().getName() + ": " + map.get("foo"));
    sleep(100);
    System.out.println(currentThread().getName()+ ": end");
  } catch (InterruptedException e) {
    e.printStackTrace();
  } finally {
    lock.readLock().unlock();
  }
};
 
executor.submit(readTask);
executor.submit(readTask);
```
Output

**Output.txt**
``` java
pool-1-thread-2: start
pool-1-thread-2: end
pool-1-thread-3: bar
pool-1-thread-1: bar
pool-1-thread-3: end
pool-1-thread-1: end
```
## StampedLock

**ExampleWithStampedLock.java**
``` java
@Override
public void run() {
  long stamp = stampedLock.writeLock();
  try {
    count = count + 1;
  } finally{
    stampedLock.unlockWrite(stamp);
  }
}
```

## Summary

||Reentrant?|Support Condition?|Support R/W Lock?|Who can release?|
| --- | --- | --- | --- | --- |
|synchronized| Yes | No | No | NA |
|ReentrantLock| Yes | Yes | No | Only owner thread |
|ReentrantReadWriteLock| Yes | Yes | Yes | Only owner thread |
|StampedLock| No | No | Yes | Any thread (But not suggested to do) |

# Deep insight into StampedLock

## tryOptimisticRead

The optimistic lock is valid right after acquiring the lock. In contrast to normal read locks an optimistic lock doesn't prevent other threads to obtain a write lock instantaneously. After sending the first thread to sleep for one second the second thread obtains a write lock without waiting for the optimistic read lock to be released. From this point the optimistic read lock is no longer valid. Even when the write lock is released the optimistic read locks stays invalid.
So when working with optimistic locks you have to validate the lock every time _before_ accessing any shared mutable variable to make sure the read was still valid.

**OptimisticRead.java**
``` java 
// Optimistic read
StampedLock stampedLock = new StampedLock();
 
executor.submit(() -> {
  long stamp = stampedLock.tryOptimisticRead();
  try {
    System.out.println("Optimistic Lock Valid: " + stampedLock.validate(stamp));
    sleep(100);
    System.out.println("Optimistic Lock Valid: " + stampedLock.validate(stamp));
    sleep(200);
    System.out.println("Optimistic Lock Valid: " + stampedLock.validate(stamp));
  } catch (InterruptedException e) {
    e.printStackTrace();
  } finally {
    stampedLock.unlock(stamp);
  }
});
 
executor.submit(() -> {
  long stamp = stampedLock.writeLock();
  try {
    System.out.println("Write Lock acquired");
    sleep(200);
  } catch (InterruptedException e) {
    e.printStackTrace();
  } finally {
    stampedLock.unlock(stamp);
    System.out.println("Write done");
  }
});
```
Output

**Output.txt**
```
Optimistic Lock Valid: true
Write Lock acquired
Optimistic Lock Valid: false
Write done
Optimistic Lock Valid: false
```
## tryConvertToWriteLock

Sometimes it's useful to convert a read lock into a write lock without unlocking and locking again. `StampedLock` provides the method `tryConvertToWriteLock()` for that purpose.

**ConvertToWriteLock.java**
``` java
// Convert to write lock
executor.submit(() -> {
  long stamp = stampedLock.readLock();
  try {
    if (testValue == 0) {
      stamp = stampedLock.tryConvertToWriteLock(stamp);
      if (stampedLock.validate(stamp)) {
        testValue = 23;
      } else {
        System.out.println("Could not convert to write lock");
        stamp = stampedLock.writeLock();
      }
    }
    System.out.println(testValue);
  } finally {
    stampedLock.unlock(stamp);
  }
});
```
Can you find bug?
There is a bug in above code, are you able to recognize?

# Best Practice

*   If readers are much more than writers, StampedLock is preferred.
*   StampedLock is not reentrant, so each call to acquire the lock always returns a new stamp and blocks if there's no lock available, even if the same thread already holds a lock, which may lead to deadlock.
*   Never use the same variable to save the stamp value returned by StampedLock API.
*   There could be a situation when you acquired the write lock and written something and you wanted to read in the same critical section. So, as to not break the potential concurrent access, we can use the _tryConvertToReadLock(long stamp)_ method to acquire read access. Now suppose you acquired the read lock, and after a successful read, you wanted to change the value. To do so, you need a write lock, which you can acquire using the _tryConvertToWriteLock(long stamp)_ method.
*   One thing to note is that the _tryConvertToReadLock(long stamp)_ and _tryConvertToWriteLock(long stamp)_ methods will not block and may return the stamp as zero, which means these calls were not successful.
*   The stamp returned by _tryOptimisticRead()_ **should not** apply to _unlock(long stamp)_.
*   If there is read lock held by others, one thread have valid optimistic read, this thread can obtain read lock by calling _readLock()_, while cannot obtain read lock by calling _tryConvertToReadLock(long stamp)_.

# Reference
Try Block with Lock: https://stackoverflow.com/questions/10868423/lock-lock-before-try 
