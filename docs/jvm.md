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

## 类加载器

### 类的生命周期
1. JVM加载字节码，变成持久化/元数据区的class对象，接着才会执行程序逻辑，加载的过程：加载loading，链接linking，初始化initializing
2. 一个类在JVM的生命周期有7个阶段：加载loading， 验证 verification, 准备Preparation，解析Resolution, 初始化initialization，使用Using，卸载Unloading; 类加载是前5个阶段
- 加载：（装载）根据class的完全限定名来获取二进制classfile的字节流，查找classpath中的jar文件或者class文件，如果找不到，抛出NoClassDefFound错误；这里不检查语法和格式，是JVM和具体的类加载器协作完成
- 校验：确保class文件中的字节流信息符合当前虚拟机的要求，不会危害虚拟机的安全；比如判断常量池中的符号，并执行类型检查，以及版本检查等，可能抛出：VerifyError，CLassFormatError，UnsupportedClassVersionError；这个过程是链接过程的第一步，可能需要加载其他类，比如超类和接口，这里也可能抛出：CLassCircularError，IMcompatibleClassChangeError
- 准备：创建静态字段，并将其初始化为标准值，并分配方法表，即在方法区分配这些变量所使用的内存空间，但并不会执行任何java代码
- 解析：解析常量池，就是字节码中的符号引用，字节码中是一个索引记录，解析阶段将其链接为直接引用，指向实际对象，引用的目标在堆中存在
- 初始化：在类的首次使用的时候才会执行类的初始化，比如构造方法，static静态变量赋值语句，static静态代码块。
3. 如果A类引用了B类，加载A类的时候并不一定会加载B类，只要当主动对B类执行第一条指令时才会导致B类的初始化，这时会先对B类进行装载和链接

### 类加载时机
1. 当虚拟机启动时，初始化用户指定的主类，就是启动执行的main方法所在的类
2. 当遇到新建目标类实例的new指令时，初始化目标类，
3. 当遇到调用静态方法的指令时，初始化静态方法所在的类
4. 当遇到访问静态字段的指令时，初始化该静态字段所在的类
5. 子类的初始化会触发父类的初始化
6. 如果一个接口定义了default方法，那么该接口的实现类的初始化会触发该接口的初始化
7. 使用反射API对类进行反射调用时，初始化这个类
8. Class.forName(), clasLoader.loadClass()等JavaAPI, 反射API，JNI_FindClass都可以启动类加载，JVM本身也会进行类加载，JVM启动的时候加载核心类java.lang.Object, java.lang.Thread等

### 类加载机制
1. 类加载过程：通过一个类的全限定名a.b.c.XXClass来获取此类的class对象，这个过程是通过类加载器完成
- 启动类加载器 BootstrapClassLoader: 加载java的核心类（JDK中的jre/lib/rt.jar里所有的class），代码层面无法获得，String.class.getClassLoader() => null 
- 扩展类加载器 ExtClassLoader: 加载JRE的扩展目录 lib/ext, 或者java.ext.dirs指定的目录中的jar包的类
- 应用类加载器 AppClassLoader： JVM启动时加载来自java命令的-classpath或者-cp选项，java.class.path指定的jar包和类路径
- BootstrapClassLoader是拿不到的，ExtClassLoader和AppClassLoader继承自URLClassLoader，
2. 特点
- 双亲委托：当一个自定义类加载器需要加载一个类时，会先委托父类加载器去加载，只要上级找到，就不用自己加载了，如果都找不到，ClassNotFoundException
- 负责依赖: 当加载一个类时，发现该类依赖其他类或接口，也会尝试加载这些依赖
- 缓存加载： 为了提高效率，一旦一个类加载，会缓存这个接在结果，不会重复加载
3. 自定义类加载器：都以应用类加载器作为父加载器
- 因为加载过程是把class文件以流的形式读出，加载器根据解析做处理，这里可以扩展加密字节码文件，load的时候使用自定义的加载器解密文件并加载
- 两个没有关系的自定义加载器之间加载的类时不共享的，这样可以实现不同的类的隔离性，可以使用不同的类加载器各自加载一个类的不同版本，互相之间不影响，这个基础上可以实现类的动态加载卸载，热插拔的插件机制，比如OSGI的模块化技术


### 问题
1. 排查找不到jar的问题：打印出各个类加载器加载的jar路径
- 启动类加载器：sun.misc.Launcher.getBootstrapClassPath().getURLs()
- 扩展类加载器：this.getClass().getClassLoader().getParent() -> .. 通过反射查询classloader加载的类
2. 排查类方法不一致问题
- NoSuchMethodError: 很可能是加载了不同版本的jar，可以查看加载的jar列表
3. 查看加载的类以及加载顺序
- 参数-XX:+TraceClassLoading 或者-verbose
4. 调整或修改ext和本地加载路径
- 参数：sub.boot.class.path   java.ext.dirs 
5. 运行期加载额外的jar包或者class
- 自定义classloader的方式
- 在当前的ClassLoader借助放射使用URLClassLoader的addURL方法，添加完使用Class.forName显式加载和初始化

## 内存模型
Java 内存模型规定了JVM如何使用计算机内存RAM

