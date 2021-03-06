# 8 章 多处理器系统笔记

多CPU共同访问内存，硬件访问细节探讨。


##关键字##


key  |名词 | 解释 
----------|-------------|----
 SMP  |  symmetric multiprocessor  | 对称多处理器，紧密耦合（与内存），共享内存访问。  
UP  |  uniprocessor  | 单处理器    
MP | multiprocessor | 多处理器  |  
*|memory model | 存储模型。  |  
*|mutual exclusion | 互斥   | 

----------


## 8.1 多核操作系统需求
满足几点：
1. 系统完整性。system integrity 
2. 性能。performance
3. 外部编程模型。external programming model

**1>2>3 优先级**

> 不能因为多核操作同一个数据结构，造成崩溃。
> 不能CPU的添加，无法获得对应性能增长。
> 对有的接口进行了改变，造成代码需要修改。开发者需要知道，这个程序是需要适应多核芯来进行修改的。代码的可移植性。


## 8.2 SMP 

**紧密耦合**     － CPU/内存／IO设备都连接在一个总线上。
**共享存储器**    － 共享内存，所有CPU都可以访问到。
**对称多处理器**  － 对称访问，大家都有平等访问权利不干扰。


CPU使用总线和内存，需要仲裁，当前总线给谁使用。
多个CPU访问同一个内存地址，需要仲裁，避免彼此干扰。

物理限制－－CPU的数量受到共享总线核存储器带宽的限制。
任何的延迟都会让CPU执行指令的速度减慢，等待内存提供下一个指令是CPU空闲。

CPU带宽  ＊ 个数  ＋ DMA 带宽 ＝ 总线或主存带宽  


## 8.3 多核存储器模型

存储模型

多核如何访问主存储器？ 
多核同时访问主存储器有什么影响？ 
操作系统开发人员： 不同MP硬件系统可能会实现不同的存储器模型。
存储器模型之间的差别，主要集中load加载和store保存指令的执行顺序。
AIX和X86，大小端序列存放值。


## 8.3.1 顺序存储模型
**按照次序，前面的指令比后面的指令先执行。**
squential memory model，也叫strong orderfing 强定序.
按照程序次序来load&store指令,**指令流中出现的次序来执行**.


## 8.3.2 原子读原子写
强定序模型下，CPU，I/O DMA，主存 单次读或写都是原子操作。

> 一旦开始读或写,不会被中断。（CPU或IO设备）
> 但是不会读太多，限制只能读一行。
> 必须释放资源，再次排队等待下次分配到使用权

共享总线实现原子访问

**原子访问原理**
> 多个CPU同时访问主存储器，
> 总线硬件在多个requestor仲裁,确定一个CPU，
> CPU允许一次读或写,连续的多个字，
> 用完后释放权限，循环仲裁,找下一个CPU或硬件。

对存储器的访问，无论位置是否相同，都会被总线仲裁器给顺序化。

谁是下一个使用内存者？调度算法。
> * FIFO －－first in first out－－先入先出。
> * round-robin 循环 RR。

**设计原则，保证每个CPU都能轮到访问内存，平等非抢占。**


竞争条件（Race Condition）
> 对一个内存块，
> 总线分配CPU的先后访问内存顺序，
> 因为调度算法不保证谁先请求就先给谁用，
> 一个写一个读，到底读到什么不确定。不确定读到新的数据，还是旧数据。
> 一个写A一个写B，到底最后是哪个结果不确定。不确定，最后结果是A还是B。

**原子读写和竞争区别**

> 原子操作内存一次读写只物理访问上单次操作不被中断，无法保证硬件层面提供操明确指令顺序保证（2个CPU同时下达指令，但是谁抢到BUS仲裁权，或者硬件调度算法，后来者先执行了），所以 **原子读写（并发写硬件仲裁，无顺序概念）**
> 
.
.
> 
> 原子一次读或一次写的先后，对确定最后数据的结果无帮助，因为单次简单操作无法确定整体的执行先后。
> 原子读或者写，只是访问硬件层面的控制一次操作不中断。
> 而竞争是多CPU的多组操作，一组操作以一个整体执行，组和组之间需要保证顺序，才能有确定结果。
>

**竞争（race condidtion，并发写确定顺序执行）** 确定多个CPU上执行指令的相对顺序或时序。


**race** 其实反映了，大家都在努力的奔跑，无法确定谁先谁后，顺序无法确定，可能造成结果也无法确定。

