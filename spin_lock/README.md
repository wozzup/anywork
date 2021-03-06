# MCS多线程队列锁
一种线程公平的自旋锁实现

原子操作需要硬件支持，各体系实现不同

实现环境：gcc 4.7+  x86_64

## 1锁类型
#### 1.1自旋锁
自旋锁是一种非阻塞锁，也就是说，如果某线程需要获取自旋锁，但该锁已经被其他线程占用时，该线程不会被挂起，而是在不断的消耗CPU的时间，不停的试图获取自旋锁.

#### 1.2互斥锁
互斥锁是阻塞锁，当某线程无法获取互斥量时，该线程会被直接挂起，该线程不再消耗CPU时间，当其他线程释放互斥量后，操作系统会激活那个被挂起的线程，让其投入运行.

具体实现可分为:独占锁(同一时刻只有一个持有者)、共享锁(读写锁，读操作共享，写操作独占)

#### 1.3应用场景
* 如果是多核处理器，如果预计线程等待锁的时间很短，短到比线程两次上下文切换时间要少的情况下，使用自旋锁是划算的
* 如果是多核处理器，如果预计线程等待锁的时间较长，至少比两次线程上下文切换的时间要长，建议使用互斥锁
* 如果是单核处理器，一般建议不要使用自旋锁。因为，在同一时间只有一个线程是处在运行状态，那如果运行线程发现无法获取锁，只能等待解锁，但因为自身不挂起，所以那个获取到锁的线程没有办法进入运行状态，只能等到运行线程把操作系统分给它的时间片用完，才能有机会被调度。单核使用自旋锁可以借鉴`nginx spinlock`的实现
* 如果加锁的代码经常被调用，但竞争情况很少发生时，应该优先考虑使用自旋锁，自旋锁的开销比较小，互斥量的开销较大

## 2 MCS队列锁
要理解MCS Locks的本质，必须先知道其产生背景(Why)，然后才是其运作原理。就像原论文提到的，我们先从spin-lock说起。spin-lock 是一种基于test-and-set操作的锁机制。
```c
function Lock(lock){
    while(test_and_set(lock)==1);
}

function Unlock(lock){
    lock = 0;
}
```
test_and_set是一个原子操作，读取lock，查看lock值，如果是0，设置其为1，返回0。如果是lock值为1， 直接返回1。这里lock的值0和1分别表示无锁和有锁。由于test_and_set的原子性，不会同时有两个进程/线程同时进入该方法， 整个方法无须担心并发操作导致的数据不一致。

一切看来都很完美，但是，有两个问题：
* (1) test_and_set操作必须有硬件配合完成，必须在各个硬件(内存，二级缓存，寄存器)等等之间保持 数据一致性，通讯开销很大。
* (2) 他不保证公平性，也就是不会保证等待进程/线程按照FIFO的顺序获得锁,可能有比较倒霉的进程/线程等待很长时间 才能获得锁。

使用mcs锁的目的是，让获得锁的时间从O(n)变为O(1)。每个处理器都会有一个本地变量不用与其他处理器同步。

当我们试图获得锁时，先将自己加入队列，然后看看有没有其他 人(predecessor)在等待，如果有则等待自己的is_lock节点被修改成false。当我们解锁时，如果有一个后继的处理器在等待，则设置其is_locked=false，唤醒他。

在Lock方法里，用fetch_and_store来将本地node加入队列，该操作是个原子操作。如果发现前面有人在等待,则将本节点加入等待节点的next域中,等待前面的处理器唤醒本节点。 如果前面没有人，那么直接获得该锁。

在Unlock方法中，首先查看是否有人排在自己后面。这里要注意，即使暂时发现后面没有人,也必须用原子操作compare_and_swap确认自己是最后的一个节点。如果不能确认 必须等待之后节点排(my_node.next == NULL)上来。最后设置my_node.next.is_locked = false唤醒等待者。

![mcs_lock](https://raw.githubusercontent.com/pangudashu/anywork/master/_img/mcs_lock.jpg)

## 3使用
```c
//初始化锁
int mcs_init(mcs_locker_t *locker) 

//每个线程添加node
void mcs_thread_add_node(mcs_locker_t *locker, mcs_lock_node_t *thread_lock_node)

//获取每个线程的node
mcs_lock_node_t * mcs_thread_get_node(mcs_locker_t *locker)

//lock
void mcs_lock(mcs_locker_t *locker)

//unlock
void mcs_unlock(mcs_locker_t *locker)
```
锁在使用前必须调用mcs_init()初始化，各线程使用前必须调用mcs_thread_add_node()在locker中加入node，否则`锁无效`