### JVM内存结构
1. 分为线程栈Thread stacks和堆内存Heap两部分
2. 线程栈
- 每个正在运行的线程都有线程栈，包含正在执行的方法链和所有方法的状态信息，
- 用于存储栈帧，栈帧包含操作数栈，局部变量数组，class的引用
- 每个线程都只能访问自己的线程栈，不能访问其他线程的局部变量
- 如果两个线程执行相同的代码，那么每个线程都会在自己的线程栈内创建对应代码中声明的局部变量
3. 堆内存
- 堆内存包含了java代码中创建的所有对象，不管是哪个线程创建的。
- 不管是创建一个对象给局部变量，还是赋值给另一个对象的成员变量，常见的对象都会保存到堆内存中。
- 如果是原生数据类型的局部变量，内容会全部保留到线程栈上，如果是对象引用，则栈中的局部变量槽位保存的是对象的引用地址，实际的对象在堆内存中
- 对象的成员变量与对象本身一起存储在堆中，不管成员变量是原生类型还是引用类型
- 类的静态变量和类定义一样都保存在堆中
- 例子
  - 线程t1-线程栈：method1(0: this-ref, 1: 局部变量var1); static_method2(param1)(0: param1, 1: 局部变量varX) // 每个方法是一个栈帧
  - 线程t2-线程栈：method1(0: this-ref, 1: 局部变量var1); static_method2(param1)(0: param1, 1: 局部变量varX)
  - 堆内存：object1(obj,obj), object2(obj,obj)...
- 原始数据类型和对象的引用地址在栈上，对象，对象成员与类定义，静态变量在堆上（加载类的时候，把元数据存储在了元空间中，class对象，静态变量都在堆中）
4. 方法区：是JVM的规范，线程共享的，存储类的信息，常量池，方法数据，方法代码等（指令和code）
- 方法区在不同JDK上实现不一样，JDK6使用永久代PermGen实现，常量池也在永久代；JDK7使用永久代实现，但常量池移到了堆中；JDK8废弃了永久代，使用Metaspace元空间实现，常量在堆中；
- 早期的永久代也占用堆空间，但元空间是占用物理内存
- 元数据信息存储在元空间中（方法区 JDK8实现），但是class对象存储在堆内存中

### 栈内存结构

1. 每启动一个线程，JVM就会在栈中分配对应的线程栈，比如1MB的空间(-Xss1m), 如果是JNI方法，则会分配一个单独的本地方法栈Native Stack 
2. 线程执行过程中，是很多方法的调用，每执行一个方法，就会创建对应的栈帧Frame
3. 每个栈帧由操作数栈，局部变量表，返回值，以及class/Method指针

### 堆内存结构

1. 堆内存是线程公用的内存空间，逻辑上分为Heap和Non-Heap, Java代码基本只能使用Heap，内存的分配和回收也是在Heap，也就是GC管理的堆，Non-Heap不需要关注（元数据区）
2. GC理论最重要的思想就是分代，程序中的对象要么用过就扔，要么存活很久
3. JVM把Heap分为年轻代Young Generation和老年代Old Generation两部分
4. 年轻代划分为新生代Eden space，存活区Survivor space, 大部分的GC算法中有两个存活区S0, S1, 任何时刻S0和S1总有一个是空的
5. 新生代的优化：TLAB Thead Local Allocation Buffer - 每个线程先划定一片空间，创建对象先在这里分配，降低并发资源锁定的开销

### JMM
1. JMM（Java Memory Model）规范明确定义了不同线程之间操作共享变量的问题，必要时需要同步操作
2. 能被多个线程共享使用的内存称为共享内存，或者堆内存
3. 所有的对象，static变量，数组都必须存在堆内存
4. 局部变量，方法的形参、入参，异常处理语句的入参不允许在线程之间共享，所以不受JMM影响
5. 多线程同时对一个变量访问时，这时只要有一个线程是写操作，就会出现冲突
6. 可以被其他线程影响的操作，比如读取，写入，同步操作，外部操作等，同步：volatile变量的读写，对monitor的锁定和解锁，线程的起始操作与结尾操作，线程启动和结束等等，外部操作是指对县城执行环境之外的操作

### CPU指令
1. 计算机支持的指令大致为两类：精简指令集，复杂指令集，复杂指令集类似整合一系列操作为一个指令
2. 任何一个指令集，对应CPU的实现都是采用流水线方式，如果CPU一条条指令的执行，那么很多CPU内核会闲置着，于是硬件设计人员就想出一个方法：指令乱序；
3. 指令乱序：CPU根据需求通过内部调度把指令打乱执行，充分利用CPU内核，只要结果等价就可以；但随着复杂度提升，并发执行的问题也很多。
4. 因为CPU会在合适时机对指令重排，但有时候这个重排会导致代码的预期不一致，因此，引入了内存屏障机制
5. 内存屏障可分为读屏障和写屏障，用于控制可见性，常见的内存屏障有：#LoadLoad #StoreStore #LoadStore #StoreLoad 
6. 这些屏障的目的是用来短暂屏蔽CPU的指令重排功能，看见这些指令时，就要保证这个指令前后的操作不会被打乱
- LoadLoad: 屏障前面的Load指令要一定执行完，才能执行屏障后面的Load指令
- StoreStore: 屏障前面的写，写完之后才能执行屏障后面的写
- LoadStore: 屏障前面的Load，执行完之后，才能进行写操作
- StoreLoad: 屏障前面的写，执行完之后，才能Load 
7. 当CPU收到这些指令时，就会做一些操作，同时发出广播，给某个内存地址打个标记，其他CPU内核和自己的缓存交互时，就知道这个缓存不是最新的，需要从主内存重新加载。

### 总结
1. JVM的内存区域分为堆内存和栈内存
2. 堆内存的实现可分为两部分：堆Heap和非堆Non-Heap
3. 堆主要由GC负责管理，按分代的方式一般分为：老年代+年轻代；年轻代=新生代+存活区；
4. CPU有一个性能提升的利器：指令重排序
5. JMM的规范对应的是JSR133,现在由Java语言规范和JVM规范维护
6. 内存屏障的分类与作用