==多个处理器SMP，访问同一个内存地址，进行同步。说一下引出下文，原子读改写。==
## 8.3.3 原子 读－改－写

read－modify－write 
过程： 读取，修改值，写回原位置。一个原子总线操作。
特殊CPU指令实现，CPU实现复杂。同步时用。银行存款绝对可靠。


### 8.3.3.2 test_and_set 测试设置指令
原理：读一个字，字和0比较，处理器设置**条件码**，1保存字上。
> 测试变量，是不是0，结果放在条件码寄存器里面，再把这个变量设置为1 。 

这种常见原子操作，硬件实现简单。
条件码：进位标志，零标志，符号标志，溢出标志。

测试和设置，需要原子交换知道那个变量是0可用，还是1上锁了。

compare-and-swap (CAS)
<https://en.wikipedia.org/wiki/Compare-and-swap>
 

swap－atomic 原子交换

```
int 
test_and_set ( volatile int * addr )
// voliatile 告诉编译器，addr指向的地址，可能被别的CPU修改。
// 请不要addr放入寄存器和缓存和优化，避免被别人改了。无法试试拿到最新的值。
// 处理器优化，可能将多余的LOAD和store优化称一个操作。
// 多处理器编程声明变量 volatile 良好编程习惯
{
	int old_value; 
	old_value = swap_atomic( addr,1 ); 
//原子操作，addr值取出放入寄存器A，
//寄存器B中1放入addr中，
//寄存器A在把值给old_vlaue	
	if ( old_value == 0 )
		return 0 ;
    return 1;	
}
```

**用法**
如果原来addr是0，那么我取到0，现在addr是1. return 0，我知道我拿到了锁，那么我独占使用这个资源。用完后再 test_and_set() 放入0，释放锁。
另外一个人去取addr，return 1 ， 设置1（原来是1）。那么我没拿到锁，然后继续重试直到 return 0，为止。
**通过这个功能来构建一个应用层的锁。**
.
.

**如果原子操作 swap_atomic() 不是原子的，会怎么样？**
这个功能实现的是寄存器和内存中值交换。取出原值，保存寄存器的值到内存中，返回原值。
2个人同时做，非原子，bus调度可能a做到一半，取出0，b开始做，b也取出0，b继续关闭锁赋值1，然后又切换回a，a也认为自己拿到了锁，然后并发数据结构被破坏，无法得到准确结果。

volatile *禁止代码优化，对多处理器访问一个变量是好习惯。*
测试和设置，需要原子交换知道锁变量是0可用，还是1上锁了。


### 8.3.3.4 Dekker算法
未写。



voliate 的问题：

```
class thread1 ｛
while(1){
  if(isbegin == true )break; //无论如何都无法执行到这句话
｝
｝
class thread2 ｛
isbegin=true; 
｝

boolean isbegin;
//release版本，被优化到寄存器或cache，无法每次都取内存，所以始终无法判断正确
voliate boolean isbegin; 

```



## 并发分析
**种类1**：
写入a，无论原来是值是什么，直接写a。 不停的覆盖一个值。
读取，不论原来是什么，给我读一下内存。 不停读取。
这种操作，本质上是指令循序的先后，来确定之行的结果。（单线程）
如果多线程，大量这种简单指令（针对一个内存地址），根本无法推测结果，因为他们的先后执行顺序，
根据CPU访问BUS的硬件调度来控制，谁先操作谁后操作无法得知，硬件层次根本不会告诉你，
CPU会空闲，你只是拿到结果中间是否发生等待不知道。

－－－－－－－－－－－－－－－－－－－－－
**种类2**
单纯a线程拿值a1覆盖变量C，单纯b线程拿值b1覆盖变量C，
a1和b1都不是基于原来变量C，都是基于自己线程的a1-old，b1-old来的。
a1和b1，都修改C，最多我们看出C最后被谁修改的。b1（1000～5k）和a1（0～999）有明确的取值范围。
这种场景几乎没意义，知道当前开灯关灯，本质上好无意义。
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
**计数器应用**
根据上次的结果本次加1，OS提供了原子加一操作。
如果硬件底层实现了，存储＋1操作原子，那么os简单包装。
如果硬件没实现,OS必须


－－这种场景，除非在app层次有一个地方排队请求，才能确定结果（人工可以推演这个结果）。
－－因为我在app层排队了，排队中的顺序如果我知道，我就能预测结果，也就消除了锁的问题把并发变成了排队。

