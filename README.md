# memory
内存主要用来存储系统和应用程序的指令，数据，缓存等

---
## 操作系统内存管理
* 段式内存管理
程序需要多少内存就分配多少实际的物理内存，通过段表，段选因子和段内偏移量计算出物理内存地址；<br>
2个问题
  * 外部内存碎片 会导致多个不连续的小物理内存，导致新的程序无法启动
  * 内部内存碎片 程序启动时候，会把所有的物理内存申请到，而程序大部分内存是不常用的，会导致内存浪费
* 页式内存管理
页式内存管理是虚拟内存分成一个个相等的大小页(4K),通过页表（MMU）管理虚拟内存跟物理内存的映射关系，用页号+页内偏移量
计算出实际物理内存地址。申请内存时候只是申请了虚拟内存，当使用真正物理内存中，通过页表查找虚拟内存跟物理内存的映射关系
未找到物理内存时，会发生缺页中断

## 内存映射
linux内核给每个进程都提供了一个独立的虚拟的地址空间，并且这个空间是连续的，这样进程就可以很
方便的访问内存，应该是虚拟内存了
虚拟地址空间的内部分成**内核空间**和**用户空间**2部分，不同字长的处理器，地址空间的范围也不同；
32位和64位系统，如下所示

<img src="https://github.com/lys861205/memory/blob/master/virtual_memory.png" width="400" heigth="500">

32位系统内核空间占用1G，位于最高处，剩下的3G是用户空间。64位系统的内核空间和用户空间都是128T

既然每个进程都有这么大的地址空间，那么所有的进程的虚拟内存加起来比实际的物理内存大的多，
所以并不是所有的虚拟内存都会分配物理内存，只有实际使用的虚拟内存才分配物理内存，并且分配
后的物理内存是通过**内存映射**来管理的。

内存映射其实就是把**虚拟内存地址**映射到**物理内存地址**，为了完成映射，内核为每个
进程都维护一张页表，记录了虚拟地址与物理地址的映射的关系；如下图所示

<img src="https://github.com/lys861205/memory/blob/master/vitural_physial_memory.png" width="400" heigth="700">

页表实际上存储在CPU的内存管理单元MMU中，处理器就可以通过硬件，找出要访问的内存。

当进程访问的虚拟地址在页表中查不到的时，系统会产生一个**缺页中断**，进入内核空间分配
物理内存，更新进程页表，最后返回用户空间，恢复进程的运行

## 虚拟内存空间分布
32位系统如下

<img src="https://github.com/lys861205/memory/blob/master/virtual_layout.png" width="300" heigth="400">

通过这张图可以看到，用户空间内存，从低到高分别是5种不同的内存段

1. 只读段，包括代码和常量
2. 数据段，包括全局变量等
3. 堆，包括动态分配的内存，从低地址开始向上增长
4. 文件映射段，包括动态库，共享内存等，从高地址开始向下增长
5. 栈，包括局部变量和函数调用的上下文等，栈的大小是固定，一般是8MB

这5段内存中，堆和文件映射的内存是动态分配的，用C的sbrk,brk, mmap 就可以分配堆和文件映射段动态分配内存

## 内存的分配与回收
### 内存的分配
libc库通过malloc申请内存，系统底层调用brk, mmap分配内存
对于小块内存(小于128K) 使用brk来分配，这些内存释放后并不会立刻归还系统，被缓存起来，这样就就可以重复使用

大块内存(大于128KB) 直接使用内存映射mmap分配，也就是在文件映射段找一块空闲内存分配出去

**2种方式各有优缺点**
```
brk() 方式的缓存，可以减少缺页异常的发生，提高内存访问效率。不过，由于这些内存没有归还系统，
在内存工作繁忙时，频繁的内存分配和释放会造成内存碎片。
```
```
而 mmap() 方式分配的内存，会在释放时直接归还系统，所以每次 mmap 都会发生缺页异常。
在内存工作繁忙时，频繁的内存分配会导致大量的缺页异常，使内核的管理负担增大。这也是 malloc 只对大块内存使用 mmap  的原因
```

### 内存的回收
系统不会任由某个进程用完所有的内存，在发现内存紧张时，系统通过一系列机制完成回收内存
* 回收缓存 使用LRU
* 回收不常访问的内存，把不常用的内存通过交换分区直接写到磁盘
* 杀死进程，内存紧张时系统通过OOM(out of memory),直接杀掉占用内存的进程

## 查看内存使用情况
Linux下通过free工具可以查看内存的使用情况
`free -h`
```
              total        used        free      shared  buff/cache   available
Mem:           376G         65G         46G        4.2G        263G        303G
Swap:            0B          0B          0B
```
|total|used|free|shared|buff/cache|available|
|-----|------|-----|-----|------|------|
|内存总大小|已使用内存大小,包括共享内存|未使用内存大小|共享内存大小|缓存和缓冲区大小|可用内存大小|
```diff
! buffer是对磁盘数据的缓存，而Cache是对文件数据的缓存，它们既会用在读请求中，也会用在写请求中
```


可以通过top工具，查看具体进程内存使用情况
`top`
```
top - 15:40:27 up 751 days, 18:30, 20 users,  load average: 1.01, 1.04, 1.05
Tasks: 847 total,   1 running, 846 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.3 us,  0.0 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 39464640+total, 49004744 free, 69201728 used, 27643993+buff/cache
KiB Swap:        0 total,        0 free,        0 used. 31859532+avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
   
400393 root      20   0  0.111t 0.058t  67628 S   0.0 15.7  14:26.57 IAServer

271306 quhang    20   0 9544604 1.537g   7912 S   0.0  0.4   6:06.27 cpptools

450356 lys       20   0 6416576 772888   2508 S   0.0  0.2  47:12.47 access_server
```
* VIRT 是进程虚拟内存的大小，只要是进程申请过的内存，即便还没有真正分配物理内存，也会计算在内
* RES 是常驻内存的大小，也就是进程实际使用的物理内存大小，但不包括 Swap 和共享内存
* SHR 是共享内存的大小，比如与其他进程共同使用的共享内存、加载的动态链接库以及程序的代码段等
* %MEM 是进程使用物理内存占系统总内存的百分比

第一，虚拟内存通常并不会全部分配物理内存。从上面的输出，你可以发现每个进程的虚拟内存都比常驻内存大得多。

第二，共享内存 SHR 并不一定是共享的，比方说，程序的代码段、非共享的动态链接库，也都算在 SHR 里。当然，
SHR 也包括了进程间真正共享的内存。所以在计算多个进程的内存使用时，不要把所有进程的 SHR 直接相加得出结果




