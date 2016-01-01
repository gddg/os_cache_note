# chepter9_master_slave_kernel

## 主从内核笔记 ##

**摘要：**
- 主从内核 master slave kernel 
- 自旋锁 spin lock 

第八章讲的单处理上的操作系统互斥问题。（内核非抢占，测试设置）

**多处理在内核的竞争，如何解决？**
主从内核，在多处理器上仿照单处理器解决竞争问题，简单实现不用大改内核代码。
==*一个处理器负责内核活动，其他处理器负责用户代码。*==,从处理器需要内核处理时，切换到主处理器。


**什么会引发从处理器切换主处理器？**
- 缺页错误 Page fault
- 算术异常 
- 内核缺陷的互斥.kernel trap handle
- 设备驱动.device driver
- 设备中断处理器.device interrupt handle

.
> 页缺失
> （英语：Page fault，又名硬错误、分頁錯誤、尋頁缺失、缺页中断、页故障等）指的是当_软件试图访问已映射在虚拟地址空间中，但是目前并未被加载在物理内存中的一个分页时_，由中央处理器的内存管理单元所发出的中断。通常情况下，用于处理此中断的程序是操作系统的一部分。
> 如果操作系统判断此次访问是有效的，那么操作系统会尝试将相关的分页从硬盘上的虚拟内存文件中调入内存。而如果访问是不被允许的，那么操作系统通常会结束相关的进程。[1]

> 算术异常 arithmetics
> 电脑算术运算中产生的非法计算结果或非法数据输入而导致处理机作岀硬件级别的处理。一般为硬件自动中断。
> 产生算术异常的可能情况包括除零，数值越界等。

> Kernel panic
> A kernel panic is an action taken by an operating system upon detecting an internal fatal error from which it cannot safely recover. The term is largely specific to Unix and Unix-like systems; for Microsoft Windows operating systems the equivalent term is "stop error" (or, colloquially, "Blue Screen of Death").
>来自wiki 
>

内核态进程队列 kernel process queue,master queue 
用户态进程队列 slave run queue ,slave queue 

从处理器执行一次系统调用，进程被放入主处理器队列。
主处理器执行一次上下文切换（执行完了内核调用），正在执行的进程如果是用户态程序放入从队列，否则放入主队列，从主队列再选一个执行。

## 9.2 自旋锁
**用途**：短期互斥
**流程**：进入临界区获得自旋锁，出临界区释放锁。
**原理**：一个进程等待另一个进程使用的锁，它忙于等待状态（busy－wait 循环自旋 ♻️）。

**需要锁的进程所在CPU不停的循环判断这个锁是否可用，
CPU应该是占用住的，因为短期互斥可能几个毫秒就能拿到锁，所以这个代价不算什么。实现简单，高效。**

**负面：**
不应该用来长期互斥，自旋期间CPU浪费无用功，不停要求读一次内存。
多处理器都争用一个锁，浪费CPU，效率低。


初始化,拿到锁，释放锁。


```c
 typedef int lock_t;
 void 
 initlock( volatile lock * lock_status)
 {
 *lock_status = 0 ; //初始化锁
 }

//借助test_and_set 原子操作，独一无二的获得锁，
//如果没获得锁，那那么不停的尝试，直到拿到锁。
 void 
 lock （ volatile lock * lock_status )
 {
 while (test_and_set(lock_status ) == 1 )
 ; 
 ｝
 
 void 
 unlock ( volatile lock * lock_status )
 {
 *lock_status = 0 ; //释放锁
 ｝
 
// voliatile 告诉编译器，addr指向的地址，可能被别的CPU修改。
// 请不要addr放入寄存器和缓存和优化，避免被别人改了。无法试试拿到最新的值。
// 处理器优化，可能将多余的LOAD和store优化称一个操作。
// 多处理器编程声明变量 volatile 良好编程习惯
int 
test_and_set ( volatile int * addr )
{
	int old_value; 
	old_value = swap_atomic( addr,1 ); 
//原子操作
//addr值取出放入寄存器A，
//1放入addr中，
//寄存器A在把值给old_vlaue	
	if ( old_value == 0 )
		return 0 ;   //拿到锁返回0 
    return 1;	 //未拿到锁返回1 
} 

//应用层
{
lock( &spin_lock);
 do_critical_section(); //不超过几百行指令
unlock( &spin_lock);
}

```

##9.3 死锁
 发生：线程一次处理需要2把锁，2个线程个拿着对方需要的那把锁，等待彼此释放锁，永远等不大到。
 
----------
### 9.3.1 AB－BA死锁
**拿锁的顺序是相反的。**
资源a和b，我拿a－b，胖子要b－a，假设我们都完成了第一步，那么第二部就进行不下去了。

**解决**：以相同顺序拿到锁和释放锁， nested－lock 解决这类问题。
----------
### 9.3.2 拿自旋锁的人再也无法释放锁

拿锁L的进程被A－CPU切换出去还没释放锁，其他CPU都在自旋等拿这个锁，A－CPU执行的下一个进程也要求锁L而自旋，那么所有CPU都死锁了。

**解决**： 不允许占有自旋锁的进程被上下文切换出CPU。
----------
### 9.3.3 中断了自旋锁进程
在拿着自旋锁的CPU上发生中断，中断程序也需要这个自旋锁。

**解决**：必须屏蔽中断。
 ----------
### 9.3.4 请求同一把自旋锁2次
一个程序申请2次同一把锁，申请到第二次的时候自旋死锁了。
recurisive locking 递归上锁

**解决**：需要释放2次，否则其他操作可能出问题。


``` c
void AAA()
{ 
lock(a)  <--- 计数器 a.count++
 	BBB();
unlock(a) 释放 a.count--  
}

void BBB()
{
lock(a)  <---检测到已经获得了这把锁，只是计数器 a.count++ 
do ...
unlock(a)  <-----释放 a.count-- 
｝

```

---------- 
### 9.4 主从内核实现

队列实现：
无序链表 unsorted linked list  
优先级链表 

多核中使用了一种典型的编码技术：临界资源和锁，放在一个结构体中。
好处：我觉得管理上方便，不会找不到锁定变量。



 
 
----------

#### 示例 2

 

```seq
Alice->Bob: Hello Bob, how are you?
Note right of Bob: Bob thinks
Bob-->Alice: I am good thanks!
```





----------

Oops在Linux 2.6内核+PowerPC架构下的前世今生
oops 消息 Unable to handle kernel NULL pointer dereference at virtual address
内核 oop do_page_fault[转]