**抢购**
排队




**单次操作的顺序问题？**
原则应该是谁先执行，多个Cpu执行指令顺序（主机时钟），可以决定执行结果，如果因为访问内存调度的问题，
造成了（BUS在一个时刻只能一个CPU用），先执行的顺序被重新排序了，而这种排序是硬件上的实现的，
人无法预测结果是否为我们预期的。
人可以预期的方式，先发先出结果！


**数据库，补充案例：**
* 不可重复读
* 不可幻读
* 不可


读改写，原子操作。
明确在并发的时候，保证我执行不会被打断。


我们使用语言的原子操作，sync（），lock（）都是对底层CPU指令的封装。
底层CPU指令的封装，又是基于硬件设计的简单化，或者实现的效率考虑。
底层的门电路实现。


#8.4 互斥

定义： 
==顺序存储模型，多个CPU同时**读写相同内存位置**，有确定次序。== 

这个定义非常经典。


多个CPU同时更新同一个位置，破坏数据结构风险。
例如：写2联系2个Byte，高位写完，CPU被切换掉，另外一个CPU重写了高位，数据的完整性不存在了。

**竞争条件：**执行结果，==取决多个CPU执行指令相对次序或时序。==
> 例如：load－store结构，读内存值到寄存器，寄存器＋1，写回内存。如果2个CPU做同样操作，必须有确定次序。



**竞争critical：**  2个CPU运行各自指令，同时在操作同一个变量或数据结构。
**临界区critical-section：**  这个2组指令序列－－2个代码段，称为临界区。

**临界资源 critical－resource：**   被多个CPU同时读写的，变量或数据结构。

**互斥 mutual－exclusion：**  一次只有一个CPU进入临界区执行。


#8.5 单处理器UNIX系统互斥
 
## 8.5.1 短期互斥

场景： 几个微秒，**==2个内核进程同时修改一个数据结构==**，单处理器一次处理一个进程，
内核中支持抢占（干到一半被抢）支持内核时间片执行，才回发生竞争。

解决： 内核设计成**==内核态执行时非抢占==**，不用修改内核代码，不支持内核时间片执行。

实现：critical section代码特别短，仅修改一个结构体。

单处理器如果==支持抢断，那么互斥必须使用同步加锁和解锁==。

> A进程加锁后，被B进程占用，
> B要锁要不到，时间片到了，再A使用继续执行。
> B根本无法执行，被抢占无意义。


## 8.5.2 中断处理器互斥

场景：**中断回调函数，什么时候被调用都不知道。**

中断处理代码和驱动代码（base－level code）使用同一数据结构。
内核态执行的代码会==*临时禁止中断。*== ，需要修改代码。

> 禁止中断 s＝splhi（） 设置成最高中断优先级
> counter＋＋
> 恢复中断 splx（s），恢复过去的中断优先级

spl＝set priority level 设置优先级，中断优先级降低，处理器忽略中断。



## 8.5.3 长期加锁   

(单处理器UNIX系统互斥)

场景：**几个毫秒到几秒量级的互斥** 

* A进程写文件a，防止别进程同时写a文件。
* A拿到写🔒，B没拿到写🔒，B睡觉让出CPU，让CPU更加有效率。
* A释放了写🔒，B才有可能获得🔒。



==**sleep（address）／walkup（address）**内核函数是针对某个地址发生变化事件，唤醒的函数。==
我们平时用 sleep（time），时间条件进行让出CPU，唤醒，重新执行。

```
//flag_ptr 存放某个大结构体中的锁变量－内存位置，
//这个值应该 voliate,多cpu必须访问内存，防止这个变量的并发问题
void lock_object(char *  flag_ptr){
  while (*flag_ptr) 
    sleep(flag_ptr)；
    
//如果值是1，一直休眠，除非这个这个锁变量值被改变，被唤醒,放入待运行队列，再竞争上岗
//但是while 而不是if ，是因为while是继续去竞争获得锁，因为N个CPU在等待这个资源释放 
//sleep是让出cpu,让cpu干别的事情，
//比纯while（＊flag_ptr）{ sleep(1)} 睡1秒，这种CPU占用少。

	＊flag_ptr = 1 ; //上锁 
}
```

**但是while(*flag_ptr),这个判断问并发问题**（多CPU同时都到判断，然后改变量），
读－改－写 或 测试－设置的措施，来保证这部分的并发。
如果单核并发则没这个问题。


解锁

