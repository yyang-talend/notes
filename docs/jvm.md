# JVM

## 常用性能指标

1. 业务指标
- 吞吐量（QPS, TPS）
    - 交易类的系统使用每秒处理的事务数（TPS）来衡量吞吐能力
    - 查询搜索类的系统使用每秒处理的请求数（QPS）
- 响应时间（RT）
    - 平均响应时间，P95/P99，Max
- 并发数
- 业务成功率
    - KO, OK
- 系统容量

2. 资源指标
- CPU：使用率，运行队列，上下文切换
- 内存：可用内存，swap占用，页面交换（Paging）
- Disk I/O: 使用率，IOPS，数据吞吐量
- Network IO：网络吞吐量

3. 性能调优
- 制定指标，收集数据
- 分析问题，找到瓶颈
- 制定方案，调整配置
- 逐步改进，持续观察

## 资源指标在Linux中获取方式

- 常用的收集工具：vmstat, iostat, iptraf, nmon, JMeter的PerfMon插件, 
```sh
[hbase@ecs-097 ~]$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 6  0      0 4591436 176804 1185380    0    0     0     0 7915 10357 83  5 12  0  0
```
```
[hbase@ecs-097 ~]$ top
top - 19:23:28 up 18:05,  3 users,  load average: 0.80, 0.60, 0.53
```

### CPU
1. 进程
- 常用状态Running, Waiting, Blocked(等待IO)
- r (Running + Waiting), b （Blocked）

2. 使用率
- us - 用户占用CPU的百分比, sy - 系统占用CPU的百分比, id - CPU空闲的百分比； 
- 使用率 = us + sy, 可接受上限通常为70% - 80%
- sy: 如果长期大于25%，需要关注in(系统中断)和cs（上下文切换）

3. 运行队列
- Running + Waiting 的进程数，vmstat的r值 
- 如果r值等于系统CPU总核数，说明已经满负荷。负载测试中，可接受上限是不超过核数的2倍

4. 上下文切换
- 上下文context是CPU寄存器和程序计数器在某时间点的内容，上下文切换context switches就是kernel挂起一个进程并存状态到内存，然后从内存中恢复下一个要执行的进程状态
- 频繁的上下文切换会导致sy增长，系统占用CPU的百分比
- cs：每秒上下文切换次数

5. 平均负载Load Average：top
- 一段时间的平均负载，top: 1mins, 5mins, 15mins的平均负载值，如果15mins过高负载，可以查看vmstat的r和b值，来确定CPU负载高还是等待IO进程多。

### Memory
1. 物理内存和硬盘上的一块空间SWAP组合起来作为虚拟内存，为进程提供一个连续的内存空间，SWAP的读写速度远低于物理内存，
2. 虚拟内存被分成页x86系统的默认页大小是4k,内核读写虚拟内存是以页为单位
3. 当物理内存不足时，内存调度会把物理内存上不常使用的内存页内容存储到磁盘的SWAP空间，物理内存和Swap空间之间的数据交换称为页面交换Paging
4. 可用内存：vmstat -> free，可用内存过小影响系统的运行效率，可接受范围是大于物理内存的20%，压力测试时，如果可用内存很小，页面交换也很少，回对系统性能造成严重影响。
5. swap占用：vmstat -> swpd, 当前swap空间的使用情况，和页面交换一起看，如果swap不为0，但si,so为0，内存资源不是系统的瓶颈
6. 页面交换Paging：vmstat - si(swap读取到内存), so（内存写入到swap）; Swap和内存的交换，如果频繁交换，需要注意

### Disk
```
[hbase@ecs-097 ~]$ iostat -dxk 1
Linux 2.6.32-504.3.3.el6.x86_64 (ecs-097)   08/01/2016  _x86_64_    (4 CPU)
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.52    0.00    0.13    0.06    0.00   99.28
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await  svctm  %util
xvda              0.10     6.63    0.40    2.57     6.22    36.80    29.00     0.04   14.63   1.19   0.35
```
1. 磁盘是系统中最慢的一部分，主要是自身速度慢，离CPU最远
2. 磁盘IO分为随机IO和顺序IO，先了解系统偏向哪种类型
3. 随机IO：随机读写数据，读写请求多，数据量小，IO速度更依赖于磁盘每秒IO次数，IOPS 
4. 顺序IO：顺序请求大量数据，但读写请求少，更重视每次IO的数据吞吐量
5. 使用率：使用过程中处理IO请求的时间与统计时间的百分比，即iostat中的%util，如果大于60%，很可能降低系统的性能
6. IOPS：每秒读写请求的数量，iostat中的r/s,w/s, 一般在100左右
7. 数据吞吐量：iostat中的rKb/s, wKb/s 

