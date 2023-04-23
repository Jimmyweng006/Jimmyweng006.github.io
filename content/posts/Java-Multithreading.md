---
title: "Java Multithreading"
date: 2023-04-23T09:55:26+08:00
draft: false
tags: [JAVA]
categories: [Multithreading]
author: "Jimmy_kiet"
isCJKLanguage: true
---

## 前言

好久沒發文了，這次來分享一下最近上完的[Udemdy課程](https://www.udemy.com/course/java-multithreading-concurrency-performance-optimization/)的心得筆記，然後也附上在公司的[code sharing](https://docs.google.com/presentation/d/1SIHBmuGt-V2TRlFyaXQmNSriK-MDHXG-KSi7eePu_7o/edit#slide=id.p)用的講稿XD。

## Section 1

### Why

1. Responsiveness on UI(can achieve by only one core -> Concurrency)
2. Performance(can achieve by multiple core -> parallel)

### Concept

1. Program from disk -> (load into memory) -> an instance of application(process)
2. process
    1. shard data(File, Heap, Code)
    2. threads(at least one main thread)
        1. stack
        2. instruction pointer(address of next excuted instruction)

## Section 2

### Context switch

1. define
    1. store current thread data into memory
    2. restore next executed thread data
2. Thrashing: spend too much time on doing context switching when too many threads situation
3. Thread context switch is cheaper than Process context switch

### Thread Scheduling

1. FCFC or shortest first might have some problems
2. modern Thread Scheduling algorithm: each thread get executed portionly in fiexd time slice(epochs)
3. Dynamic Priority = Static Priority(set in program) + Bonus, to prevent some threads not get change to be executed(starvation)

### Thread Creation

1. pass an runnable object to Thread constructor
2. implement a class that extends Thread class

## Section 3

### Thread Termination

* application will not stop as long as at least one thread is running, even if main thread is stopping
* thread.interrupt() need to be used with Thread.currentThread().isInterrupted() or catch (Exception e)
* Daemon Thread: prevent application running even when main thread stop

### Thread Coordination(introduce)

* context: Thread B computation depends on Thread A result, then how Thread B need to start working after Thread A finish?
    1. Thread B looping check Thread A finish? -> waste CPU resource
* Thread Join: use thread.join(timeout) to wait this thread at most timeout milisecond -> handle timeout threads

```
need to use join avoid race condition -> one thread is not finished when checking, 
but before application finish, it becomes finished but have no chance to change finished state...
```

## Section 4

### performance metric

1. latency
    * achieve optimal performance -> N(number of thread) should close to number of cores
3. throughput(introduce)

### Use case

1. Latency: image processing
2. Throughput: improve throughput by # threads
    * Thread Pool: eliminate the cost of recreating threads for each task

## Section 5

* Memory Region
    1. Stack: each thread have its own stack
    2. Heap: shared by each threads
* shared resources between threads
    * Race Condition: multiple threads use non-atomic operation on shared resource

## Section 6

> synchronization happened on object level

### Two ways for handling critical section problem, both use synchronized keyword
    
1. Monitors on method
2. synchronized section

### Atomic Operation

* volatile keyword: use for assign long/double type variable
* operations:
    1. assignment to primitive type variable
    2. assignment to reference
    3. assignment to long/double using volatile

### Data Race

* reorder instructions order(for high performance, and no dependency between lines of code)may cause this problem
    * sol: use volatile to guarantee order

### Deadlock

* number of lock for shared resources
    1. single
        * easy but lost benefit of parrallel execution
    3. multiple
        * have benefit of parrallel execution but may cause deadlock
* 4 condtions to cause Deadlock
    * easiest solution: avoid circular wait(acquire lock in same order)

## Section 7

### ReentrantLock

* avoid lock not be unlocked -> use finaly keyword to unlock
* use tryLock() to check current lock is available, and to avoid thread suspend.
    * if not check, execution will proceed therefore critical section is not being protected

### ReadWriteLock

* motivation: prevent read threads block each others
* properties:
    1. read threads will not block other read thread
    2. read/write threads will block opposite thread

## Section 8

### Semaphore

* a lock restricts number of threads to access resource
    * disadvantage: other thread may ralease Semaphore that acquired by other thread, which results in bug that multiple threads go into critical section in same time.
    * release() -> -= 1
    * acquire() -> += 1
    * if cnt != 0 -> mean blocking
    * Quiz 12: Semaphores - Barrier(introduce)

### Inter-Thread Communication

* cases
    1. thread.sleep()
    2. thread.join()
    3. acquire/release Semaphore
* Condition Variable
    1. await()
    2. notify()
* Quiz 13: Condition Variables -> difference between codition variable usage

## Section 9

* Lock-Free technique
    * lock might have some problems:
        1. thread A get lock, and before unlock thread A be context swithed to thread B, and thread B require same lock with thread A, then blocking happened...
    * introduce: atomic-integer-example
        * 4:08. 31. Atomic Integers & Lock Free E-Commerce, example showcase why we need to use join instead of thread.sleep()
        * even thought each method is atomic, that wrap method is not atomic!!!
* AtomicReference: do atomic operations on reference of object
* CAS(CompareAndSet): prevenet other thread modify value unexpectedly

## Section 10

Congradulation!!!