```
void unlock_object(char * flag_ptr){
	* flag_ptr = 0 ; //解锁
	walkup( flag_ptr );  //手工唤醒
}

```
.
.

**sleep（要互斥的数据结构的地址），为什么这样设计接口？**
快速过滤，这个地址上等待的线程是很方便的。


### sleep函数调用过程：
1. a进程调用sleep函数，
2. 在一张表记录下a进程事件。随后wakeup唤醒这些关注这个事件的线程。
3. 执行一次上下文切换，选择另外一个进程。
4. a进程不再执行，直到被唤醒。

### wakeup函数调用过程:
1. wakeup(flag）清除标志。
2. 唤醒**所有**等待这个锁的进程，放入等待队列。从sleep表中选择等待这个锁。如果没有人等待就返回。
3. wakeup之后，sleep函数的线程还会进入休眠列表。
4. scheduler调度器，选择某个进程，从上次停止的地方开始执行。
5. 假如是sleep函数的那个线程被调度执行（可能多个被唤醒的sleep线程），那么sleep函数返回执行下去，while 重新去判断锁，有可能获得说，有可能无法获得锁。

特别疑问，因为是单核核心，短期互斥这一节，选择了非抢占式，所以 while（判断）－上锁，这个步骤不需要额外的锁。





这部分又和linux进程调度关联了。
引入一个关联图。


## 8.6 多处理器上跑单处理器UNIX会发生的问题

在多处理器MP上使用为单处理系统设计的UNIX会有哪些问题？

在多处理器场景下

1. **短期互斥失效**
单处理设计的内核，无运行在多处理器上（MP），因为多处理器如果不抢占，那么内核一个CPU一个线程在运行，效率低，否则违反短期互斥的假说（不抢占），否则内核结构体被破坏。

2. **中断互斥失效**
中断处理可能失效，spl函数屏蔽最高中断优先级，只会影响所在的处理器的处理优先级。
中断硬件设计，可以在传递给系统中任何一个CPU，或定向传递给一个CPU。

3. **长期互斥失效**
长期互斥依靠短期互斥（内核非前瞻）

长期加锁面临**2**个问题

- 2个进程都以为自己获得了锁，都去关门，都在执行。 
   同时有N个线程进入，都执行了了①和② 。  

- 一个进程判断到锁被占用准备自己睡眠，另外一个进程正在释放锁和唤醒，那么睡眠的进程将**不被唤醒**，直到下一个用完锁释放的人。
  A线程做完①还没做②，B线程在③和④并且一起做完了，那么②就陷入无人唤醒的睡眠中。

```c 
void lock_object(char *  flag_ptr){
  while (*flag_ptr) ①
    sleep(flag_ptr)；② 
  ＊flag_ptr = 1 ; //上锁 
}
```

```c
void unlock_object(char * flag_ptr){
	* flag_ptr = 0 ; //解锁          ③
	walkup( flag_ptr );  //手工唤醒   ④
}
```



## 8.7 小结

fault tolerance  容错





～～～～～～～～～～～～～～～～～～～～～～～～～～～～
旁白：


乱序执行技术 编辑
乱序执行（out-of-order execution）是指CPU采用了允许将多条指令不按程序规定的顺
序分开发送给各相应电路单元处理的技术。比方Core乱序执行引擎说程序某一段有7条指令，
此时CPU将根据各单元电路的空闲状态和各指令能否提前执行的具体情况分析后，
将能提前执行的指令立即发送给相应电路执行。




针对同一个存储位置，多个CPU同时读或写，不保证确定次序。也就不确定结果。
多个CPU同时更新同一个位置，破坏数据结构风险。
次序不确定引发竞争。

执行结果，取决多个CPU执行指令，相对次序于时序。

RISC处理器，load－store结构，
load从内存取数据，CPU只计算寄存器，store把寄存器计算结果返回到内存。
高速度结构简单。


多处理器编程的艺术
	
	
多个CPU同时访问主存储器，总线硬件在多个requestor仲裁,确定一个CPU，CPU允许一次读或写,连续的多个字，用完后释放权限，循环仲裁,找下一个CPU

一个CPU一次读一个高速缓存行,假设传输一个字,一个周期

限制每次读，不能读太多，占用总线太长。为了均匀进程处理速度。
阻塞问题?引申出cpu延时,os无法知晓的

因为总线的带宽-无法满足同时大量数据传输 ， 一个cpu读取主存的量,







	