### Network
```
[root@ecs-097 ~]# iptraf -d eth0
x Total rates:         67.8 kbits/sec        Broadcast packets:            0                                                                                                                x
x                      54.2 packets/sec      Broadcast bytes:              0                                                                                                                x
x                                                                                                                                                                                           x
x Incoming rates:      19.2 kbits/sec                                                                                                                                                       x
x                      25.4 packets/sec                                                                                                                                                     x
x                                            IP checksum errors:           0                                                                                                                x
x Outgoing rates:      48.7 kbits/sec                                                                                                                                                       x
x                      28.8 packets/sec 
```
1. 常规的服务端性能测试都是在局域网中，因为首先需要关注自身的性能，所以只需要关注网络吞吐量即可，稳定系统之后，需要在外部给出足够的带宽
2. iptraf  

## 字节码

### 指令
```
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class test/Hello
         3: dup.....
```
1. java字节码由单字节指令组成，8bit = 256， 理论上256个操作码，java使用了200左右
2. JVM 规范： https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-7.html https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html 中文：https://www.jianshu.com/p/e057695f1184 
3. javap：反编译class文件
- javac: 编译，生成的字节码中默认没有LocalVariableTable, 编译时使用 -g: Generate all debuging info 
- javap: -c -v 
4. 常量池Constant pool: `javap -verbose test.HelloWorld` 输出附加信息（编译版本，编译时间，MD5，类的详细信息），常量池中有#1，#2 代表对常量池的引用，对应的值有Class，Methodref..
5. 方法信息：descriptor，flags，Code（stack 栈深度，args_size: 参数size- 非静态方法默认有this参数）
6. stack: 执行该方法时栈的深度（默认构造方法时1，main方法是2）；locals (局部变量size)， args_size(参数size)
7. Code中的0: 3: -> new是java opcode，十六进制值是bb，占用一个位置,#2 分别占用两个位置，dup开始的位置就是3号槽位，class文件用16进制编辑器可以看到存储这个方法的部分，java中的asm，javassist之类的字节码操作工具就可以修改字节码
8. 对象初始化指令：
- new dup invokespecial一起是创建类的实例对象： new只创建对象，invokespecial 调用构造函数，dup复制栈顶的值
- astore {N}或astore_{N} 赋值给局部变量，N是局部变量表中的位置
- putfield - 将值赋给实例字段
- putstatic - 将值赋给静态字段
9. 栈内存操作指令
- dup复制栈顶的值
- pop删除最顶的值
- swap交换栈顶两个值的位置
- 给局部变量赋值时，需要指令store，比如astore_1,store指令会删除栈顶值，相应load指令会将值从局部变量表压入操作数栈
10. 流程控制指令
- 分支和循环，根据检查条件来控制程序的执行流程
- for循环会产生隐藏的局部变量协助指令执行
- if_icmpge if integer compare great equal
- goto
11. 算术运算指令：iadd isub imul(*) idiv irem(%) ineg(-{})
12. 类型转换指令：i2l i2f i2d i2b(byte) i2c(char) i2s(short)
13. 方法调用指令和参数传递：invokespecial invokestatic invokevitual invokeinterface 
14. invokedynamic : 动态类型语言，比如Lamda表达式

### 线程栈
1. 每个线程有自己独有的线程栈JVM stack, 用于存储栈帧Frame
2. 方法调用时JVM都会自动创建一个栈帧Frame，栈帧由操作数栈Operand Stack，局部变量数组Local variables，以及一个class引用组成(指向当前方法在运行时常量池Constant pool中的class)
3. 局部变量数组：就是局部变量表LocalVariableTable,包含了方法的参数，以及局部变量。数组的大小在编译时就确定了，和局部变量和形参的个数有关，还要看每个变量、参数占用多少字节
4. 操作数栈：LIFO后进先出结构的栈，用于压入和弹出值，指令操作时会按需压入和弹出






