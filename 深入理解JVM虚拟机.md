#  深入理解JVM虚拟机

# 自动内存管理

## JAVA内存区域与内存溢出异常

对于c++/c程序开发人员来说，拥有对内存的最高权力，但是又从事最基础工作的劳动人民-管理每个对象的生命周期

java则都交给了JVM虚拟机，但是如果不了解JVM 造成内存泄漏和溢出方面的问题，那么修正错误将变得异常困难

## 运行时数据区域

java虚拟机在执行java过程中把它管理的内存分为若干个不同的数据区域 这些区域都有各种的用途，有的随着虚拟机进程的启动一直存在，有的则依赖用户进程的启动和结束

![image-20201205193713101.png](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201205193713101.png)

### 程序计数器PC

是一块较小的内存空间，它可以看作是当前线程所执行的字节码行号指示器，字节码解释器工作的时候就是通过改变这个计算器来选取下一条需要执行的字节码指令，它是程序分支，循环，跳转的基础

### JAVA虚拟机栈

与程序计数器一样，也是线程私有的。每个方法被执行的时候，会创建一个栈帧用于存储局部变量表，操作数栈，动态链接，方法出口等信息，每一个方法被调用直到完成，都对应这一个栈帧在虚拟机栈中入栈出栈的过程

局部变量表存放了编译器可知的各种java基本数据类型，对象引用等

这些数据类型在局部变量表中的存储空间以局部变量槽来表示，其中64位长度的类型占用2个变量槽，当进入一个方法时，这个方法需要在虚拟机栈分配多大的内存空间完全时确定的

(譬如按照1个变量槽占用32个比特、64个比特，或者更多）来实现一个变量槽，这是完全由具体的虚拟机实现自行决定的事情。)

### 本地方法栈

与JAVA虚拟机栈类似，区别只是JAVA虚拟机执行JAVA方法，本地方法栈执行本地任意语言方法

### JAVA堆

java虚拟机规范里面对java堆的描述：所有的对象实例以及数组都应当在栈上分配

JAVA堆是垃圾收集器的管理区别，因此也被称为GC堆，从回收内存的角度来看，由于现代垃圾收集器大部分都是基于分带收集理论设计的，所以JAVA堆中经常会出现新生代，老年代等名词

如果JAVA堆中没有内存可分配，或者堆也无法扩展时 会抛出 OutOfMemoryError异常

### 方法区

方法区和JAVA堆一样，是线程共享的，它用于存储以被虚拟机加载的类型信息，常量，静态变量等

JAVA8之前很多人把方法区称作永久代

由于内存泄漏的原因，未来的发展并且移植Jrockit优秀功能等原因 移除了永久代 。

改用在本地内存实现的元空间来代替	

在过去 由于类calss是jvm实现的一部分，又不是由应用创建的，所以又被认为是非堆内存

类class包括类的版本，字段，方法，接口，静态变量等 而且又很少被卸载或者收集  所以被称为永久的

永久代的垃圾收集和老年代捆绑在一起，因此无论谁满了都会触发两者的垃圾收集，会导致大量的性能问题和OOM错误 

JDK7开始永久代的移除工作，储存在一部分数据已经转移到了JAVA堆或者本地内存 但是永久代仍然存在于java7

如果方法区无法满足新的内存分配需求时，会抛出OOM(OutOfMemoryError)异常

**方法区主要存放类的版本号，方法，常量池，接口数组，等  常量池又分为符号引用，字面量**

**符号引用有：类的全限定名，字段名和属性，方法名和属性**

**字面量：被定义为final的常量，文本字符串，基本数据类型的值，等**

![image-20201210182021876](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210182021876.png)

![image-20201210182049522](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210182049522.png)

### 直接引用和符号引用和字面量

直接引用：直接引用是指可以直接指向目标的指针

符号引号：是指编译器，java类不知道所引用类的具体地址，只能用符号来引用 比如Person类引用了address类，但是编译期间不知道address类的具体地址只能用符号来替代

字面量：如String 类型的 “ABC”，可以称为字面量ABC，就是描述自己的量	

### 运行时常量池

运行时常量池是方法区里面的一部分，常量池表用于存放符号引用和字面量等，这部分数据会被类加载器加载后存放到方法区的运行时常量池中

JAVA对class文件的每一部分都有严格的规定，否则不被虚拟机所认可 但是对于运行时常量池《《JAVA虚拟机规范》》没有任何细节的要求，不同厂商可以按照自己的需求来实现运行时常量池、运行时常量池相对于CLASS文件常量池还具备动态性，也就是说可以在运行期间将新的常量放入池中

运行较多的就是String的intern()方法

### 直接内存

直接内存不是JAVA虚拟机运行时数据区的一部分，也不是java虚拟机规范中定义的内存区域

在jdk1.4中新加入了NIO，是一种基于通道和缓冲区的I/O方式，它可以使用Native函数直接分配堆外内存，然后通过堆中的DirectByteBuffer对象作为这块内存区别的引用进行操作。

避免了JAVA堆和Native堆中来回复制数据，增加性能

但是受物理内存的影响 也就是本机总内存

# HotSpot虚拟机对象揭秘

## 对象的创建

当java虚拟机遇到一条字节码new指令时，会首先去检查这个指令的参数能否在常量池中定位到该类的引用，并且判断该符号引用的类是否已经被加载，如果没有被加载则去进行类加载机制。当类被初始化后

JAVA虚拟机会为该对象分配内存空间。 假设JAVA堆的内存是规整的，一边是未分配内存的，一边是使用过的内存，中间指针作为分界点，当一个对象需要分配内存时指针只需要偏移和对象等大小的距离则代表已经分配好内存，简单又高效。这种分配方式称为指针**碰撞**。 假设此时JAVA堆是不规整的，那么可能分配的内存和未分配的内存交互在一起，那么虚拟机就必须维护一个列表，记录哪些内存是可用的哪些内存是不可用的，为对象分配内存后并更新列表的记录，这种方式称为**空闲列表** 

内存的分配方式取决于JAVA堆的规整程度，JAVA堆的规整程度又取决于垃圾收集器是否具有空间压缩整理的能力

因此使用Serial,ParNew等带由空间压缩整理过程的收集器时，则使用的是指针碰撞，否则是空闲列表

还有问题，对象分配内存在并发情况下也不是线程安全的，可能刚给一个对象分配完内存还没来得及移动或者修改指针，第二个线程又在原有的指针上分配内存

有两种可解决方案，第一种：使用CAS操作，第二种：为每个线程建立本地缓冲区，哪个线程需要分配内存就在本地缓冲区分配，分配完了再用同步锁

分配完之后还需要对对象进行设置，如对象的哈希值，偏向锁信息，对象是哪个类的实例等等

这些信息会存放在对象头重 此时对于虚拟机来说对象已经创建完毕，但是对于程序员来说对象才刚刚开始创建--构造函数

## 对象在内存的布局

对象在堆里面主要可以分为对象头，实例数据，对齐填充

**对象头存放两种信息**

**一种是对象的哈希值，偏向锁信息.GC分代年林，线程ID等等 称为Mark World  32位机器是32BIt 64位机器则是64位Bi**t 

![image-20201228205059172](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228205059172.png)

假设32位机器并且当前处于无锁情况下  那么对象头里面就包含的是  对象的哈希值25位  4位是存储对象分代年龄   2位是锁信息 1位是0

![image-20201208175347862](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208175347862.png)

另外一种类型指针，即对象指向它的类型元数据的指针，通过该指针来确定是哪个类的实例。但是不一定查找对象的元数据需要通过对象本身

自己的理解就是 对象头的另一部分存放的是类里面的各种属性数据地址  

![image-20201208175815637](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208175815637.png)

接下来就是**实例数据**

该地方才是真正存储对象里面数据的，无论是父类继承下来的还是子类继承下来的都需要记录

存储顺序默认为longs/doubles、ints、shorts/chars、bytes/booleans、oops（Ordinary Object Pointers，OOPs） 相同宽度的字段会被分配到一起

第三部分对齐填充，这部分主要是为了满足HotSpot虚拟机自动内存管理要求每个对象必须是8字节的整数倍，所以对象内存不是8字节的整数倍时会填充

## 对象的访问定位

栈上的Reference数据来操作各种对象，一般都会分为句柄和直接引用

句柄相当于可以间接通过一个句柄池来访问对象

句柄池存放的是对象的地址，而栈上的Reference数据则是指向的句柄池

![image-20201208180435837](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208180435837.png)

而直接引用就是直接存放该对象的地址

![image-20201208180444032](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208180444032.png)

句柄池的好处是：稳定 当进行对象移动的时候(垃圾回收的时候会频繁移动) 但是栈里面的引用根本不会受影响，因为是引用的句柄池

直接引用的好处是：速度快

HotSopt而言使用的是直接引用。但是整个软件范围来看，采用句柄池也十分常见

## 实战OOM异常

### 判断堆溢出

```java
public class HeapOOM {
    static class OOMObject{

    }

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<>();
        while (true){
            list.add(new OOMObject());
        }
    }
}
```

![image-20201208183049603](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208183049603.png)

首先要判断是内存泄漏还是内存溢出

如果是泄漏通过GC ROOT找到引用链进行分析，内存映像分析工具

如果是溢出 优化程序

### 虚拟机栈和本地方法栈溢出

JAVA虚拟机规范中定义了两种：

1）如果线程请求的深度大于虚拟机所允许的最大深度，则会StackOverFlowError

2）如果虚拟机的栈允许动态扩展，当无法申请到足够内存时，会抛OOM异常

但是对于HotSpot虚拟机来说，不支持动态扩展

下面测试 1）使用 -Xss减少栈内存容量

​             -Xss128K

![image-20201208204655188](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208204655188.png)

![image-20201208204701481](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208204701481.png)

上面就是超过虚拟机栈所允许的最大深度

第二种  定义大量本地变量，扩大栈帧中本地变量表的长度

![image-20201208204920345](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208204920345.png)

![image-20201208204927368](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208204927368.png)

![image-20201208204931494](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208204931494.png)

事实证明无论是由于栈帧太大，还是虚拟机栈容量太小，当新的栈帧无法继续分配内存时，都抛出的是StackOverFlowError异常

但是如果测试时不限于单线程，那么HotSpot也是可以抛出OOM的，但是和栈关系就不是很大了，因为操作相同为每个进程分配内存是有限制的

![image-20201208205304240](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208205304240.png)

![image-20201208205309505](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208205309505.png)

### 方法区和运行时常量池溢出

运行时常量池也是方法区的一部分.

String::intern()方法是一个本地方法，会返回该字符串在字符串常量池中的引用，如果该字符串常量池中存在则直接返回引用，如果不存在则加入到字符串常量池中再返回该引用 jdk6之前，常量池都是存放在永久代中 如下图

![image-20201208205755583](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208205755583.png)

![image-20201208205804012](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208205804012.png)

可以很明显的看到报出永久代OOM

在jdk7以后 常量池被移到了堆中  以上代码则只会永久的陷入循环，但是限制最大堆就可以发现很明显的区别了 使用-Xmx参数

![image-20201208210206130](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208210206130.png)

关于字符串常量池更有意思的事情

![image-20201208210245263](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208210245263.png)

这段代码在jdk6中会返回两个false,但是在7及以上会返回true 

原因是6中一个是常量池中的引用 一个是堆中的引用 肯定都是false

但是7及以上 字符串常量池被移到了堆中，第一个返回true 因为堆中的常量池保存的是该对象的引用 两个是同一个引用 

第二个是因为字符串常量池已经有它的引用，不符合首次出现的原则 所以为false 计算机软件则是首次出现的所以为true

JDK7只把永久代中的运行时常量池中的字符串常量池移除到了堆中 JAVA8则是废除永久代 改为本地内存的元空间

HopSpot提供了一些参数作为元空间的预防措施：

-XX : MaxMetaSpaceSize  元空间的最大值 默认是-1，即只受限于本地内存

-XX：MateSpaceSize : 元空间的默认大小，以字节为单位 超过该值就会触发垃圾收集器进行类型卸载，同时收集器对该值进行调整 如果释放了大量的空间，就减少该值 让其尽早触发，如果释放不多，则增加该值，让迟点触发

java8和java7一样 符号引用在native heap  常量池在普通堆中  java8主要解决了永久代的内存溢出问题和垃圾回收 使用的是本地内存

## 本地直接内存溢出

![image-20201208212352233](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208212352233.png)

![image-20201208212403632](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201208212403632.png)

很明显的特征就是heap Dump文件中没有什么明显的异常情况，DUMP文件还很小，程序如果直接或者间接使用了直接内存(典型例子就是NIO)，那就可以考虑直接内存了

#  垃圾收集器和内存分配策略

## 对象已死？

在堆里面几乎存放这所有的对象对象实例，在垃圾收集器进行对堆回收时，需要判断哪些是死亡的，哪些是存货的

## 引用计数算法

在对象中添加一个引用计数器，每当有一个地方引用它时，引用计算器就加1，虽然很多地方都使用到引用计数算法对内存进行管理。但是目前主流的JAVA虚拟机都没有使用该算法，主要是因为难解决两个对象相互引用，导致它们的引用技术都不可能为0；

```java
public class ReferneceCountingGc{
    public Object instance = null;
     
    public staic void GC(){
     RefernceCountingGc A =    new RefernceCountingGc();
     RefernceCountingGc B =    new RefernceCountingGc();
        A.instance = B;
        B.instance = A;
        A = null;
        B = null;
        System.gc();
    }
     
}
```

在java程序中是可以被回收的，也侧面证明了java虚拟机不是使用引用计数法判断对象是否存活的

## 可达性分析算法

可达性分析算法是被JAVA领域所使用的，都是通过可达性分析算法来判断对象是否存活的

该算法的基本思路是通过GC ROOTS作为起始节点，从这些节点开始 根据引用关系向下搜索，任何被引用的对象都会通过一条引用链到达GC ROOTS  如果到达不了则认为对象已经无任何引用

用图论的说法就是不可达

![image-20201210174614251](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210174614251.png)

如上图，虽然5 6 7 对象互相引用，但是它们到GC Roots是不可达的，因此被判断是可回收的对象

GC Roots相当于就是**一组必须活跃的引用**

在JAVA体系中，固定可作为GC Roots的对象包括以下几种：

**在虚拟机栈中引用的对象，比如参数，局部变量等**（活跃的栈帧中指向GC堆对象的引用）   

**在方法区中类静态属性引用的对象，比如JAVA类的静态属性变量**

**在本地方法栈中JNI引用的对象**（通常所说的na'tive方法）

**所有被同步锁持有的对象**

上述两种都离不开引用，JAVA的引用JDK1.2之前是很传统的引用，即如果Reference类型的数据存放的是另一块内存的起始地址，那么就称为引用

1.2之后扩展了 强引用，软引用，弱引用，虚引用	

## 生存还是死亡？

即使在可达性分析算法中判断为不可达对象，那么不是”非死不可的“。 因为要宣告一个对象的真正死亡，必须通过两次标记，当对象被第一次判断不可达时，会进行一次标记，随后进行一次筛选，判断是否需要执行finalize()方法，如果没有重写该方法，则再次标记过后就会死亡

如果这个对象被判定需要执行该方法，会放置到一个队列，稍后启用一个低优先级的线程来执行队列里面的方法，但是不承诺一定会等待其结果，因为某个对象的finalize方法如果执行缓慢，可能会造成系统崩溃

如果在finalize上面拯救了自己(只要重新与引用链上的任意一个对象建立关联即可) 那么在第二次标记时它将被移出即将回收的集合

该对象只会被拯救一次，因为任何一个对象的该方法都只会被系统自动调用一次

![image-20201210181158745](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210181158745.png)

![image-20201210181220399](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210181220399.png)

![image-20201210181224700](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210181224700.png)

![image-20201210181229060](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210181229060.png)

但是程序员应该避免使用该方法，没啥用

## 回收方法区

在java堆中，对新生代进行垃圾收集(新生代指的是刚创建的对象，被认为生而夕灭)通常可以回收百分之70到百分之99的内存空间，相比之下方法区回收的条件就显得十分苛刻

方法区主要回收两部分：废弃的常量和不再使用的类型，假如一个字符串java曾经进入常量池，但是当前系统没有任何地方引用到这个字面量，那么就会被回收

不再使用的类型回收起来比较苛刻：

该类的所有实例已经回收，堆中不存在该对象以及派生子类的实例

加载该类的类加载已经被回收

CLASS对象没有被引用，即无任何使用反射的地方

在大量使用动态代理，反射的框架，通常都需要JAVA虚拟机具有类型卸载的能力

## 垃圾收集算法

从对象消亡的角度出发，垃圾收集算法可以分为引用计数式垃圾收集和追踪式垃圾收集两大类，这两类也常称为直接收集和间接收集

### 分代收集理论

**当前的垃圾收集器，大多遵循了分代收集的理论进行设计 即** 

**弱分代假说：指的是认为绝大多数对象都是朝生熄灭的**

**强分代假说：对象经过越多的垃圾收集器回收越难消亡**

这两个分代假说奠定了多款垃圾收集器的一致设计性原则

即新生代，老年代 新生代是指弱分代假说，在新生代里面的对象都是经过一次垃圾回收机器死亡的。

如果有部分对象没消亡，则将新生代的对象转移到老年代，又因为针对不同的区域产生了不同的垃圾收集算法。

这也是HotSpot分代式垃圾收集框架，鼓励尽量使用这个框架来开发垃圾收集器 但是大部分公司都没有遵守，因为框架会产生约束， 而且内存区域也不应该这么简单被区分。因为对象之间还会存在跨代引用

假设要进行新生代的垃圾收集(Minor GC),但是新生代的对象可能被老年代所引用，为了找出该区域的存货对象，就必须额外遍历整个老年代区域来确定结果的正确性，反过来也一样

为了解决这个问题，分代收集理论添加第三条经验法则：

**跨代引用假说：跨代引用相对于同代引用来说占极少数**

举个例子，如果新生代的对象被老年代所引用，那么老年代对象难以消亡，那么新生代经过一次垃圾收集器的收集会晋升到老年代，跨代引用自然就消失了

依据这条假设，我们就不需要额外遍历同代之外的对象了，只需要在新生代上建立一个全局的数据结构，这个结构把老年代分为若干小块，标识出老年代的哪一块内存会存在跨代引用。

Partial GC 部分收集 ：指目标不是完整的收集整个堆，其中又分为

新生代收集(Minor GC/Young GC) : 目标只是新生代的垃圾收集

老年代收集(Major GC) ： 目标只是老年代的垃圾收集 目前只有CMS收集器会有单独收集老年代的行为

整堆收集(FUll GC) : 整堆收集

### 标记-清除算法

需要进行回收的对象会进行标记，第二次会进行清除

缺点是清除后的内存分布不连续。空间碎片太多可能导致当前的程序在运行过程中需要分配较大的对象时无法找到足够的内存空间不得不提前触发一次垃圾收集

![image-20201210194757569](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210194757569.png)

### 标记-复制算法

为了解决清楚算法带来的问题，使用了标记算法， 将空间分为两块，一块存放所有的对象，一块为空 每当垃圾收集的时候，复制活的对象移动到空的块中。

![image-20201210194909268](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210194909268.png)

缺单是内存空间可用内存缩小为原来的一半

IBM公司对新生代做了更量化的诠释，百分之98都熬不过第一轮收集，因此不需要按照1：1来分配内存

Appel式回收优化了该算法，HotSpot虚拟机上的serial，ParNew等新生代收集器均采用了这种策略来设计新生代的布局

把新生代空间分为：一块较大的Eden空间和两块较小的Survivor空间

每次分配内存时只使用Eden和其中一块survivor。发生垃圾收集时，将存活 的对象复制到空的survivor空间，当空间不够大时，需要移动到其他内存块(一般都是老年代进行分配担保)

### 标记-整理算法

上面的复制算法很明显不适合老年代区域，因为老年代的大多数对象都是存活的。针对老年代又出现了标记-整体算法。标记也是和标记清除一样，标记后，但不是向可回收对象进行清理，而是将存活的对象都像内存空间一段移动，然后清理掉边界以内的内存

![image-20201210195922409](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201210195922409.png)

如果移动存活对象，尤其是老年代下的对象。那么将非常耗时，但是如果跟清除算法比较的话，总吞吐量还是高的。因为清除算法导致的空间碎片化问题只能依赖更为复杂的内存分配器和内存访问器来解决

更加耗时，因为内存是用户操作最频繁的区域。相当于垃圾收集而言标记清除算法只需要清除回收的对象是优于标记-整理的。 但是对于总用户程序来说 标记-整体更好，因为解决复杂的空间碎片更耗时

## HotSpot的算法实现细节

### 根节点枚举

GC ROOTS固定可作为类的静态变量，常量等，但是即便如此但在查找过程中要做到高效也并非是一件容易的事情

**迄今为止，所有收集器在根节点枚举过程中都需要暂停整个用户程序。查找引用链的过程中可以达到并发。相当于在一个一致性的快照当中，这里的一致性指的是程序没有任何分析，相当于冻结到了某个点上，不会存在根节点集合的对象引用关系还在不断变化**。即使号称停顿时间可控或者几乎不会发生停顿的CM1丶G1丶ZGC等收集器，枚举根节点也需要停顿

目前主流虚拟机都是准确式垃圾收集(即在类加载时直接保存GC ROOTS的位置信息)，

所以不需要一个不漏的检查上下文检查对象引用，只需要有一个存放所有引用集合的地方，HotSpot解决方案是使用OOP数据结构(相当于记录下类型信息)来达到这个目的，一旦类加载完成，虚拟机就会把对象内什么偏移量是什么类型的数据计算出来，即时编译过程中，也会在特点的位置记录下哪些位置是引用。

### 安全点

前面已经提到，在特点的位置记录了哪些位置是引用，这些位置被称为安全点。安全点一般选取的地方是是否具有让程序长时间执行的特征，长时间执行最明显的特征就是指令复用，一般在循环跳转，方法调用，异常跳转等地方设置安全点

对于安全点，一个需要考虑的问题，就是在垃圾收集发生时，如何让所有线程都跑到最近的安全点，然后停顿下来。有两种方案 抢占式中断和主动式中断。

抢占式中断：抢占式中断是指当需要发生垃圾收集的时候，直接暂停所有线程，暂停了以后如果有某个线程不在安全点上恢复该线程 过一会再中断该线程

主动式中断：相当于先设置一个标记位，每个线程主动去轮询这个标志位，当需要垃圾收集器的时候，线程执行到标记位上自己中断掉。线程如何自己中断呢？

![image-20201211203036122](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201211203036122.png)

轮询指令如图。如果需要暂停用户线程时，虚拟机把上面的0x160100的内存页设置为不可读。当线程执行到此处产生一个异常信号，预先注册的异常处理器会挂起当前线程实现等待

### 安全区域

上面都是基于线程活动的状态，但是如果线程状态是SLEEP或者BLOCKED呢？

这个时候线程无法走到标记位挂起自己，那么就需要安全区域了

安全区域是指，在某一段代码片段内，引用关系不会发生变化，因此在这个区域任意地方执行垃圾收集都是安全的

当线程要离开安全区域时，必须检查是否已经完成了根节点枚举

没有则一直等待

### 记忆集和卡表

在解决跨代引用的时候，说明了用记忆集的数据结构来判断哪一块非收集区域被引用

![image-20201211203846610](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201211203846610.png)

有以下的记录精度：因为要记录哪个区域或者对象被引用

字长精度：每个记录精确到一个机器字长，该字包含跨代指针。

对象精度：每个记录精确到一个对象，该对象里有字段包含跨代指针

卡精度：每个记录精确到一个区域，该区域包含了跨代指针

其中卡精度被应用广泛，也被称为卡表的方式实现记忆集

![image-20201211204207989](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201211204207989.png)

![image-20201211204215834](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201211204215834.png)

是一个Byte数组，没个元素都代表这对应的内存区域中一块特定大小的内存块，被称为卡页 一般都是2的N次幂 HotSpot使用的是2的9

数组每个元素对应一个块，如果哪个块存在跨代引用，则把该元素值修改为1

### 写屏障

我们上面已经说了要暂停线程，但是我们要用什么样的手段来暂停线程，维护卡表，修改卡表的值呢

**就要用到写屏障(注意 和解决并发问题的内存屏障不同)**

卡表元素何时变脏是确定的-当有其他分代区域对象引用了本区域对象 那么就需要变脏来让GC扫描引用本区域对象的内存块

写屏障是指在赋值的前后形成一个环绕通知和AOP切面一样，在写前和写后进行赋值，一般都是用的是写后赋值，当发生引用赋值时，则修改卡表元素的值，因为经过即使编译后的代码已经称为了机器指令流了

![image-20201211205039421](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201211205039421.png)

除了写屏障的开销外，卡表还会造成伪共享问题。假设缓存行通常是64字节，卡表一个字节，那么64个卡表元素对应的卡页大小为64*512字节 等于32KB 

那么就处在同时写一个缓存行，会发生伪共享问题，因为缓存一致性协议MESI不允许同时对缓存行里同一个内存数据进行更改

![image-20201211205453615](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201211205453615.png)

### 并发的可达性分析

在根节点枚举下，GC ROOTS相比于整个JAVA堆对象相比来说还是占用少数，但是需要寻找整个引用链就非常复杂了

假设三种不同的颜色：

白色：未被垃圾分析的时候对象假设是白色，经过垃圾收集分析结束后后如果还是白色，那么代表这个对象要被回收，即不可达

黑色：代表这个对象已经被访问过，并且这个对象的所有引用已经扫描过，它是安全存货的

灰色：代表被分析过，但是至少还有一个引用没被扫描

但是并发情况下会出现问题，一种是把原本要消亡的对象标记为存活，一种是把存活的对象标记为死亡 因为用户程序可能会修改引用关系

![image-20201211210120666](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201211210120666.png)

![image-20201211210124156](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201211210124156.png)

当下面两个条件同时满足时，会产生对象消失的问题

赋值器插入了一条或多条从黑色对象到白色对象的新引用(简而言之就是被垃圾扫描标记过安全的对象引用了一条白色的问题 是有问题的)

赋值器删除了一条或多条从灰色对象到白色对象的直接或者间接引用

因此解决并发的问题，只需要满足一个条件即可

增量更新和原始快照；

增量更新：是指当插入了一条或者多条黑色对象到白色对象时，虚拟机记录插入位置，并且再将该黑色引用为空再重新扫描一次，也就是说黑色对象一旦插入了白色对象，那么久会被重新扫描

原始快照：是指当删除了一条或者多条灰色对象

到白色对象时，虚拟机记录删除位置，并将灰色对象为根重新扫描一次。

CMS是基于增量更新来做并发标记的，G1则是按照原始快照

## 经典垃圾收集器

jdk7 Update4之后，JDK11正式发布之前，OracleJDK中的hotSpot虚拟机所包含的全部可用的垃圾收集器，使用经典二字为了与几款目前仍处于实验状态的收集器分割开来

![image-20201216180139394](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201216180139394.png)

如果存在连线，则代表它们可以搭配使用，因为有的是适用于老年代而有的是适用于新生代

所以需要搭配起来

### Serial收集器

这里的单线程指的并不仅仅是说明它只会适用一个处理器或者一个线程去进行垃圾回收，更重要的说明在垃圾回收过程中，必须暂停其他所有工作线程，直到它收集结束 即 Stop The World；

采用了标记复制算法，用作于新生代。

![image-20201216180635794](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201216180635794.png)

看起来好像很差劲，但是目前仍然是客服端模式下的主流垃圾收集器，有这由于其他收集器的地方，简单而高效，是所有收集器额外消耗内存最小的

### ParNew收集器

和Serial收集器很相似，最大的区别就是支持并行垃圾收集，但是还是要停止其他所有工作线程

![image-20201216181019379](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201216181019379.png)

比如说收集算法，Stop The World，对象分配规则，回收策略等都和serial相等

该收集器是运行在服务端模式下的收集器，尤其是jdk7之前，需要和CMS收集器进行配合

除了Serial收集器，只有他能和CMS收集器进行配合

但是由于收集器技术的不断改进，出现更优秀的G1收集器登场

官方直接取消-XX：+UserParNewGC参数，意味这ParNew和CMS从此只能搭配适用，可以理解为ParNew合并入CMS，该收集器可以说是第一个退出历史舞台的垃圾收集器

该收集器在单线程环境下不会有比serial收集器更好的效果

在谈论垃圾收集器的上下文语境中

并发可以理解为：用户程序和垃圾收集程序可以说是都在运行，但是垃圾收集程序占用了系统资源，会导致吞吐量降低

并行可以理解为：多个线程同时在执行垃圾收集

### Parallel Scavenge

该收集器也是新生代收集器，也是基于标记复制算法，也支持并行垃圾收集，那么它与ParNew收集器有什么区别呢？

该收集器与其他收集器不同，CMS等收集器的关注点是尽可能的缩短垃圾收集时用户程序的暂停时间，而该收集器则关注的是程序的吞吐量

吞吐量 =  用户程序/用户程序+垃圾收集时间

![image-20201216182341672](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201216182341672.png)

如果虚拟机完成某个任务需要100分钟，运行用户代码时间花了99分钟，垃圾收集1分钟

那么吞吐量就是百分之99 停顿时间越短，吞吐量越高，越适合需要与用户交互或者要保证服务响应时间的程序

Parallel Scavenge 该收集器提供了两个参数用来控制程序的吞吐量

分别是控制最大垃圾收集器停顿时间 和 直接设置吞吐量大小的两个参数

-XX : MaxGCPauseMilis参数 最大GC暂停毫秒数 

-XX : GCTimeRatio 参数  设置吞吐量大小

MaxGCPauseMilis参数是一个允许大于0的毫秒数，但是不能太小，因为它是以牺牲吞吐量和新生代空间为代价的，系统把新生代调小，收集300MB肯定会比500MB收集的快，那么导致发生垃圾收集频繁，甚至发生Full GC 

原本10秒停顿一次，停顿时间100毫秒，现在5秒停顿一次，停顿时间70毫秒，停顿时间确实减少了

但是总吞吐量降低了

-XX：GGTimeRatio的参数值是一个大于0小于100的整数，默认值为99 也就是默认允许百分之1的垃圾收集时间

调小该值相当于增加垃圾收集时间

由于与吞吐量关系密切，该收集器也经常被称为吞吐量优先收集器，该收集器还有一个参数

-UseAdaptiveSizePolicy， 这是一个开关参数，开了就不需要再给新生代设置eden,survivor的比例

，虚拟机会根据当前系统的运行情况来自适应的设置，动态调整这些参数以提供最合适的吞吐量或者最大的吞吐量

### Serial Old收集器

jdk5之前该收集器经常用于和Parallel Scavenge配合适用，是适合于客户端的老年代收集器。基于标记整理算法 ,另外一种就是CMS收集器发生失败的后备方案

![image-20201216192341340](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201216192341340.png)

### Parllel Old收集器

这款收集器是jdk6提供的，在此之前 Parllel Scavenge是处于很尴尬的地步。因为老年代除了Serial Old收集器没有其他选择，其他表现良好的老年代收集器如CMS，则无法与之配合，由于老年代的拖累效果，导致不能吞吐量最大化的效果，直到该收集器出现以后，吞吐量收集器终于有了搭配组合.

![image-20201216192900779](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201216192900779.png)

### CMS收集器

该收集器是以停顿时间最短为目标的老年代收集器,也是基于标记清除算法,CMS(Cocurrent Mark Sweet) ,运作过程更复杂一点,整个过程分为四个步骤

初始标记 : 会很快,只是标记一下能直接关联到的对象 ,停止用户程序

并发标记 : 可以和用户程序并发执行,但是吞吐量会降低 该过程是遍历整个对象图 寻找需要垃圾回收的对象进行标记 遍历和GC ROOTS的直接关联对象遍历

重新标记 : 可能出现并发问题,CMS采用的增量更新,如果有则重新标记,这个过程会比初始标记慢一些

![image-20201216193814072](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201216193814072.png)

并发清理 : 清除删除掉标记阶段判断死亡的对象,由于是清除 所以不需要移动对象,也可以和用户程序并发执行

该收集器的三个缺点:

CMS对处理器资源特别敏感,CMS默认启动的回收线程是(处理器核心数+3)/4,也就是说如果处理器核心数在四个以上,并发回收垃圾时只占用不超过25%的处理器运算资源,但是如果处理器负载很高,还要给垃圾收集器分配资源,会导致用户程序大幅度降低,虚拟机提供了一种增量式并发收集器,让其和用户程序交替执行,但是已经被废弃,效果不好

无法处理浮动垃圾,因为程序在运行过程中还是会产生垃圾,这部分垃圾对象是出现在标记后,所以CMS无法对它们进行垃圾收集,只能下一次再进行处理 那么问题来了,如果垃圾回收的速度赶不上分配对象内存空间的速度,会导致并发失败,可能会导致FULL GC的 STOP THE WORLD ,但是虚拟机的后备预案是适用Serial old收集器来重新进行老年代的垃圾收集

适用标记清除算法,该算法不会整理内存空间,存在许多的碎片空间,将会给分配大对象产生很大麻烦,往往老年代还有很多空间,但就是找不到足够大连续的空间来为大对象分配,而不得不触发一次Full GC的情况

### Garbage First收集器

G1收集器是垃圾收集器技术历史上的里程碑,它开创了收集器里面的面向局部收集和基于Region的内存布局形式

没有所谓的老年代新生代,所有的空间都是基于Region的 哪里有垃圾就收集哪里

可以对堆任何部分进行回收,哪块内存存放的垃圾最多,回收利益最大是它的衡量标准

G1是面向服务端的,jdk9发布之日,G1宣告取代Parllel Scavenge/Parllel Old组合,成为服务端默认的垃圾收集器 HotSpot虚拟机提出了统一垃圾收集器接口,将内存的回收与实现分离开,为CMS退出历史舞台铺垫

每一个Region都可以根据需要扮演新生代eden,Survivor空间或者老年代空间,收集者都能对扮演不同角色的Region采用不同的策略

Region还有一类特殊的Humongous区域,专门用来存储大对象,只要超过Region容量一半就认为是大对象

G1收集器之所以能建立可预测的停顿时间模型,因为维护了一个优先级列表,让G1收集器去追踪各个Region区域的价值大小,根据用户设置允许的收集停顿时间,去优先回收价值大的区域

Region存在跨Region引用应该如何? 前面的收集器都是单项引用,**但是在G1本质是一个哈希表,双向引用,key是别的region区域起始地址,value是卡页的索引号 双向的卡表结构 卡表是 我指向谁**

则这个还记录了谁指向我,实现起来非常复杂,并且由于这种结构还需要更大的内存空间

CMS收集器是适用增量更新保证并发的不干扰的正确运行,G1则是适用原始快照的算法实现的

G1为每一个Region区域设计了两个TAMS的指针,用来把一部分空间划分出来用于分配新对象的产生,与CMS类似,也有可能出现并发失败导致FULL GC的情况

G1的停顿预测模型是衰减平均值为理论基础的,平均值代表整体平均状态,而衰减平均值更代表最近的平均状态

推荐设置停顿时间为一两百毫秒,如果小了会导致每次选出来的回收集只是很小的一部分,最后导致分配对象没空间,导致FULL GC

从整体看基于标记-整理算法,从局部区域看又基于标记-复制算法

G1收集器运作过程可以分为4个步骤

初始标记 : 只是标记能直接关联到GC ROOTS的对象 很快,需要停顿线程,很短

并发标记 : 遍历整个对象图,进行标记,出现并发问题还要重新处理STAB记录下有引用变动的地方

最终标记 : 对用户程序做另外一个暂停,用于处理STAB记录

筛选回收 : **移动对象,情况旧的Region区域,把数据移动到新的区域,暂停用户程序**

![image-20201216202846067](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201216202846067.png)

G1相比于CMS不是压倒性的优势,因为为了垃圾收集产生的内存占用和程序运行时的额外负载都要高于CMS

因为G1的卡表更复杂,更占用内存

为了实现原始快照还需要适用写前屏障来记录并发时的指针记录情况

CMS目前在小内存引用上的表现由于G1

## 低延迟垃圾收集器

衡量垃圾收集器的三个重要指标: 内存占用，吞吐量，延迟。三者共同构成了一个不可能三角，想要同时拥有三个方面都完美的收集器几乎是不可能的，一款优秀的垃圾收集器最多可以同时达成其中的两项

在内存占用方面，随便计算机硬件性能的增长可以容忍，但是内存大了扫描整个堆的垃圾就会更慢，会导致延迟停顿过大，我们的目标是延迟的快慢不和堆大小有关系

![image-20201218171501565](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218171501565.png)

可以看到G1和CMS都需要暂停用户程序的，CMS和G1分别采用增量更新和原始快照技术，实现了标记阶段的并发

我们可以看到最后两款收集器Shenadoah,ZGC几乎是全程支持和用户一起并发运行的，这两款收集器目前还处于实验阶段的垃圾收集器，被官方命名为低延迟垃圾收集器

### Shenadoah收集器

不属于所谓的“官方”收集器，不是Oracle公司旗下的，拒绝在jdk12中支持该垃圾收集器，只有OpenJdk有。由RedHat公司发展的新型收集器项目，这个项目的目标是在任意堆大小下，都可以把垃圾停顿时间限制在十毫秒以内。

该款收集器更像是G1的继承者，许多处地方都和G1相似。比如Region区域，同样有这存放大对象的Humongous Region，默认回收策略也是回收优先价值最高的Region，但是相比于G1至少有三个不同点。

没有适用G1的记忆集数据结构，不存在区域互相引用记录，并且支持标记-整体阶段支持和用户程序进行并发执行，而G1是不允许的，只是可以并行。是没有适用分代理论，Shenadoah收集器屏蔽了在G1上大量耗费内存的记忆集，改名为连接矩阵，降低了处理跨代指针的记忆集维护消耗，也降低了伪共享的发生概率，连接矩阵可以看作是一张二维表格，如果Region N有对象指向M，则在二维表格上的N行M列打上一个标记，在回收的时候通过这张表格就可以看到哪些区域产生了跨代引用

![image-20201218174735626](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218174735626.png)

收集器工作过程大致可以分为九个阶段：

初始标记：与G1一样，标记GC ROOTS直接关联到的对象，也是要暂停用户程序的，但是很短并且停顿时间与堆的大小无关

并发标记：标记整个对象图，可以和用户进行并发执行，标出全部可达对象。

最终标记：与G1一样，处理剩余的STAB扫描，并在这个阶段统计出回收价值最高的Region,将这些Region组成一组回收集

并发清理：这个阶段用户清理有的Region没有一个存活的对象，对该内存空间直接进行清理，并且后续可以被对象分配空间

并发回收：并发回收阶段是和G1区别最大的阶段，因为这块是可以和用户并发执行的，Shenadoah这个阶段要做的事情就是将存活对象复制到新的Region区域，如果暂停用户程序是非常简单的，但是如果要和用户程序并发执行，如果用户此时访问移动的对象内存地址，可能会访问到旧地址，那么就会出错，解决办法采用的是反转指针，后面会提到

初始引用更新：复制对象后，需要将堆中所有指向旧对象的指针修正到新地址处，这个操作称为引用更新，但是初始引用更新这个阶段其实并没有什么处理，只是确保线程都执行完了各自的移动对象任务，建立一个线程集合点

并发引用更新：真正开始引用更新的，这个阶段是和用户并发执行的，不需要沿着对象图来搜索，只需要按照内存物理地址的顺序，线性的搜索出引用类型，把旧值改为新值

最种引用更新：解决了堆中的对象引用更新，还要解决GC ROOTS的引用更新，做得最后一次停顿，也很短，只与GC ROOTS数量有关

并发清理:经过并发清理和引用更新以后，整个回收集已无存活对象，最后再调用一次并发清理来回收这些区域的空间，供对象分配内存

![image-20201218180434029](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218180434029.png)

其实也就是三个最重要的并发阶段(并发标记，并发回收，并发引用更新)

Brooks Pointer Brooks是一个人的名字，发表了反转指针，大概就是在对象移动的旧内存地址设置一个设置保护陷阱，让操作系统进入内中断，然后适用预设好的异常处理器跳转到新的内存地址。

但是代价很大，会频繁使操作系统从用户态转换为核心态，不能频繁使用

![image-20201218180830454](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218180830454.png)

Brooks提出的新方案不需要用内存保护陷阱，而是在对象布局结构的最前面加上一个统一的引用字段，如果对象没有发生变化，则指向自己

和句柄有点像，先访问句柄，句柄池存放真正对象的引用地址

只需要更新转发指针让其指向新内存地址

但是会一次额外的内存访问开销

并且也不是线程安全的 必须所有同步来解决

因为

1）收集器线程复制了新的对象副本

2）用户线程更新对象的某个字段

3）收集器线程更新转发指针的引用值为新地址

如果2发生在3之前，就会发生问题，写入到旧的地址，所以必须使用CAS保证线程安全

如果站在访问对象这四个字上分量是很重的。对于访问，写入，对象的比较，对象的哈希值，用对象加锁等都属于访问对象的范畴，该收集器都要同时使用写丶读屏障进行拦截

在读取都加入了额外的转发处理，特别是读屏障，消耗很大，因为对对象的读是非常频繁的。

所以JDK13中计划只拦截对象中引用对象的类型，基于引用访问屏障

![image-20201218181633306](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218181633306.png)

其实也没有达到10ms目标，并且总停顿时间还是非常大的，吞吐量也不会很优越

### ZGC收集器

ZGC和Shenadoah的目标是高度相似的，都是目标任意堆大小垃圾停顿时间10ms内，ZGC实现了

和Azul的PGC和C4是高度相似的

ZGC的内存布局与Shenadoah一样，但是ZGC的区域具有动态性

X64硬件平台下，ZGC具体可以有大丶中丶小三类容量

小型Region：固定大小为2MB，放置小于256kb的对象

中型Region：固定大小为32MB，放置大于256kb小于4MB的对象

大型Region：容量不固定，可以动态变化，用作防止4MB以上的大对象，并且容量的变化必须是2的整数倍

接下来ZGC的核心问题-并发整理算法的实现 使用的染色指针技术

三色标记，有的标记直接标记到对象头上(Serial收集器)，有的标记记录在与对象相互独立的数据结构上(G1,Shenadoah)使用了相当于堆大小1/64空间的BitMap的结构来标记信息，则ZGC则是直接把记录信息存放到指针上，与其说是遍历对象图还不如说是遍历引用图来标记引用

为什么指针本身可以存储信息呢？因为64位系统下理论可以支持2的64次方也就是16EB字节，但是实际上根本到不了这么多。Linux下的64位指针的高18位不能用来寻址，但是还有46还有64TB的内存。所以ZGC的染色指针技术顶上了46位指针，把高四位提取出来存储四个标记信息

还有2的42次方4TB内存空间

![image-20201218185107892](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218185107892.png)

也不支持压缩指针。也不用记录对象引用的变化情况

不过想要顺利使用到高4位，操作系统能不能支持？因为无论什么数据都会被转换成机器指令流交付给处理器执行，处理器才不会管什么是标记，什么是地址等

采用内存多重映射将多个不同的虚拟内存地址映射到同一物理内存上，把染色指针中的标志位看作是地址的分段符，只要将这些不同的地址段都映射到同一物理内存空间，经过多重映射后，就可以使用染色指针正常进行寻址了![image-20201218191234403](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218191234403.png)

![image-20201218191715912](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218191715912.png)

ZGC的运行过程大致可以分为4个阶段，都是可以和用户程序并发的

仅有中间两个阶段会有小停顿

并发标记：与G1，SH一样也是遍历对象图，但是不是对对象做标记，而是在指针上，标记阶段会更新染色指针上的marked0,marked1标志位

并发预备重分配：这个阶段要根据特定的条件查询出来要清理的Region区域，和G1还是有区别的。G1是目的是找到收益优先的回收，相反ZGC每次回收都会扫描所有的Region,用更大的扫描成本换取省去G1集中记忆集的维护成本，因此ZGC的重分配只是将存活的对象移动到新的Region，里面的Region会被释放，因此标记过程是针对全堆的

并发重分配：**这个过程是要把重分配集中的存活对象复制到Region中，有染色指针的支持，可以直接判断对象的标志位是不是处于重分配集中，并且为重分配集里面每个Region区域维护一个转发表，记录旧对象到新对象的转向关系，如果用户线程此时并发的访问了位于重分配集中的对象，这次访问会被预制的内存读屏障所截获，然后根据Region区域上的转发表直接转发到新复制的对象上，并同时修正该引用的值，使其指向新对象，ZGC将这种行为称为指针的自愈，这样的好处是只有一次慢，对比Shenadoah的转发指针，那是每次对象访问都必须付出固定的开销，简单的来说每次都慢，还有一个好处就是由于染色指针的存在，一旦重新分配集中的某个Region的存活对象都复制完毕，这个Region就可以立即释放用于新对象的分配，但是转发表还要留着**

并发重映射：重映射就是指修正整个堆中指向重分配集中旧对象的所有引用，但是并发重映射并不是迫切的完成任务，因为上面提到过指针是拥有自愈功能的，因此ZGC把该次工作合并到了下一次垃圾收集循环中的并发标记

ZGC没有用到任何的写屏障，CMS中的卡表也不需要，但是也有缺点

就是能承受对象的分配速率也不是太高，如果对整个垃圾收集要发生10分钟，但是在这10分钟有大量新对象生成，会造成堆满

ZGC还支持NUMA 非统一内存访问架构，由于摩尔定律的逐渐失效，频率收到限制转而向多核发展，每个核心都有自己的内存管理，如果要访问其他处理器核心管理的内存，必须通过通道来完成，会非常慢

因此ZGC收集器会优先尝试请求线程在当前处理器的本地内存上分配对象，以保证高效内存访问

![image-20201218203802104](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218203802104.png)

都快和以吞吐量著称的Paraller收集器差不多

![image-20201218203823330](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218203823330.png)

停顿时间更是直接碾压

![image-20201218203837758](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201218203837758.png)

## 选择合适的垃圾收集器

### Epsilion收集器

该收集器更像是自动内存管理子系统，干的不仅仅是垃圾收集

JDK11中以可以不断进行垃圾收集为卖点的垃圾收集器 听起来让人感到十分违反逻辑

一个垃圾收集器除了进行垃圾回收以外，还要负责堆的管理与布局，对象的分配，与解释器的协作等等

随着微服务的潮流，传统java面对短时间，小规模的服务形式感到不适，为了应对，jdk逐渐加入了提前编译，面向应用的类数据共享等支持，Epsilion收集器也有这类似的目标，如果你的应用只要运行数分钟甚至数秒，只要JAVA虚拟机能正确分配内存，在堆耗尽之前就可以退出，使用Epsilion收集器是非常好的选择

## 虚拟机以及垃圾收集器日志

jdk9之前，并没有业界标准，统一的日志处理框架，9之后才统一

使用-Xlog参数，HotSpot所有的功能日志都整合到该参数

![image-20201222180209066](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222180209066.png)

命令行最关键的部分是selector,它由tag标签和level组成

比如 -Xlog gc

日志级别从高到低，默认是Info

![image-20201222180340140](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222180340140.png)

可以使用修饰器来要求每行都加上额外的附加内容

![image-20201222180429839](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222180429839.png)

如果不指定，默认是level,tags,uptime

User 是指用户代码在用户态下的执行时间

Sys 是指用户代码在核心态下的执行时间

Real 是指用户代码从开始到结束的时间

查看GC基本信息 9之前需要使用 -XX : PrintGC  9之后 -Xlog : gc GCTest(类名)

![image-20201222180632909](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222180632909.png)

查看GC详细信息 同上 -XX : PringGCDetails  9之后 -Xlog ：gc*  使用通配符*将所有细节打印出来

查看GC前后的堆，方法区容量变化 之前是 -XX：PrintHeapAtGC 之后是 -Xlog: gc+heap = debug

![image-20201222180838669](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222180838669.png)

查看GC过程中用户程序并发时间以及停顿时间

9之前 -XX : PrintGCApplicationConcurrentTime以及 -XX:PrintGCApplicationStoppedTime,JDK9之后

-Xlog : safepoint : 

![image-20201222181413909](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222181413909.png)

自动设置堆大小空间各分代区域大小，收集目标等内容 从Parallel收集器开始支持

-XX : PrintAdaptive-SizePolicy   -Xlog : gc+heap* = trace:

![image-20201222181808200](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222181808200.png)

查看熬过收集后剩余对象的年龄分布信息 -XX : PrintTenuring-Distribution(分配)  -Xlog : gc + age = trace:

![image-20201222182312033](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222182312033.png)

![image-20201222182353464](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222182353464.png)

![image-20201222182403426](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222182403426.png)

![](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222182412921.png)

### 垃圾收集器参数总结

UseSerialGC 使用串行SerialGC/Old  打开此开关后，程序默认使用serial+serial old收集器

UseParNewGC 使用ParNew+Serial Old jdk9之后不再支持

UseConcurrentSweep 使用CMS+ParNew+SerialOld  serialOld用作备选方案 出现并发失败后的后备收集器使用 

UseParallelGC 9之前运行在Server端下的默认模式 Parallel Scvange + Serial GC 

UseParallelOldGC 打开此开关后 使用吞吐量组合

SurvivorRatio 新生代中Eden区域与Survivor区域的比值  默认为8 

PretenureSizeThreshold  大于这个参数的对象直接在老年代分配

MaxTenuringThreshold 晋升到老年代的对象年龄 每次+1 到该值晋升

![image-20201222183808923](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222183808923.png)

![image-20201222183828469](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222183828469.png)

### 内存分配与回收策略

自动化管理，最根本解决了两个问题，自动给对象分配内存以及自动回收分配给对象的内存

#### 对象优先在Eden区分配

大多数情况下，对象在新生代Eden区分配，当Eden区没有足够空间时，发生一次Minor GC

#### 大对象直接在老年代分配

java中的大对象要避免，因为在分配空间时，容易导致内存明明还有很多空间却提前触发垃圾收集

提供了-XX : PretenureSizeThreShold(门槛大小)参数 指定大于该数直接放入老年代

此参数只针对新生代的 ParNew和Serial有效 对吞吐量收集器无效 如果必须使用该参数调优 则使用CMS+ParNew

#### 长期存活的对象会进入老年代

每个对象都有一个年龄计数器，存储在对象头 当第一次没有被收集后，会从eden区域挪到survivor区域同时把年龄参数加1 以后每经过一次垃圾收集并且没被回收时该参数就会加1 直到达到ThreShold默认值15后 会移动到老年代 也可以使用-XX : MaxTenuringThreShold参数指定该值

#### 动态对象年龄判断

如果Survivor区域年龄相同的对象加起来大于Survivor区域的一半，年龄大于或者等于的对象则也会进入到老年代，不需要等到年龄计数器

#### 空间分配担保

在发生Minor GC时，虚拟机必须先检查老年代最大的连续空间是否大于新生代所有对象总空间，如果这个条件成立，那么进行新生代垃圾收集就是成立的

如果不大于，那么查看是否开启了-XX : HandlePromotionFailure参数是否允许担保失败，如果不允许则直接发生FULL GC 如果允许则去判断 老年代最大的连续空间是否大于历次(过去每次的)晋升到老年代对象的平均大小，如果大于则会垃圾收集Minor GC，否则进行FULL GC

是有风险的，万一这次晋升到老年代的对象非常大，那就很影响效率了

## 虚拟机性能监控丶故障处理工具

### 基础故障处理工具

#### jps 虚拟机进程状况工具

该工具可以列出正在运行的虚拟机进程，并显示虚拟机执行主类main函数所在的类

以及这些进程的本地虚拟机唯一ID(LVMID) 

![image-20201222190912024](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222190912024.png)

![image-20201222190939591](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222190939591.png)

jps还可以根据RMI协议查询  参数hostid是为RMI注册的主机名字

RMI : 远程方法调用 主要是面向对象的 需要知道调用的对象和对象中的方法

RPC : 远程过程调用 主要是调用具体的子程序

![image-20201222191238216](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222191238216.png)

-l 输出主类的全类名，如果线程执行的是JAR包则输出JAR路径

#### jstat 虚拟机统计信息监视工具

可以用来观察本地或者远程虚拟机进程中的类加载，内存，垃圾收集，即时编译等运行数据

无GUI页面

![image-20201222192301744](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222192301744.png)

![image-20201222192316458](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222192316458.png)

jstat -gc 2764 250 20

解释为 打印虚拟机进行2764 中的 垃圾收集信息 每隔250毫秒执行一次 总执行20次

![image-20201222192449115](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222192449115.png)

选项option中代表用户希望查询的虚拟机信息，类加载，垃圾收集，运行期编译状况	

![image-20201222192635988](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222192635988.png)

-gc 监视java堆状况，分区信息，垃圾收集器时间合计等信息

-gcutil 监视内容与-gc相同 但是输出主要关注已使用空间占总空间的占比

![image-20201222200755691](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222200755691.png)

![image-20201222200817782](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222200817782.png)

GCT GC TIME GC总耗时 

#### jinfo Java配置信息工具

jinfo作用是实时查看和调整虚拟机参数，jps -v 查看虚拟机启动时显示指定的参数列表， 如果想查看未被显示指定的系统参数 则需要jinfo -flag选项来进行查询了

![image-20201222201100823](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222201100823.png)

#### jmap Java内存映像工具

jmap命令用于生成堆转储快照，如果不使用该命令则需要暴力 如 -XX ： +HeapDumpOnOutOfMemoryError，可以让虚拟机在内存溢出后生成堆转储快照

jmap不仅可以生成快照，还可以查看finalize队列，java堆和方法区的详细信息

![image-20201222201329159](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222201329159.png)

![image-20201222201334275](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222201334275.png)

![image-20201222201340268](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222201340268.png)

jmap -dump:formate = b,file =..

#### jhat 虚拟机堆转储快照分析工具

与jmap结合使用，但是没啥用，一般人很少用 主要用于分析快照，但是快照又占用资源一般都选择其他机器，那么其他机器使用GUI工具更为强大

![image-20201222201851870](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222201851870.png)	![image-20201222201901000](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222201901000.png)

#### jstack java堆栈跟踪工具

生成线程快照，线程快照就是每一条线程正在执行的方法堆栈的集合，一般用来定位出现长时间停顿的线程原因，如死锁，死循环等

![image-20201222202032858](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222202032858.png)

![image-20201222202039242](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222202039242.png)

一般都是 jps-l,jstack -l

![image-20201222202059074](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222202059074.png)

### 可视化故障处理工具(GUI页面)

#### JHSDB 基于服务性代理的调试工具

整合了上面的所有基础工具所能提供的专项功能

![image-20201222202739186](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222202739186.png)

![image-20201222202810074](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222202810074.png)

#### JConsole Java监视与管理控制台

是一款基于JMX的可视化监视，管理工具

JMX: 是一个为应用程序植入管理功能的框架，就是通过将监控和管理设计到的各个方面的问题和解决方法放到一起，统一设计

具有的功能

内存监控 相当于可视化的jstat命令

线程监控 相当于可视化的jstack命令

### VisualVM 多合故障处理工具

比起上面的还提供性能分析

JMC 可持续化在线的监控工具

![image-20201222203835455](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222203835455.png)

![image-20201222203854843](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222203854843.png)

JFR 飞行记录 存放各种信息

### HotSpot虚拟机插件及工具

#### HSDIS 编译器的反汇编插件

可以将字节码反编译为汇编语言进行分析

![image-20201222204019915](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222204019915.png)

![image-20201222204026501](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222204026501.png)

## 调优案例分析与实战

### 案例分析

#### 大内存硬件上的程序部署策略

一般有两种方式

通过一个单独的JAVA虚拟机来管理大量的JAVA堆内存

同时使用若干个JAVA虚拟机，建立逻辑集群来利用硬件资源

#### 集群间的同步导致的内存溢出

集群中每当接受新请求时，均会更新最后一次操作时间，把这个时间同步到所有节点，使得一个用户不能在一段时间内在多台机器上登录，导致节点之间网络交互频繁，网络情况不能满足传输速度时，大量数据在内存中堆积，造成内存溢出

#### 堆外内存导致的溢出错误

B/S电子考试系统选用了逆向AJAX技术，类似于Websocket后台主动向前台推数据，是NIO，本地

![image-20201222205740806](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222205740806.png)

DirectByteBuffer这个对象是堆内引用堆外的直接内存 大小受本地内存限制

![image-20201222205857752](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222205857752.png)

#### 服务器虚拟机进程崩溃

![image-20201222210002946](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222210002946.png)

![image-20201222210050477](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222210050477.png)

#### 不恰当数据结构导致内存占用过大

一台PRC机器，ParNew+CMS组合，平时对外服务的Minor GC时间大约在30毫秒

但业务每10分钟加载一个80MB的数据到内存进行数据分析 并且使用Map<Long,Long>Entry数据结构，会造成500毫秒的停顿

![image-20201222210244043](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222210244043.png)

Long占用8个字节，2个Long 16字节 又把long类型包装为Long  就分别具有8字节的对象头mark World 

8字节的kclass指针，再加上8字节存储数据的long值

![image-20201222210421817](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222210421817.png)

改用数据结构

#### 由安全点导致的长时间停顿

![image-20201222210525255](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222210525255.png)

安全点的选取一般都是选择循环，方法调用等地方

但是由于HotSpot虚拟机为了避免安全点过多导致的负担，会对循环进行优化，认为循环次数少时，执行时间也不会台长，所以使用int类型或者范围更小的数据类型作为索引值默认是不会放置安全点的

所以将循环里面的变量int改为long 问题消失	

### 调优

#### 调整内存设置控制垃圾收集频率

![image-20201222211117198](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222211117198.png)

每一次FULL GC都会随着老年代空间扩容，所以这种情况直接设置老年代初始大小，最大之和最小值，使其固定

#### 选择收集器降低延迟

更换符合场景需求的垃圾收集器

![image-20201222211250800](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201222211250800.png)

#### 降低编译时间和类加载时间

升级jdk版本

# 虚拟机执行子系统

## 类文件结构

### 无关性的基石

JAVA虚拟机不与JAVA程序语言绑定，它只与Class文件有关，Class文件包含了指令集，符号表等等

JAVA虚拟机规范中要求Class文件必须应用许多强执行的语法和结构化约束，但具备图灵完备的字节码格式(也就是指一套针对数据操作规则可以完成图灵机模型的任何功能，就是指图灵完备)，保证了任何一门语言都可以放到虚拟机上执行。JAVA虚拟机不仅仅为JAVA提供服务，在设计的时候也考虑到了其他语言编译为字节码文件在虚拟机上运行

![image-20201228121901849](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228121901849.png)

也就是如上图 JAVA虚拟机只要符合的字节码文件，不关心上层用了什么语言和什么编译器，只要符合JAVA虚拟机规范定义的字节码规则，那么就可以放到虚拟机上运行。

### Class类文件的结构

任何一个Class文件都对应这一个类或者一个接口，但是接口和类并不一定都在文件里，因为类或接口可以动态生成，直接放到类加载器里。

Class文件是一组以8个字节为基础单位的二进制流，当遇到需要占用8个字节以上的数据，那么将会分割数据，让高位在前的方式分割成若干个8个字节进行存储

无符号数属于基本的数据类型，以u1,u2,u3,u4来分别代表一个字节，二个字节，三个字节，四个字节

表是由多种无符号数或者其他表作为数据项构成的复合数据类型

![image-20201228174015330](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228174015330.png)

无论是无符号数还是表，当要描述同一类型但数量不定的多个数据时，经过会使用前置的容量计数器，加若干个连续的数据项的形式，这时候称该数据为某一类型的集合

### 魔数与Class文件的版本

每个Class文件的头四个字节被称为魔数，并且该值是固定的,0XCAFEBABE,紧接这是存储Class文件的版本号，第4和第5个字节是次版本号，6和7是主版本号，JAVA的版本号从45开始

![image-20201228174352472](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228174352472.png)

在jdk6编译输出的Class文件为基础来进行讲解 因为JVM虚拟机的Class文件基本没什么改动

![image-20201228174434953](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228174434953.png)

### 常量池

紧接这主次版本号之后的是常量池入口，常量池可以比作Class文件的资源仓库，通常也是占用Class文件空间最大的数据项目之一

常量池容量(0x00000008)为十六进制数0x0016，十进制为22，从1开始1-21，有21个常量数

常量池中主要存放字面量和符号引用，字面量一般有文本字符串，被final声明的常量等，描述常量的一种常量被称为字面量，而符号引用一般包括：

​                    **类和接口全限定类名**

​                    **被模块导出或者开放的包**

​                    **字段的名称和描述符**

​                     **方法句柄和方法类型**

​                    **动态调用点和动态常量**

JAVA代码在进行javac编译的时候，没有像C和C++那样有连接这一步骤(就是各类数据)，而是在虚拟机加载文件的时候进行动态连接，也就是说这些符号引用必须等到虚拟机转换期间才能得到真正的内存入口地址，当虚拟机做类加载的时候，会从常量池中获得对应的符号引用，再在类创建的时候或者运行时候解析，翻译到具体的内存地址等

虽然C和C++使用的是逻辑地址，但是也是地址，而java则是没有地址，都是一些符号引用来标识

常量池中每一项都是一个表

![image-20201228175740842](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228175740842.png)

![image-20201228175748040](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228175748040.png)

![image-20201228180005025](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228180005025.png)

类或者接口的字段引用，是一个表，表中有两个类型的数据，数量都是1 一个是tag也就是标签，表明是类或者接口的常量 值为0x07,一个字节，name_index是常量池的索引值，它指向常量池中一个CONSTANT_Utf8_info类型常量，也就是说，它存放的是该类型常量的地址。而后面的u2则是两个字节，0x0002，则代表从常量池中找第二个常量，0x01(D) ,查表可知第一个值确实是该常量，utf-8编码的字符串，该表的结构为![image-20201228181047396](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228181047396.png)

tag为1代表是该常量，第二个类型u2则代表的是长度，两个字节0x001D,也就是长度为十进制的29个字节![image-20201228181616326](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228181616326.png)，名称为bytes的数据是使用utf8缩略编码，缩略编码和普通uft8编码的区别是在0-127的值范围，缩略编码占用一个字节![image-20201228181525571](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228181525571.png)

那么bytes的数量就是29

![image-20201228181745305](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228181745305.png)

被蓝色标记的，都是代表该值

![image-20201228182104022](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228182104022.png)

摘取自ASCII表上的可以看出0x6f对应的就是o值，以此类推，则代表了![image-20201228181616326](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228181616326.png)刚好长度为29个

我们已经分析了两个常量池中常量池的两个，还有许多

Oracle公司为我们提供了javap来分析字节码文件

![image-20201228182321722](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228182321722.png)

![image-20201228183315217](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228183315217.png)

###  访问标志

在常量池访问结束以后，紧接这两个字节代表访问标志(access_flags)，这个标志用于识别一些类或者接口的访问层次，包括这个Class类是类还是接口，是否是public,是否是abstract等

![image-20201228192236238](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228192236238.png)

![image-20201228192327235](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228192327235.png)

上面的类被public修饰，所以值应该为0x0020+0x0001 为21 可以看出图中蓝色部分确实是21

### 类索引，父类索引和接口索引集合

类索引(this_class)和父类索引(super_class)都是一个u2类型的数据，而接口索引则是一组u2类型的集合，因为父类只允许继承一个，而接口可以实现多个。

![image-20201228192745242](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228192745242.png)

先找第一个常量，找到Class_Info再找第二个常量，然后就定位到了类索引

![image-20201228192938418](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228192938418.png)

查找过程和上面的查找org那一块方法一样，只不过有javap更方便

### 字段表集合

上面我们已经说完了所有关于类的，如类的全限定名查找，访问标志，以及类索引，父类索引查找，下面开始说字段字段可以被很多修饰，如static,public,final,private,volatile等等，字段的数据类型

![image-20201228193244651](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228193244651.png)

![image-20201228193309703](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228193309703.png)

访问标志和类的访问标志很像，都是2个字节大小的，name_index也是指向CONSTANT_UTF-8的常量，

一个类的全限定名仅仅是将类全名的.改为/，为了多个全限定名之间不产生混乱，通常最后会加一个;代表结束，简单名称则是指方法名,如inc(),和m。字段和方法的描述符就要复杂一些，描述符是指方法参数的类型以及返回值的类型，或者是字段的数据类型

![image-20201228193706990](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228193706990.png)

对于数组，每一个维度使用一个前置的[字符，如java.lang.String[],被定义为[[Ljava/lang/String;，一个int[]被定义为[I。

以上是描述字段的

![image-20201228194416658](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228194416658.png)

根据上图，第一个2字节的是字段数量，第二个2字节为访问标志为private，第三个为指向第5个常量为m，描述符为I，那么可以推断出该字段为 private int m; 描述符之后跟随一个属性表集合，用于存储一些额外的信息，本字段中的字段m,属性值为0，但是如果将m改为private static final m = 123;

那么123的值就会存放在属性表中，那就可能存在一项名称为Constant Value的属性，字段表中不会列出从父类或者父接口中继承而来的字段，但是可能出现原本JAVA代码中不存在的字段，譬如为了在内部类保证可以对外部类的访问性，编译器就会自动增加指向外部类实例的字段

### 方法表集合

![image-20201228195403056](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228195403056.png)

和字段表的结构几乎一样，![image-20201228195416024](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228195416024.png)

到现在，那我们方法里面的代码去哪了呢？ 存放到什么位置了？ 方法的定义可以通过name_index,access_flag,descriptor_index描述清楚，但是方法的代码呢？方法的JAVA代码，经过javac编译器编译成字节码指令之后，存放在一个方法属性表为Code的属性里，属性表作为Class文件中最具有扩展性的一种数据项目。后面会讲到

![image-20201228195801836](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228195801836.png)

也有一个方法计数器，有两个方法，一个初始化方法和inc方法，第7个常量为<init>,描述符为第8个常量为()V,属性计数器的值为1，属性的name_index为第9个常量Code。()V的意思是无参数，返回值为void，实例构造器

同样的，如果父类中的方法子类没有重写，那么子类方法表集合中就不会出现该方法，同样的编译器会自动添加方法，如类构造器<clinit>，和实例构造器<init>;

### 属性表集合

前面讲过，Class文件，字段表，方法表都可以携带自己的属性表集合

![image-20201228201104748](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228201104748.png)

![image-20201228201116336](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228201116336.png)

属性表每一个属性，它的名称都要从常量池中引用CONSTANT_UTF8_info类型的常量来表示，而属性的结构则是完全自定义的，只需要一个u4的长度属性去说明属性值所占用的位数即可

![image-20201228201659126](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228201659126.png)

最大可表示4个字节的长度，然后如果有100个，那么就会有1字节的100个info信息

#### Code属性

JAVA程序方法体的代码经过JAVAC编译以后，最终会变为字节码指令存放到Code属性内

如果是抽象方法就没有该属性，因为没有方法体

![image-20201228202558594](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228202558594.png)

上面是Code属性表的结构，attribute_name_index指向CONSTANT_Utf8_info型常量的索引，属性名称索引和属性值一共为6个字节

max_stacks代表了操作数栈深度的最大值，max_locals代表了局部变量表所需要的空间，max_locals的单位是变量槽，变量槽是虚拟机为局部变量分配内存所使用的最小单位，int,char,boolean和ReturnAddress等长度不超过32位的数据类型，每个变量存放一个变量槽，而double,long占用两个变量槽，方法参数，异常处理程序的参数，方法体中的局部变量都需要使用到变量槽，max_locals

操作数栈和局部变量表直接决定一个方法所占用内存的大小，使用变量槽重用。当代码执行到超过一个局部变量的作用域时，会重复使用该局部变量存放的变量槽，根据同时生存的最大局部变量数量和类型计算出max_locals的大小

code和code_length则是方法体编译后的字节码，code_length为方法长度，虽然是u4，但是java虚拟机规范允许不能超过65535也就是2的16次方

以TestClass.class文件为例子，init方法的Code属性，操作数栈和本地变量表的都为0x0001，字节码长度为0x000005,以此读取5个字节

![image-20201228204121794](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228204121794.png)

![image-20201228204728116](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228204728116.png)

操作数栈主要是为了方法体里面的数据运算

![image-20201228205924719](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228205924719.png)

读入2A，查表得0x2A的指令为aload_0,这个指令的含义是将第0个变量槽的reference类型的本地变量推送到操作数栈顶

读入B7，查表得0xB7指令为invokespecial,这个指令的含义是将栈顶的reference数据所指向的对象作为参数的接受者，调用该对象的实例化方法,private方法，或者它的父类方法

这个方法还有一个u2类型的参数说明具体调用哪一个方法，指向常量池CONSTANT_METHOD_info常量，此方法的符号引用

读入000A，查表得0x000A指令为invokespecial,这个指令是一个符号引用，查常量池得对应的常量为实例构造器init方法的符号的引用

读入0xB1,该指令是一个return指令，并且返回值为void

![image-20201228211140411](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228211140411.png)

![image-20201228211208018](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228211208018.png)

上图无论是实例化方法还是inc方法理论上来讲都是没有参数的，那么为什么两个参数都为1呢？其实还有一个this关键字，在任何实例方法中，都可以通过this关键字访问到此方法所属的对象，这个机制对java很重要，是将JAVA关键词的访问转变为对一个普通方法参数的访问，在调用该方法的时候传入参数就行了，因此在局部变量表中也会预留出第一个变量槽来存放对象实例的引用，所以实例方法参数值从1开始计算，这个处理只对实例方法有效

![image-20201228212532924](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228212532924.png)

上图是如果存在异常表，就是该结构

这些字段的含义是，如果start_pc到end_pc这些行之间(不包含end_pc行)出现了catch_type或者其子类的异常，则转到handler_pc继续处理，当catch_type的值为0是，表示任意情况都需要转到handler_pc处理

异常表实际上是java代码的一部分，尽管字节码最初为处理异常而设计的跳转指令，但是JAVA虚拟机规范中明确要求java语言的编译器应该使用异常表而不是跳转指令

下面例子

![image-20201228212931602](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228212931602.png)

![image-20201228212937436](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201228212937436.png)

上图是代码编译

0-4行 将整数1赋值给变量x，并且复制副本写入到本地变量槽，然后将本地变量槽中的值读到操作数栈顶，当返回值使用

如果没有异常则会执行到5-9行 将x的值赋值为3，然后写入变量槽，返回操作数栈顶的1元素 也就是return value

如果出现异常 PC寄存器则会跳转到第10行

#### Expections属性

上面的异常表是结构信息，以及catch语句块，存放的是代码出现的异常信息，而该属性列举出了方法中可能抛出的受检异常，也就是方法描述符后面的throw关键字后面列举的异常

![image-20201229174118782](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229174118782.png)

number_of_exceptions表示方法可能抛出多少种异常数，而具体的异常是exception_index_table，该表指向常量池中的Class_info类型的常量的索引，代表了该受检异常的类型

#### LineNumberTable属性

该属性用于描述字节码行号和源代码行号之间的关系，缺少该属性调试程序无法根据行号来调试断点

![image-20201229174445625](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229174445625.png)

line_number_table是一个类型为line_number_info的集合，该表包含start_pc和line_number两个u2类型的数据项，前者代表字节码行号，后者代表java源码行号

#### LocalVariableTable和LocalVariableTypeTable属性

localVariableTable属性是用于描述局部变量表和java源码中定义变量之间的关系，如果关闭该属性，最大的影响就是当其他人引用这个方法的时候，所有的参数名称都会消息，会使用arg0,arg1来代替，而且在调试期间无法根据参数名称从上下文中获得参数值

![image-20201229175145095](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229175145095.png)

![image-20201229175201890](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229175201890.png)

start_pc代表了这个局部变量属性的生命周期开始的字节码偏移量，length代表了作用范围覆盖的长度

两者结合组成了局部变量在字节码之中的作用范围，

name_index和descriptor_index分别指向常量池中的utf8类型的索引，即方法名称和方法描述符，index是这个局部变量在栈帧的局部变量表中变量槽的位置，当数据类型是64位的时候，占用的槽是index和index+1

jdk5之后引入泛型后，加入了一个localVariableTypeTable，基本与上面一样，区别只是将描述符改为特征签名，如果没泛型一致，如果有泛型则不一致

比如

```java
public static void String(List<Integer> list)
```

特征签名是指integer，因为会泛型擦除，所以使用该属性来存放特征签名

#### SourceFile和SourcedDeubugExtension属性

SourceFile属性用于记录生成这个Class文件的源码文件名称

![image-20201229180656660](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229180656660.png)

![image-20201229180715786](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229180715786.png)

都差不多，下面提供额外的信息	

#### ConstantValue

ConstantValue属性的作用是通知虚拟机自动为静态变量赋值，必须是被static定义的变量，比如static int i = 123,必须是类变量，并且赋值不是赋值为123，对于非静态变量是在init构造器中完成的，而对于类变量，有两种方式一个是类构造器clinit,一个就是ConstantValue属性，Oracle实现的javac编译器，如果同时使用final和static来修饰一个变量(准确的说是常量)，并且数据类型是基本类型或者String类型，那么就使用ConstantValue属性来进行初始化，反之，则通过类构造器

![image-20201229181553765](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229181553765.png)

因为只是对应常量池中的索引，只能表示基本类型或者String

#### InnerClasses属性

该属性用于记录内部类和宿主类的关系，如果一个类中定义了内部类，那么会有该属性

![image-20201229181955925](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229181955925.png)

number_of_classes代表需要记录多少个内部类信息，每一个内部类由inner_classes_info所描述

![image-20201229182109012](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229182109012.png)

前两个指向utf8常量池的索引，分别代表了内部类和外部类的符号引用

代表内部类的名称，如果是匿名的，这项值为0，最后一个是访问标志

![image-20201229182449549](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229182449549.png)

#### Deprecated以及Synthetic属性

如果被修饰了Deprecated注解，则会出现该属性

Synthetic是由编译器自动生成的，用于越权访问的(如private)

![image-20201229182613953](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229182613953.png)

#### StackMapTable属性

该表位于Code属性的属性表中，相当于一个类型检查器，省去了分析数据流类型推导的过程，大幅度提升性能，在编译阶段将一系列的验证类型直接记录在Class文件中，通过这些检查验证代替了类型推导，该验证器在jdk6首次提供

![image-20201229184844402](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229184844402.png)

包含0到多个栈帧映射

#### Signature属性

该属性用于记录泛型签名信息，可以出现在类，方法，字段结构的属性表中，之所以要记录，是因为会泛型擦除![image-20201229185149007](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229185149007.png)

因为泛型擦除的原因会导致无法运行期获得泛型信息，该属性出现后，就可以获取泛型类型

![image-20201229185247595](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229185247595.png)

该值对应一个常量池的有效索引，并且该索引出的项必须是utf8_info结构

####  BootstrapMethods属性

如果常量池中出现过InvokeDynamic_info类型的常量，那么就需要有该类型

![image-20201229185659247](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229185659247.png)

![image-20201229185705501](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229185705501.png)

bootstraps_methods数组中的每个成员必须包含以下三个内容

ref，必须是一个有效索引，必须是一个Constant_MethodHandle_info结构

#### MethodParameters属性

用于存储方法参数名称的，localVariableTable是Code属性的子属性，但是有的没有方法体，比如抽象类和接口方法，那么导致调用该方法的时候没有方法名称，所以jdk8加入了该属性

![image-20201229190720131](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229190720131.png)

![image-20201229190726648](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229190726648.png)

![image-20201229190734001](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229190734001.png)

#### 运行时注解属性

RuntimeVisibleAnnotations是一个变长属性，它记录了类，字段或者方法的声明上运行时可见注解，当我们使用反射API来获取类字段或者方法上的注解时，返回值就是通过这个属性获取到的

![image-20201229200253585](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229200253585.png)

![image-20201229200305104](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229200305104.png)

每一个annotations代表了一个运行时可见注解，type_index指向utf8常量的索引值，该常量以字段描述符的形式表示一个注解，element_value_pairs代表了注解的参数和值，num代表了计数器，如果有多个参数和值则会在计数器上说明

### 模块化相关属性

![image-20201229200022667](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229200022667.png)

## 字节码指令简介

JAVA虚拟机的指令是由一个字节长度的，代表这某种特定操作含义的数字称为操作码，以及跟随其后的代表该操作的参数(操作数)，由于JAVA面向操作数栈，所以大多数情况下指令只有操作码，操作数都存放在操作数栈中![image-20201229201023346](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229201023346.png)

### 字节码与数据类型

iload代表从局部变量表中加载int类型的数据到操作数栈中，而fload指令则是加载float,但是JAVA代码可能它们都是同一句话，但是Class文件中的字节码则不一样

![image-20201229201452483](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229201452483.png)

![image-20201229201533818](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229201533818.png)

以上各种指令对应的数据类型，大多数指令都没有支持byte,short,char,甚至没有任何指令支持boolean类型，编译器会在编译器或者运行期将这些类型带符号扩展为int类型

#### 加载和存储指令

加载和存储指令用于将数据在操作数栈和局部变量表之间来回传输，这类指令包括

将一个局部变量表加载到操作数栈：iload,iload_<n>,aload,aload<n>

将一个操作数栈写入到局部变量表：istore,istore_<n>,astore,astore_<n>

将一个常量加载到操作数栈 bipush,sipush,ldc,ldc_w,aconst_null,

扩充局部变量表的访问索引的指令: wide

![image-20201229202222130](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201229202222130.png)

iload_0和操作数为0操作码为iload指令语义完全一样，都是将局部变量表操作数为0的数据加载到操作数栈

####  运算指令

加法指令 ：iadd,ladd,fadd,dadd

减法指令：isub,lsub,fsub,dsub

,乘法指令 ：imul,lmul,fmd,dml

求余指令：irem  取反指令：ineg 位移指令：ishl,ishr,iushr,lshl,lshr,lushr

按位或指令：ior,lor，按位异或指令：ixor,lxor

局部变量自增指令：iinc，比较指令：dcmpg,dcmpl

#### 类型转换指令

有数值类型宽化类型转换，小范围到大范围安全

还有大范围到小范围 不安全  i2b,i2c,i2s,l2i等等

#### 对象创建与访问指令

创建类实例的指令：new，JAVA中创建数组和类实例使用了不同的字节码指令

创建数组的指令：newarray，anewarray，multianewarray

访问类指令(static字段，类变量) 和实例变量(非final的)的指令：getfield,putfield,getstatic,putstatic

将一个操作数栈的值存放到数组元素中的指令：bastore,castore,sastore,iastore,fastore,dastore,aastore

取数组长度的指令:length

检查类实例类型的指令：instanceof,checkcast

#### 操作数管理指令

将操作数栈顶的一个或两个元素出栈 pop,pop2

复制栈顶一个或两个数值并将值重新压入栈顶：dup，dup2，dup_x1，dup2_x1

将栈最顶端的两个数值交换 swap

#### 控制转移指令

控制转移指令可以让JAVA虚拟机有条件或者无条件的从指定位置指令的下一条指令继续执行，可以认为控制转移指令就是在有条件或者无条件的修改PC寄存器的值

条件分支：ifeq,iflt,ifle,ifnull

复合条件分支：tableswitch,lookupswitch

无条件分支：goto，goto_w,ret

#### 方法调用和返回指令

方法调用(分派，执行过程)，5条指令用于方法调用

invokevirtual 用于调用对象的实例方法

invokeinterface 用户调用接口方法，它会在运行时搜索一个实习了该接口的对象找出适合的方法进行调用

invokespecial指令：用于调用一些需要特殊处理的实例方法，比如初始化方法，私有方法和父类方法

invokestatic指令：用于调用类静态方法

invokedynamic：用于在运行时动态解析出调用点限定符引用的方法，并执行该方法，用户可以改变

方法返回指令：ireturn  方法的返回指令是根据返回值的类型区分的 

#### 异常处理指令

java中显式抛出异常的操作都由athrow指令来实现，而异常捕获，catch等 都是通过异常表

#### 同步指令

monitorenter,monitorexit，如果是方法级同步 则是通过ACC_SYNCHRONIZED访问标志来得知是否被声明为一个同步方法，调用指令首先会检查方法的该标志位，如果设置了，执行线程就要求先成功持有管程，然后才能执行方法

## 虚拟机类加载机制

JAVA虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验，解析和初始化，最终形成可以被虚拟机直接使用的JAVA类型，这个过程被称为虚拟机的类加载机制，与那些编译时要进行连接的语言不同，在JAVA语言类加载是在运行过程中进行的，因为编译要连接的话要直接解析，形成逻辑地址，装入内存等等操作

### 类加载的时机

加载-》验证-》校验-》解析-》初始化-》使用-》卸载	

![image-20201231202459747](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201231202459747.png)

其中的验证，校验，解析统称为连接

并且图中加载，严重，准备是确定的，但是解析却不一定，它在某些情况下可以在初始化阶段之后再开始，注意只是按部就班的开始，不是完成和进行，只保证顺序开始

JAVA虚拟机严格规定了有且只有六种情况必须对类进行初始化(加载，验证，准备自然要在其之前开始)

**1）遇到new,getstatic,putstatic,invokestatic，遇见该四条指令的时候必须对类进行初始化，如调用一个类的静态方法时**

**2）使用java.lang.reflect的包对类型进行反射调用的时候，如果类型还没有进行初始化，则需要先触发初始化3）当初始化类的时候，如果父类还没有初始化，则要先触发对父类的初始化**

**4）当虚拟机启动的，用户要指定一个主类 main方法的那个类，要对该类进行初始化**

**5)  当一个接口中定义了java8中的default方法，如果这个接口的实现类发生了初始化，则该接口要在其前初始化**

**因为普通情况下，如果接口中没有default方法，接口是不需要初始化的，只有调用接口中的方法才需要**

![image-20201231203418207](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201231203418207.png)

会输出 SuperClass init

对于静态字段，只有直接定义这个字段的类才会初始化。

![image-20201231203521704](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201231203521704.png)

不会对类进行初始化，会生成另外一个类，org.fenixsoft.classloading.SuperClass的一维数组。数组中的方法都实现在这个类里

![image-20201231203700242](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201231203700242.png)

以上也不会触发，因为static加final被放置了ConstantValue属性里面，而不是被类构造器加载

### 类加载的过程

#### 加载

加载阶段是类加载阶段中的一个阶段，JAVA虚拟机应该完成三件事情

**1）通过一个类的全限定名来获取此类的二进制流**

**2）将这个字节流所代表的静态存储结构转变为方法区的运行时结构**

**3）在内存中生成一个代表这个类的Class对象，作为方法区对这个类的各种数据的访问入口**

JAVA虚拟机规范对这三点要求不是特别具体

比如说从ZIP获取类的二进制流，从网络上，运行时计算生成，动态代理 等等

对于数组类而言，情况不同，因为数组不是通过类加载器加载的，而是直接在内存中动态生成的，但是数组类与类加载还有密切的关系，因为数组的类型，比如int类型，引用类型的数组 ，这些类型需要类加载器来完成，如果是引用类型，数组类将被标识在加载该数组类型的类加载器的类名称空间上，一个类型必须与一个类加载器一起确定唯一性

如果是基本数据类型，那么虚拟机会把数组类标记为与引导类加载器关联

并且数组类型与对数组的可访问性一直，如果不是引用类型，则它的数组类的可访问性默认为public

#### 验证

验证是连接阶段的第一步，确保Class文件的字节流中包含的信息符合JAVA虚拟机规范的全部约束

验证阶段是非常重要，这个阶段是否严谨直接决定了JAVA虚拟机能否承受恶意代码的攻击

从整体上看 验证会分为四个步骤

文件格式验证，元数据验证，字节码验证和符号引用验证

##### 文件格式验证

是否以魔数0XCAFEBABE开头，主次版本号是否在当前JAVA虚拟机范围之内，

常量池中的常量是否有不被支持的常量类型(检查常量Tag标志)

utf8_info信息是否有不符合UTF8的编码的数据等等等

实际上第一阶段的验证远不止这些

##### 元数据验证

这个类是否有父类

这个类是否继承了不允许被继承的类

如果这个类不是抽象类，是否实现了接口或者抽象类中的抽象方法

类中的字段是否与父类的字段产生矛盾

等等

##### 字节码验证

第三个阶段是整个验证过程中最复杂的一个阶段，主要目的是通过数据流分析和控制流分析，确定程序语义是合法的，第二个阶段对元数据信息校验完毕以后，这个阶段就要开始对方法体进行校验

如

保证任意时刻操作数栈的数据类型与指令代码序列都能配置工作

保证任何跳转指令都不会跳转到方法体以外的字节码指令上

保证方法体中的类型转换是有效的

由于数据流分析和控制流分析的复杂性，为了避免过多的时间耗费在字节码验证阶段，JDK6之后的javac编译器和java虚拟机进行了一次联合优化，让验证阶码尽可能多的移动到编译阶段，**具体做法是给Code属性的属性表中添加了一个StackMapTable的新属性**，这项属性描述了方法体中所有的基本块开始时本地变量表和操作栈应有的状态，在字节验证期间，就不需要根据程序推导这些状态的合法性，只需要检查StackMapTable属性中的记录是否合法，从类型推导转变为类型检查，理论上该属性也存在错误或者被篡改的可能

##### 符号引用验证

最后一个阶段发生在虚拟机将符号引用转换为直接引用的时候，这个动作将在连接的第三个阶段完成，解析。

**符号引用通过字符串描述的全限定名是否能找到对应的类**

**在指定类中是否存在符合方法的字段描述符以及简单名称以及简单名称所描述的方法和字段**

**符号引用中的类，字段，方法的可访问性是否可被当前类访问**

#### 准备

准备阶段是正式为类中定义的变量分配内存并且设置类变量初始值的阶段，

假如public static int value = 123,那么赋值的时候是设置为0，而不是123

把value赋值为123的putstatic指令是在程序被编译后，存放在类的构造器方法之中，所以把value赋值为123的动作要到类的初始化阶段才会执行

![image-20201231212546233](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20201231212546233.png)

上面提到的是通常情况，如果是特殊情况，比如

public static final int value = 123;

编译时JAVAC会被value生成ConstantValue属性，在准备阶段虚拟机就会将value值根据ConstantValue的设置将value赋值为123

#### 解析

解析阶段是将Java虚拟机常量池内的符号引用替换为直接引用的过程

直接引用：直接引用是可以直接指向目标的指针，相对偏移量或者是可以间接定位到目标的句柄

符号引用：符号引用以一组符号来描述所引用的目标

JAVA虚拟机规范并未规定解析阶段发生的具体时间

要求在17个用于操作符号的字节码指令之前 如getfield,getstatic,instanceof,new,invokestatic等等

![image-20210111175649756](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111175649756.png)

先对它们所使用的符号引用进行解析。所以虚拟机实现可以根据需要来自行判断是要在类被加载时就解析还是等到一个符号引用被使用才去解析它

对一个符号引用进行多次解析是很常见的事情，除invokedynamic指令之外，虚拟机实现可以对第一次解析的结构进行缓存，譬如在运行中直接引用常量池中的记录，避免重复解析

但是对于invokedynamic指令而言上面的规则就不成立了，因为是为了支持动态语言的![image-20210111180156302](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111180156302.png)

主要针对于类或者接口，字段，类方法，接口方法等进行解析

CONSTANT_CLASS_info,CONSTANT_Fieldref_info

##### 类或接口的解析

假设当前代码所处于的类为D，如果要把一个从未解析过的符号引用N解析为一个类或者接口C是一般需要以下三步骤：

1）如果C不是一个数组类型，那么把符号引用N传递给D的类加载器去加载该类，在加载过程中，由于元数据验证，字节码验证的需要，又要去触发其他相关类的加载动作，例如加载这个类的接口或者父类。一旦这个加载过程出现异常，解析过程将宣告失败

2）如果C是一个数组类型，并且数组类型为对象，那么会按照1）的规则先去加载该类型，加载完之后，接着由虚拟机生成一个代表该数组的数组对象

3）如果上面两步无异常，那么C在虚拟机中是一个有效的类或者接口了，但是在解析完成之前还需要对其进行访问权限验证，确定D是否具备对C的访问权限

##### 字段解析

要解析一个未被解析过的字段，首先会对字段表中的class_index项中索引的CONSTANT_Class_info符号引用进行解析，也就是字段所属的类或者接口的符号引用

先去解析该字段所属于的类或者接口，解析完成后，把这个字段所属于的类或者接口用C表示，那么JAVA虚拟机规范要求对C进行后续搜索：

如果C已经包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束

否则，如果该类实现了接口，则会从下往上以此找符合的接口或者父接口，查找结束

否则，如果该类不是Object类型，则会同上，查找结束

否则，查找失败，抛出异常

解析字段完成后，会对其进行权限解析。

![image-20210111182208410](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111182208410.png)

但在实际情况中其实更严格，如果把Sub定义的属性去掉，那么将会报错，因为父类和接口同时定义了并且子类没定义

##### 方法解析

方法解析与字段解析步骤很相似，也是先解析该方法所属于的类或者接口用C表示，解析完成后，JAVA虚拟机规范要求按以下步骤进行后续的方法搜索：

由于Class文件格式中类的方法和接口的方法符号引用常量是分开的，如果发现C是个接口的话，直接抛出异常

否则如果在该类中找到了简单名称和描述符与目标相符的方法，则直接返回

否则在类的父类中递归查找是否有相符的

否则在类C的接口列表以及它们的父接口之中递归查找是否有相符的

否则抛出异常

查找过程中成功返回了符号引用，则对其进行权限验证

##### 接口方法验证

接口方法也是先解析出接口方法表的class_index，依然用C

如果发现是个类，则抛出异常

否则在接口C中查找符合的方法，查找结束

否则在接口C的父接口递归查找，由于JAVA接口允许多继承，如果C的不同父接口中有多个简单名称都与目标相匹配的方法，那么会查找一个并且返回

#### 初始化

类的初始化是类加载过程的最后一个步骤，

![image-20210111193831769](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111193831769.png)

clinit方法是由编译器自动收集类中的所有静态变量和静态代码块中的语句合并产生的

准备阶段变量已经赋过一次系统要求的初始零值，而在程序员通过程序编码定制的变量和其他资源会在初始化阶段完成

clinit与虚拟机视角的init方法不同，它不需要显式的调用父类构造器，java虚拟机会保证在子类的clinit方法执行前父类会先执行

![image-20210111194148537](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111194148537.png)

输出为2

<clinit>方法对于类或接口来说并不是必须的，如果一个类中没有静态代码块，也没有对变量的赋值操作，那么编译器可以不为这个类生成该方法

接口也是同理，但是与类的clinit方法不同的是，执行接口的clinit方法不需要先执行父接口的clinit方法，只有当父接口的变量或者接口被使用的时候才会进行执行

并且JAVA虚拟机必须保证一个类的clinit方法在多线程环境下被正确的加锁同步

![image-20210111194442962](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111194442962.png)

![image-20210111194449368](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111194449368.png)

同一个类加载器下，一个类型只会被初始化一次

### 类加载器

JAVA虚拟机在类加载阶段中的通过一个类的全限定名来获取描述该类的二进制字节流这个工作放在JAVA虚拟机外部去实现，以便让应用程序自己决定如何去获取所需的类，实现这个动作的代码被称为类加载器

#### 类与类加载器

**类加载器虽然只实现类的加载动作，但是在JAVA程序中作用远超类加载阶段，对于任意一个类都需要由类加载器和这个类本身确定唯一性，比较两个类是否相等，必须是在同一类加载器下才有意义，否则，即使这两个类是同一个CLASS文件，但是只要加载它们的类加载器不同，那么它们就不相等**

![image-20210111195025948](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111195025948.png)

![image-20210111195035613](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111195035613.png)

上面两个类虽然是一个类，但是由于一个是应用程序类加载器加载的，一个是由我们自定义的类加载器加载的，所以返回false

#### 双亲委派模型

其实就是首先会向上查找，直到最顶层的类加载器找不到该类才会向下。

**最上层就是启动类加载器**，加载<JAVA/HOME>下的lib目录，并且必须是符合格式的 如rt.jar,tools.jar，否则哪怕是在lib目录下也不会被加载

![image-20210111195535401](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111195535401.png)

如果需要把自定义的类加载器委派给引用类加载器时，只要返回null即可

**扩展类加载器**：该类去加载JAVAHOME/lib/ext目录中的，用于扩展类库的，允许用户将通用型的类库放在该目录下扩展JAVASE的功能

**应用程序类加载器**：我们自己创建的类一般都是由应用程序类加载器来加载的，主要加载classpath类路径下的类，该加载器是getSystemClassLoader的返回值，有时候也被称为系统类加载器

![image-20210111195933579](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111195933579.png)

java9之前的应用都是通过该三种类加载器互相配合完成的，还可以自定义类加载器进行扩展，或者通过类加载器实现类的隔离

该图展示的层次关系被称为双亲委派原则。双亲委派原则要求除了启动类加载器以外，其他的类加载器都必须要有父类，不过一般都不是用继承的方式，而是使用组合的方式，类里面有个parent属性，会优先使用该属性的类加载器去加载![image-20210111200344816](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111200344816.png)

#### 破坏双亲委派模型

在JDK1.2之前ClassLoad就已经存在，但是双亲委派模型在jdk1.2才被引入，为了兼容已存在代码无法以新技术避免loadClass()方法，只能在jdk1.2之中添加一个protected方法findClass()，并引导用户程序去重写该方法，如果父类加载失败，则会调用自己的findClass来完成类的加载

第二次被破败是由于这个模型自身缺陷导致的，该模型解决了基础类型的一致性问题，越基础的类越由上层类加载器加载，但是如果我们有基础类型需要调用回用户代码呢？ 该怎么解决？

比如说JDBC需要上层调用下层提供的接口，比如说JNDI，JNDI的目的就是管理资源，那么就需要调用由其他厂商实现并且部署在应用程序的classPath下的JNDIf服务提供者接口，那么需要怎么办?

然后提出了一个线程上下文类加载器，可以通过线程Thread类方法就是设置，如果线程创建未设置，那么会从父线程中继承一个，默认就是应用程序类加载器。相当于是从上下向去访问了，破坏了双亲委派模型，会从上层加载器去加载下层的服务提供代码。并且JDK6提供了ServiceLoad类，以META-INF/services中的配置信息，加上责任链模式，才算给SPI加载提供了一个相对合理的解决方案

第三次破坏是用户对程序动态性的追求所导致的，这里所说的动态是指热替换(Hot Sweep)，热部署等等，使得程序修改代码不用重启即可继续使用

**我们来看看OSGI是如何通过类加载器来实现热部署的**

OSGI实现模块化热部署的关键是它自定义类加载器机制的实现，每一个程序模块(OSGI称为Bundle)都有一个自己的类加载器，当需要更换一个Bundler时，就把Bundler连同类加载器一起换掉，每当收到类加载请求 的时候，**OSGI按照下面的顺序进行类搜索**：

将以java.*开头的类，委派给父类加载器加载

否则，将委派列表名单内的类，委派给父类加载器加载

否则，将Import列表中的类，委派给Export这个类的Bundle的类加载器加载

否则，查找当前的ClassPath,使用当前Bundle的类加载器去加载

否则，查找类是否在Fragment Bundle中，如果在，则去加载

否则，查找Dynamic Import列表中的Bundle,委派给对应的

否则，查找失败

### JAVA模块化系统

jdk9引入的模块化系统，是JAVA技术的一次重要升级，并且使用G1为默认垃圾收集器

jd9的模块不像之前只是简单的容器，还定义了以下内容：

依赖其他模块的列表，导出包的列表，即其他模块可以使用的列表，开放的包列表，使用的服务列表，提供服务的是实现列表

![image-20210111211930687](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210111211930687.png)

主要是可配置的封装隔离机制，还解决了类路径上跨JAR文件的public类型的可访问性问题，意味这程序中不再可以随意访问到public类型，并且使得运行期异常可以提前到启动时，直接显式声明好哪个模块依赖哪个模块

#### 模块的兼容性

JDK9提出了与类路径相对应的模拟路径的概念。简单来说，就是某个类库到底是模块还是传统的JAR包，取决于它存放在哪种路径上。

![image-20210112175904185](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112175904185.png)

#### 模块化下的类加载器

首先，扩展类加载器被平台类加载器所取代，是一个很成功的变动，既然整个JDK都基于模块化构建，那已经满足了天然可扩展的需求，自然也不需要lib/ext目录。类似的，也取消了JAVAHOME/jre目录，因为随时可以构建出模块化的JRE环境

![image-20210112180228079](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112180228079.png)

其次，平台类加载器和应用程序类加载器都不再派生于URLClassLoader，现在启动类加载器，平台，应用程序类加载器都继承于jdk.internal.loader.BuiltinClassLoader

![image-20210112180523343](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112180523343.png)

![image-20210112180535640](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112180535640.png)

启动类加载器9之后已经是和JAVA类库共同协作实现的类加载器，

但是为了保持与之前代码的兼容，仍然会选择返回Null来代替

比如Object.class.getClassLoader();

![image-20210112181314245](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112181314245.png)

9之后其实也算是一次破坏

因为平台类加载器在收到类加载器请求之前，首先会去判断该类能否归属到某一个系统模块中，如果可以找到该模块，就优先让负责那个模块的加载器来完成加载

在JAVA模块化系统明确说明了三个类加载器负责加载的模块，如下所示：

.启动类加载器所加载的模块

![image-20210112181542554](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112181542554.png)

![image-20210112181552123](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112181552123.png)

![image-20210112181557652](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112181557652.png)

## 虚拟机字节码执行引擎

**执行引擎是JAVA虚拟机核心的组成部分之一，虚拟机是一个相对于物理机的概念，两种机器都有执行代码的能力，区别在于物理机的执行引擎直接建立在处理器，缓存，指令集，操作系统上的，而虚拟机的执行引擎则是由软件自行实现的.**

**JAVA虚拟机规范中指定了JAVA虚拟机字节码执行引擎的概念模型，通常会有解释器(通过解释器执行)和编译执行(通过即时编译器产生本地代码执行)**

**虚拟机的执行引擎输入和输出都是一致的，输入的是二进制字节流，处理过程是字节码等效的执行过程，输出是执行结果**

### 运行时栈帧结构

Java虚拟机以方法作为最基本的执行单元，栈帧则是支持虚拟机进行方法调用和方法执行背后的数据结构，每一次进栈出栈都伴随这一个方法的进栈入栈，也是虚拟机运行时数据结构数据区的虚拟机栈中的栈元素，栈帧包含局部变量表，操作数栈，方法返回地址和动态链接等信息，在编译JAVA源码的时候，栈帧中需要多大的局部变量表和多大的深度就已经被确定了，不会受运行期的影响

一个线程中的方法调用链可能会很长，以JAVA的角度来看，同一时刻，同一线程里面有多个方法执行，但是对于执行引擎来说，只有位于栈顶的方法才是在运行的，被称为当前栈帧

#### 局部变量表

局部变量表以变量槽为最小单位，方法的Code属性max_locals确定了该槽的大小。没有明确的指定大小，只是说各种数据类型都可以以32位或者更小的内存空间来存储 

![image-20210112194850311](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112194850311.png)

引用类型的话存放的是内存地址。32位占用一个变量槽，64位如Long,Double占用两个变量槽，

当一个方法被调用的时候，Java虚拟机会使用局部变量表来完成实参到形参的传递。如果执行的是实例方法，那么参数列表的第一个参数永远都是this对象的地址，即代表当前方法所属类的实例。

为了节省栈帧耗用的空间，局部变量表中的的变量槽式可以复用的

![image-20210112195746130](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112195746130.png)

上面没有回收也说的过去，因为还处于作用域内

![image-20210112195838929](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112195838929.png)

![image-20210112195847000](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112195847000.png)

但是可以发现，离开了作用域依然没有被回收 为什么呢？

![image-20210112195917801](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112195917801.png)

placeholder能否被回收的根本原因是，局部变量表中是否还存在placeholder数组对象的引用，代码虽然离开了作用域的范围，但是之后没有对该变量槽进行覆盖，也就是没有任何读写操作，所以GC roots还依然保持这对它的关联，后面多定义了一个变量，则将该变量槽的值覆盖，所以才成功被回收

![image-20210112200457051](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112200457051.png)

并且局部变量必须赋初始值

#### 操作数栈

是一个后入先出的栈，操作数栈也在方法Code属性中的max_statcks数据项之中，32位数据类型所占的栈容量为1，64位数据类型栈容量为2

一般来说操作数栈是用于给方法提供运算的，都会伴随这出栈入栈的过程计算数值，并且很严格 当执行iadd指令的时候，要求操作数栈的两个数必须都是int，概念模型中，两个栈帧作为不同方法的虚拟机栈元素，是完全独立的。但是大多数虚拟机会优化，让其两个部分重叠，比如一个栈帧的局部变量表中的共享区域和一个栈帧中的操作数栈共享区域重叠，可以在方法调用的时候直接共用数据，不需要传递数据了

![image-20210112201543527](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112201543527.png)

#### 动态连接



每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，大白话就是栈帧里面存放了的地址，持有这个引用的目的是为了动态连接，JAVA有一部分是需要在运行期将符号引用转换位直接引用。所以相当于栈帧中保存了一个引用，相当于C的指针，指向该方法在运行时常量池的位置。

#### 方法返回地址

有两种方式，一种是正常调用完成，第二种是异常调用完成，不会提供返回值

### 方法调用

方法调用不等于方法代码被执行，只是决定方法的版本(即调用哪一个方法)。暂时还未涉及方法内部。

方法调用是最频繁的操作之一

#### 解析

在类加载的解析阶段，会有一部分符号引用转换为直接引用，能够成立的前提是运行期不可变，并且已经确定要调用的是哪个方法，这类方法的调用被称为解析

JAVA语言中符合编译器可知，运行期不可变这个要求的方法，主要有静态方法，私有方法，被final修饰的方法，前者和类关联，后者在外部不可以被访问

![image-20210112202837757](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112202837757.png)

只要是一和二都可以在编译器被已知，都可以在解析阶段确定唯一的版本，静态方法，构造器方法，私有方法，父类方法，被final修饰的方法 都会在编译器转换为直接引用，被称非虚方法，其他被称为虚方法

![image-20210112203052673](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112203052673.png)

![image-20210112203104897](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112203104897.png)

解析调用一定是一个静态的过程，在编译器就完全确定，而另外一种调用复杂很多，分派调用。会被分为静态分派，动态分配，单分派，多分派，静态多分派，动态单分派，它可能是静态的也可能是动态的

#### 分派

本章节讲解的分派会揭示多态性的一些最基本的体现，如重载和重写是怎么实现的，我们关心的是虚拟机如何正确执行我们想要的方法

##### 静态分派

重载就是基于静态分派的，静态分派

![image-20210112203929088](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112203929088.png)

上面都会输出hello,guy

Human为静态类型，MAN或者Woman为动态类型。两者都可变，区别是静态类型只会在使用时发生变化，变量本身类型不会变，实际类型的变化只有在运行期才会被确定，编译器不知道对象类型

![image-20210112204949200](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112204949200.png)

![image-20210112205419536](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112205419536.png)

![image-20210112205425733](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112205425733.png)

![image-20210112205431727](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112205431727.png)

![image-20210112205439179](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112205439179.png)

##### 动态分派

了解了静态分派，动态分派与JAVA语言多态性的另外一个重要体现-重写

![image-20210112210052951](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112210052951.png)

上图输出man,woman,woman

![image-20210112210412687](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112210412687.png)

**关键在于invokevirtual指令，该指令大致步骤为：**

**找到操作数栈顶第一个元素所指向对象的实际类型 记作C**

**然后寻找C中方法符合描述符和简单方法名的，返回找到则方法这个方法的直接引用**

**否则，继续查找父类等**

**正是因为该指令第一步就是查找所指向的实际类型的引用，所以调用该指令并不是把符号引用解析到直接引用就结束了，**还会根据实际类型来匹配

![image-20210112211600277](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210112211600277.png)

实现多态的三个条件：有继承关系，子类重写了父类的方法，父类的引用指向子类的对象

上图，字段和多态无关 0，4，2  因为gay就是Father类型

##### 单分派和多分派

方法的接收者和方法的参数统称为方法宗量，根据分派基于多少种宗量，可以分为单分派和多分派

单分派是根据一个宗量对目标方法进行选择，多分派则是根据多个

方法的接收者可以理解为实际执行该方法的对象

![image-20210113181317713](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113181317713.png)

结果为

father choose 360

son choose qq

**首先我们分析静态分派，也就是编译阶段编译器的选择过程，静态类型可以选择是Father,Son，参数类型可以是360或者QQ 因此静态分派属于多分派类型**

**动态分派，在运行期执行invokevirtual执行的时候，由于编译器已经选择了Son.hardChoice(QQ)的参数必须是QQ类型，所以唯一可以影响虚拟机执行因素的只有该方法的接收者实际类型是Son,Father，只有一种宗量进行选择，因此动态分派属于单分派类型**

##### 虚拟机动态分派的实现

动态分派是执行非常频繁的动作，因为动态分派的版本选择要根据接收者的实际类型来确定方法的版本号，由于性能的原因，是不会去频繁的搜索类型元数据的

元数据：关于数据的数据

一般的优化方法有为类型在方法区建立一个虚方法表，记录每个方法，搜索的时候直接根据该表去查询

![image-20210113182522042](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113182522042.png)

每一个类型都记录了一个方法表，该方法表里面的方法地址指向别人，有的指向自己

比如Father方法写了一个hardChoice方法，那么方法表中就记录了该方法的入口地址，并且如果子类没有重写父类的方法，那么该方法的入口地址和父类中的方法入口地址是一致的，都指向父类的方法入口地址

并且为了程序实现方便，具有相同的签名的方法，在父类，子类中方法表应当具有一样的索引编号，当需要类型变换的时候，可以直接定位

虚方法表一般在类加载的连接阶段进行初始化，准备了类的变量初始值以后，虚拟机会把该类的虚方法表也一同初始化完毕

### 动态类型语言支持

前面提到过，方法的符号引用在编译时产生，而动态类型只有在运行期才能确定方法的接收者，在JAVA虚拟机上不得不使用曲线救国的方式，比如编译时使用占位符，运行的时候生成字节码实现具体类型到占位符的匹配，但是这种方式会导致复杂度增加，方法调用产生的一大堆动态类，所以必须解决 JDK7出现的invoke包和invokedynamic解决了该问题

#### inovke包

该包的主要目的是单纯依靠符号引用来确定目标方法这条路之外，提供一种新的动态确定目标方法的机制，称为方法句柄，相当于C/C++的函数指针，

![image-20210113201740607](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113201740607.png)

实际上是模拟invokevirtual指令，只不过没有固化在虚拟机上，而是在用户代码上动态生成的

反射是在模拟java代码层面，而MethodHandle是在模拟字节码层面

MethodHandle.Lookup相当于方法句柄，用于在指定类中查找符合的方法

#### invokedynamic指令

某种意义上说，invokedynamic指令与MethodHandle机制的作用是一样，都是为了解决invoke*指令固化在虚拟机上，把如何查找目标方法的决定权从虚拟机嫁接到具体用户代码之中

![image-20210113202247830](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113202247830.png)

#### 实战：掌握方法分派规则

![image-20210113202320003](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113202320003.png)

JDK7之前很难解决，7之后可以使用Invoke包解决

![image-20210113202418357](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113202418357.png)

![image-20210113202428099](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113202428099.png)

![image-20210113202436760](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113202436760.png)

#### 基于栈的字节码解释执行引擎

![image-20210113202533918](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113202533918.png)

在JAVA语言中，javac编译器完成了程序代码经过词法分析，语法分析到抽象语法树，再遍历语法树生成字节流的过程，这一部分是在JAVA虚拟机之外进行的，而解释器在虚拟机的内部，因此Java程序的编译就是半独立的实现

#### 基于栈的指令集和基于寄存器的指令集

基于寄存器的话就是直接基于硬件的，速度较快，在CPU内部

栈指令集的缺点是会慢一些，所有的物理主机都是基于寄存器也证实了该点，并且对栈的入栈出栈操作本身也需要指令，并且栈是在内存之中，意味中会更为频繁的访问内存，所以会慢，但是优点也很明显，就是可移植性高

![image-20210113203628942](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113203628942.png)

bipush的作用是将单字节的整型常数值推入栈顶

![image-20210113203738539](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113203738539.png)

一般方法执行的时候，程序计数器，操作数栈，局部变量表都会变化

并且操作数栈顶的最大深度和局部变量表的大小都会被确定下来

### 类加载及执行子系统的案例与实战

#### Tomcat：正统的类加载架构

![image-20210113204256019](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113204256019.png)

放在/common目录下，代表类库可被所有应用程序和Tomcat

放在/server目录下，代表类库只能被tomcat使用

放在/shared目录下，代表类库能被所有应用程序使用

放在/WebApp/Web-INF目录下，代表只能被该应用程序使用

#### OGSI：灵活的类加载架构

OSGI中的每个模块(称为Bundle)与普通的JAVA类库区别不大，但是一个Bundle可以声明它所依赖的package(通过import-Package描述)，也可以声明它允许发布的package(通过Export-Package描述)，在OSGI里里面，从传统的上层模块依赖底层模块变为平级模块之间的依赖

OSGId的Bundle类加载器之间只有规则，没有固定的委派关系。例如，当一个Bundle声明了它所依赖的package，如果有其他Bundle声明发布了该package,那么所有对这个package的类加载动作都要委派给发布它的Bundle加载器去完成

![image-20210113210234478](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113210234478.png)OSGI类加载可能进行的查找规则如下：

找java.*开头的类，委派给父类加载器

否则，委派列表名单内的类，委派给父类加载器

否则，Import列表中的类，委派给Export(也就是声明发布它的Bundle)去加载

否则，查找当前Bundle的Classspath

![image-20210113210523158](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113210523158.png)

![image-20210113210624600](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113210624600.png)

![image-20210113210629198](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210113210629198.png)

这个代理类的实现很简单，为传入接口的每一个方法，以及Object中继承来的equals,hashCode,toString方法都僧成了都应的实现并且上面的this.h 实际上就是父类Proxy中保存的invocationHandle的实例变量，所以说本质上都是调用了this.h.invoke方法，也就是invocationHandle的Invoke()方法，所以说无论调用哪一个方法，相当于就是调用了this.invoke方法

# 程序编译与代码优化

## 前端编译与优化

前端编译器(称为编译器的前端更加准确) ： 把.java文件转变为.class文件的过程

即时编译器(常称为JIT编译器) ： 运行期将字节码转变为成本地机器码的过程

提前编译器：常称AOT编译器，直接把程序编译为与目标机器指令集相关的二进制码

代表性的编译产品

前端编译器：增量式编译器ECJ

即时编译器：G1,G1,Graal

提前编译器：JDK的jaotc丶GNU

### Javac编译器

从JAVA代码的总体结构来看，编译过程大致可以分为1个准备过程和3个处理过程，分别是
**1） 准备过程：初始化插入式注解处理器**

**2）解析与填充符号表过程：包括**

**词法，语法分析，将源代码的字符流转变为标记集合，构造出抽象语法树**

**填充符号表，产生符号地址和符号信息**

**3）插入时注解处理器的处理过程。**

**4）分析与字节码生成过程 包括：**

**标注检查。对语法的静态信息进行检查**

**数据流及控制流分析。对程序动态运行过程进行检查**

**解语法糖。将简化代码编写的语法糖还原为原有的形式**

**字节码生成。将前面各个步骤所生成的信息转换为字节码**

![image-20210121204003176](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210121204003176.png)

![image-20210121204009425](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210121204009425.png)

### 解析与填充符号表

#### 词法语法分析

词法分析是将源代码的字符流变为标记集合的过程，单个元素是程序编写的最小元素，但是标记才是编译的最小元素。关键词，变量名，字面量，运算符都可以作为标记，如 int a = b+2，这句代码中就包含了6个标记 int,a,=,b,+,2

语法分析是根据标记序列构造出抽象语法树的过程，包，修饰符，运算符，接口都可以是一种特定的语法结构

#### 填充符号表

完成了语法分析和词法分析之后，下一个阶段就是对符号表进行填充的过程，符号表由符号地址和符号信息构成的数据结构，可以想象为类似Hash表的那种key-value结构，里面的信息在编译的不同阶段都要被用到。譬如在语义分析中，符号表所产生的内容用于语义检查(如检查一个名字的使用是否和原先声明的一致)和产生中间代码。当对符号名进行地址分配时，符号表是地址分配的直接依据

#### 注解处理器

相当于Lombok，是程序员便捷开发，底层就是依赖的注解处理器实现的

#### 语义分析与字节码生成

经过语法分析以后，编译器获得了程序代码的抽象语法树表示，但是只能保证结构正确的源程序，无法保证程序的语义是符合逻辑的。而语义分析的主要任务是对结构上正确的源程序进行上下文相关性质的检查，譬如类型检查，控制流检查，数据流检查等等

![image-20210121213202563](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210121213202563.png)

后续代码中，只有第一种的语法是没错误的，能够通过检查和编译，其余两种在java是不符合逻辑的。无法编译

这种情况的出现是需要语义分析来保证正确的

##### 标注检查

Javac在编译过程中，语义分析过程大致分为标注检查和数据及控制流分析两个步骤

标注检查检查的内容包括诸如变量使用前是否已经被声明，变量与赋值之间的数据类型是否匹配，等等。上图的例子就术语标注检查的范围，在标注检查中，还会进行一个常量折叠的代码优化

![image-20210123170758542](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123170758542.png)

比如上面的式子，经过常量折叠优化后会变为 int a = 3

##### 数据及控制流分析

数据流分析和控制流分析是对程序上下文逻辑更进一步的验证，它可以检查出局部变量是否被赋值，方法的每条路径是否都有返回值

![image-20210123171108601](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123171108601.png)

上图该方法，两个编译出来的字节码没有任何区别，因为局部变量在常量池中并没有CONSTANT_FIELDred_info的符号引用，自然也不可能存储访问标志access_flag，所以声明对局部变量声明final对运行期没有任何影响，final的定义仅仅由javac在编译期间来保证

##### 解语法糖

还原语法糖，比如自动装箱拆箱，lambda表达式等

##### 字节码生成

字节码生成是javac最后一个步骤，字节码生成阶段不仅仅是把前面各个步骤转换的信息(抽象语法树，符号表)转化成字节码指令写道磁盘中，编译器还进行了少量的代码添加和转换工作

例如前面多次出现的Clinit方法和init方法，类构造器方法和实例构造器方法

### JAVA语法糖的味道

#### 泛型

由于JAVA要保证二进制向后兼容性，譬如jdk1.2编译出来的Class文件，在jdk12中也能运行，由于这种情况，没有使用与C#一样非常棒的泛型，而是使用了不是很好的泛型，泛型擦除

裸类型应被视为所有该类型泛型化实例所有的负类型

![image-20210123172215265](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123172215265.png)

![image-20210123172226231](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123172226231.png)

![image-20210123173323567](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123173323567.png)

因为原始数据类型不支持从int,long到Object的数据转型，索性就又对原始类型进行自动装箱拆箱，成为JAVA泛型慢的主要原因，也是如今Valhalla项目要重点解决的之一

![image-20210123173615954](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123173615954.png)

还必须额外提供Class的类型才能构造一个数组类型，因为list传入后造成类型擦除了

#### 值类型与未来的泛型

![image-20210123173957448](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123173957448.png)

值类型可以与引用类型一样，具有构造函数，属性字段等，但是赋值操作不是赋值地址，而是整体赋值，并且值类型的分配是在方法的调用栈上，不会给垃圾收集子系统造成压力

在Valhalla项目中，JAVA的值类型方案被称为内联类型，计划通过一个inline来定义，即时编译现在只支持C2，是通过逃逸分析优化来处理内联类型的

#### 自动装箱，拆箱与循环遍历

![image-20210123174300530](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123174300530.png)

增强for循环其实底层就是一个迭代器，这也是为何循环遍历需要实现Iterable接口的原因

![image-20210123174416770](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123174416770.png)

如果数组没有大于127，也就是一个字节的大小，那么就会被缓存起来，下次直接获取缓存里面的值不需要创建对象

true,false,true,true,true,true

#### 条件编译

JAVA语言也是支持条件编译的，根据布尔值的真假，编译器会把分支中不成立的代码块消除掉。这一工作在解除语法糖的时候完成

![image-20210123175017600](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123175017600.png)

![image-20210123175027113](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123175027113.png)

#### 	实战：插入式注解处理器

##### 实战目标

对JAVA的命名规范进检查

![image-20210123175232054](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123175232054.png)

##### 代码实现

实现自定义的注解处理器需要继承AbstractProcessor，在JAVA的ElementKind中定义了18个Element。因为抽象语法树的每一个结点都可以看作一个元素

![image-20210123175424873](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123175424873.png)

![image-20210123175437938](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123175437938.png)

![image-20210123175446197](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123175446197.png)

![image-20210123175457894](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123175457894.png)

![image-20210123175504359](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210123175504359.png)

里面的Character.charCount方法可以返回该字母占用几个字节

## 后端编译与优化

### 即时编译器

目前主流的两款商用JAVA虚拟机(HotSpot,OpenJ9)里，java程序最初都是通过解释器进行解释执行的，当虚拟机发现某个方法或者代码块运行特别频繁，就会把这些代码定义为热点代码，为了提升热点代码的执行效率，在运行时，虚拟机会直接将这些代码编译为本地机器码，并且以各种手段进行优化，运行时完成这个任务的后端编译器被称为即时编译器：本章节我们会连接HotSpot虚拟机内的即时编译器的运作过程，此外我们还解决如下几个问题？

为何HotSpot虚拟机要使用解释器与即时编译并存的架构？

答：如果单纯使用解释器，那么每次都要运行调用某个热点方法都需要从字节码转换为本地机器码，效率太低了。

为何HotSpot虚拟机要实现两个(或三个)不同的即时编译器

答：因为在不同场景下需要使用不同复杂程度的即时编译器 如C1前端编译器实现较为简单优化较普通，C2后端编译器实现复杂优化程度高。并且还可以使用分层编译，先使用C1为C2提供复杂优化的铺垫

程序何时使用解释器执行？何时使用编译器执行？

程序初始化或者首次启动的时候或者调用某个方法次数较少的时候都会使用解释器执行，当频繁调用某个代码段会使用C1或者C2即时编译器进行优化

哪些代码会被编译为本地代码？如何编译为本地代码？

被频繁调用的代码。

如何从外部观察到即时编译器的编译过程和编译结果？

#### 解释器和编译器

解释器和编译器两者各有优势：当程序需要迅速启动和执行的时候，解释器优先发挥作用，省去编译的时间，直接运行。当程序启动后，随着时间的推移，编译器逐渐发挥作用，把越来越多的代码编译为本地代码，这样可以减少解释器的中间损耗，获得更好的执行效率

![image-20210125211518685](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210125211518685.png)

如果编译器采用了激进优化出现问题，可以通过逆优化回到解释状态继续执行。也可以回到客户端编译器充当逃生门的角色

JAVA中有2个(或者3个即时编译器)C1,C2,Graal编译器

解释器与编译器搭配使用的方式在虚拟机被称为混合模式

用户可以使用-Xint强制虚拟机运行于解释模型，这时候编译器不介入工作

也可以使用-Xcomp强制虚拟机运行于编译器模式，这时候优先采用编译器运行程序，但是解释器仍然要在编译无法进行的时候介入

![image-20210125212319492](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210125212319492.png)

由于即时编译器编译本地代码需要占用程序运行时间，通常要编译出优化程序较高的代码，所花的时间就越长，并且为了精确优化，还需要解释器在运行时期收集性能监控信息，为了程序启动响应速度和运行速率达到平衡，在编译子系统中增加了分层编译

第0层：程序纯使用解释器执行，解释器不开启性能监控

第1层：使用客户端编译器将字节码编译为本地代码来运行，进行简单可靠的稳定优化，不开启性能监控

第2层：仍然使用客户端编译器，仅开启回边次数，方法次数统计等性能监控

第三层：仍然使用客服端编译器，但是开启所有性能监控，比如分支跳转，虚方法调用版本

第四层：使用服务端编译器进行负责的编译优化，激进优化

实施分层编译后，解释器，客户端编译器和服务端编译器就会同时工作，热点代码可能会被多次编译，用客户端编译获取更高的编译速度，服务端编译器获取更高的编译质量。在解释执行的时候不需要进行性能监控，而在服务端采用高复杂度的优化时，客户端编译器可先采用简答优化为服务端争取时间

![image-20210126132218262](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126132218262.png)

#### 编译对象与触发条件

热点代码主要分为两类

被多次执行的循环体

被多次调用的方法

对于上面两种情况，编译的都会是方法体，而不是单独的循环体。尽管编译动作由循环体触发的，但是编译器依然必须以整个方法作为编译对象，只是执行入口(从方法的第几条字节码指令执行)会稍有不同，编译时会传入入口点字节码序号。因此被很形象的称为栈上替换，因为这种编译方法是发生在方法执行的过程中，直接到方法体里面的循环体进行即时编译

次数问题

热点探测判定方法有两种：

基于采样的热点探测：会周期性的检查各个线程的调用栈顶，如果发现某个方法经常出现在栈顶，那么就认为是热点代码

基于计算器的热点探测：会为每个方法体甚至是代码块建立计数器，统计方法的执行次数，如果超过一定阈值，就认为是热点方法

HotSpot使用的是第二种，为每个方法准备了两类计数器，回边计数器和方法调用计数器回边的意思是指从循环边界向回跳转，也就是从循环回到顶部过程。

方法调用计数器客户端模式下默认阈值为1500次，服务端模式下为10000次，这个阈值可以通过-XX ：CompileThreshold来人为设定。当方法被调用后，会检查是否存在被即时编译过，如果不存在，则将计数器加1，否则执行本地机器码，如果+1以后判断调用计数器和回边计数器之和是否大于调用计数器，如果大于则向即时编译提交一个该方法的代码编译请求

并且该次数不是一个绝对次数，它会随着时间衰减，当一定时间没有调用该方法，计数器会衰减一半，可以使用参数来关闭热度衰减-XX ：UseCounterDecay来关闭热度衰减

![image-20210126203324031](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126203324031.png)

回边计算器阈值设置方法：

虚拟机运行在客户端模式下：方法调用计数器阈值乘以OSR比例除以100 -XX : OnStackReplacePercentage是OSR 默认为933，如果都取默认值，客户端计算器阈值为13995 

在服务端模式下：方法调用计数器阈值*OSR-解释器监控比例的差值除以100，默认值为10700

![image-20210126203708200](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126203708200.png)

![image-20210126203753586](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126203753586.png)

一个是在方法调用的时候发出的即时编译请求，而回边则是方法体里面的，所以被称为栈上替换	

源码MethodOop.hpp

![image-20210126203901547](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126203901547.png)

#### 编译过程

后台执行编译过程的时候，编译器具体会做什么事情呢？

服务端编译器和客户端编译器的编译过程是有所差别的。对于客户端来说大致分为三段式编译器，主要关注局部的优化

第一个阶段：将字节码构造成一种高级代码表示(HIR),与目标机器指令集无关的中间表示，在此之前编译器已在字节码阶段进行简单的方法内联，常量传播优化

第二个阶段：将字节码从HIR转换为低级代码表示LIR(与目标机器指令集有关)，而在此之前以前进行另外一些优化，比如空值检查消除，范围检查消除等

最后阶段就是使用平台相关的后端使用线程扫描算法在LIR上分配寄存器，并在LIR上做窥孔优化，然后产生机器代码

![image-20210126204558153](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126204558153.png)

而服务端编译器则是专门面向服务端的典型应用场景，也是一个能容忍很高优化复杂度的高级编译器，能执行的优化有如：无用代码消除，循环展开，循环表达式外提，消除公共子表达式，常量传播，基本块重排序等，另外还可能进行一些激进优化，如守护内联，分支频率预测

![image-20210126205156250](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126205156250.png)

服务端编译器的中间表示形式是一种名为理想图的程序依赖图，可以使用Ideal Graph Visualizer对这些信息进行分析

![image-20210126205558578](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126205558578.png)

上图的doubleValue方法虽然只有2行字，但是形成的图形蛮复杂的

大致步骤为

1）程序入口：建立栈帧。

2）设置j=0，进行安全点轮询，跳转到4的条件检查

3）j++;

4) 判断j<10000，跳转到3

5）设置i=i*2，进行安全点轮询

![image-20210126205916372](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126205916372.png)

###  提前编译器

因为JAVA编译器的口号是一次编译，到处运行，这与平台的理念冲突，但是在某些领域眼里，只要能获得更好的性能，什么平台中立性，字节膨胀，动态扩展，一些皆可舍弃，唯一的问题提前编译真的会获得更好性能吗

#### 提前编译器的优劣得失

现在提前编译器产品其研究有着两条明显的分支：一条分支是指与传统C++，C编译器类似，在程序运行之前就

将代码提前翻译为机器码的静态翻译动作，第二条分支是把原本即时编译器在运行时要做的编译工作提前保存下来，下次运行这些代码直接把它加载进来使用

我们来说第一条，这是传统的提前编译应用形式，它在java中存在的价值直指即时编译的最大弱点：即时编译器最大弱点就是慢，即时编译要占用程序运行时间和运算资源，即时现在的即时编译已经够快，即时现在已经有了分层编译，即时客户端编译器可以先快速但低质量的编译为服务端慢速但高指向编译争取时间，但是也还是需要时间，无论如何，即时编译消耗的是原本可用于程序运行的时间

举个例子：在编译过程中最耗时的应该是通过过程间分析来获取某个程序点上某个变量是否一定为常量，是否调用某个虚方法只能有单一版本等结论，并且还必须精确，对各种流，路径，上下文敏感，为了得到这些信息，必须在全程序做大量极耗时的计算工作

关于提前编译的第二条路径本质是在为即时编译做缓存加速，去改善java的启动时间，以及预热一段时间才能达到最高性能的问题。这种编译被称为动态提前编译，真正让业界关注的是OracleJDK9中所带的Jaotc提前编译器，比如提前加载最基础的java类库

提前编译代码输出质量一定比即时编译高吗？提前编译因为不是在程序运行时，可以毫无顾忌的使用各种优化手段。这是一个优势，但是即时编译难道就没有有竞争的地方吗？当然是有的。下列有三种即时编译器相对于提前编译器的天然优势：

**性能分析制导优化**：前面提过，在运行时器解释器会收集各种性能监控信息，譬如条件通常会走哪条分支，某个程序抽象类通过会是什么类型等等，这些信息都是在运行期获得然后即时编译进行优化的，然而提前编译是没有办法得到这些信息进行优化的，或者说无法得到唯一的解

**激进预测性优化**：进行激进优化，哪怕错了也没事，最多就是回退到解释器执行，但是静态优化这样是不行的，并且根据类继承关系分析进行虚方法内联

**链接时优化**：对于C或者C++认为这是不可能的，因为主程序和动态链接库的代码在编译时是独立的，其实是可行的

#### 实战：Jaotc的提前编译

使用Jatoc来编译JAVA SE的基础库

![image-20210126213632993](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126213632993.png)

![image-20210126213700459](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210126213700459.png)

nm命令查看是否包含了构造函数和main方法的入口信息，然后通过静态链接库来输出Hello World

### 编译器优化技术

![image-20210127141655786](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127141655786.png)

![image-20210127141714000](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127141714000.png)

![image-20210127141724319](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127141724319.png)

即时编译器对这些代码的优化是建立在代码的中间表示或者机器码之上的，绝不是JAVA源码上去做的，这里为了讲解方便，使用JAVA语言的语法来表示这些优化技术

![image-20210127141858321](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127141858321.png)

首先第一个优化就是方法内联，可以去除方法调用(如查找方法版本，建立栈帧等)，二是为了其他优化建立条件，因为内联后方法体膨胀可以更大范围内的做优化手段

![image-20210127142032064](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127142032064.png)

第二步就是代码冗余访问消除，会将 z = y;前提是中间操作不会改变b.value的值，如果把value看作是一个表达式也可以看作是一种公共子表达式消除优化

![image-20210127142305284](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127142305284.png)

第三步是复写传播  因为 z = y; 所以可以改为y=y; 因为不需要一个额外的变量 y和z是相等的

![image-20210127142351763](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127142351763.png)

第四步无用代码消除

![image-20210127142430234](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127142430234.png)

上图中y=y是没有意义的

四种代表作优化技术，分别是：

最重要的优化技术之一：方法内联

最前沿的优化技术之一：逃逸分析

语言无关的经典优化技术之一：公共子表达式消除

语言相关的经典优化技术之一：数组边界检查消除

##### 方法内联

如果没有方法内联，那么许多优化是没有办法进行的，因为它们在单独的方法里面可能都会有意义，但是组合到一起就没有意义，比如

![image-20210127142818321](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127142818321.png)

上图代码可能分开来看可能都是有意义的，但是经过内联以后，就可以明显看出，方法内联其实就是将目标方法的代码复制到发起调用的方法之中，避免真实的方法调用，按照编译原理的经典优化理论，大多数java方法都是无法内联的

无法内联的原因在于除了invokestatic,和invokespectial指令调用静态方法和父类方法和构造方法，还有一个invokevirtual指令的final方法，其他的都是虚方法，简而言之，大多数java方法都是虚方法，它们在运行过程中可能不止一个方法版本，可能会存在多个，比如父类和子类都有一个方法，如果进行方法内联的话，就不知道需要具体内联哪个方法版本。并且java还推荐程序员去使用多态的方法去调用方法，那么就更糟糕，C++和C选择了默认是非虚方法，如果需要用到多态，就用virtual关键字来修饰，但是java选择了在虚拟机解决这个问题

java虚拟机解决方法是通过**类型继承关系间分析**的技术，这个技术是应用程序范围内的类型分析技术，可以分析一个类是否有子类，接口是否有多于一种的实现，某个子类是否覆盖了父类虚方法等，这样编译器在内联的时候就会根据判断来做出不同的选择

比如当调用目标方法，该目标方法只有单一版本的时候，那么就可以直接假设这种内敛是正确的，因为可能会有反射之类的调用该方法，所以必须为该方法设置好逃生门。比如说回退到解释模式重新执行，或者重新编译，这种方法被称为守护内联

假设通过类型继承关系间分析到该方法确实有多个版本，那么即时编译器会使用内联缓存来缩减方法调用的开销。这种状态下方法调用真实发生了的，但是比起查虚方法表要更快一些，内联缓存是建立在方法入口之前的缓存，在未发生方法调用的时候，内联缓存为空，发生了一次以后会记录该方法的版本信息，以后每次调用该方法会和方法版本进行判断如果每次都一致，那么直接就可以继续使用该内联，被称为单内联缓存。否则不一致就退化为超多态内联缓存，开销相当于查询虚方法表

**所以说JAVA虚拟机进行方法内联都是属于一种激进优化**

##### 逃逸分析

简而言之就是判断局部变量会不会逃脱到方法体之外，或者说线程之外的技术，从不逃逸，方法逃逸，线程逃逸

如果将局部变量作为参数传递给另外一个方法，如果没有逃逸到线程之外，那么可以使用一系列的优化，比如一个方法创建的对象可以直接在栈上分配，不需要在堆里面创建，随这栈帧出而消灭，并且也可以只创建对象里面某个使用的变量等等操作，

栈上分配：就是直接为栈上创建对象，不用在到堆里面去了

标量替换：**不允许方法逃出方法范围内**，JAVA中不可以被拆分的数据类型被称为标量，比如原始基本数据类型，而JAVA中的对象就被称为聚合量，当方法里面调用对象的某个标量的时候，可以不创建该对象，直接让其创建其对象里面需要的成员变量或者标量，直到不能拆分为止

同步消除：线程同步本身是一个相对耗时的过程，如果逃逸分析能够确定一个变量不会逃逸出线程之外，无法被其他线程访问到，那么这个变量的读写肯定不会有竞争，对这个变量的同步措施也可以安全的消除掉

但是逃逸分析计算成本非常高，甚至不能保证逃逸分析性能收益会高于它的消耗。并且如果分析了以后几乎找不到几个不逃逸的对象，那就运行期耗的时间就白白浪费了

C++和C本身就支持栈上分配和标量替换，并且C#支持值类型，也支持标量替换，在灵活运用栈方面java确实是个弱项，以至于Valhalla项目设计了新的inline关键字用于定义java的内联类型，使其进行方法逃逸更加简单

![image-20210127152506891](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127152506891.png)

![image-20210127152530680](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127152530680.png)

![image-20210127152603492](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127152603492.png)

##### 公共子表达式消除

![image-20210127152637182](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127152637182.png)

上面的代码javac编译器不会做任何优化，编译为字节码以后，即时编译器会优化

c*b看作E

那么 d = E * 12 + a +a+ E = 13E+a+a

##### 数组边界消除

java是一门动态安全类的语言，比如说定义了数组foo[]，那么遍历i的时候不能越界，否则会抛出数组越界异常，但是这种每次遍历数据多一次判断有没有数组越界带来的开销也是不小的

不管怎样，数组边界检查都是要做的，但是是不是必须要在运行期一次不漏的检查也是可以商量的

比如可以在javac编译的时候直接通过数据流分析看循环变量的取值范围是不是在[0,arr.length)中，又或者根据数据流获取到数组的长度，直接判断下标有没有越界，因为JAVA中的各种检查，如空指针，除数除以0等等各种隐式开销带来的性能问题，java是怎么解决这些问题的呢？还有一种方案是隐式异常处理，空指针和算术除以0都采用了这个方案，

![image-20210127154136725](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127154136725.png)

虚拟机会注册一个异常处理器，这个处理器是进程层面的异常处理器，如果当访问的时候不为空，那么不会带来任何开销，如果为空，那么就会转到异常处理器抛出空指针异常，这种情况会涉及到用户态到内核态再到用户态的过程，如果一个地方频繁为空，那么反而会更慢，但是JAVA可以根据运行期收集到的性能监控信息来自动选择最何时的方案

### 实战：深入理解Graal编译器

#### 历史背景

jdk10，Graal编译器可以替换服务端编译器，成为HotSpot分层编译中最顶层的即时编译器，并且将Graal从HotSpot代码中分离出来，发布了JVMCI编译器皆苦，JVMCI主要提供三个功能：

用户可以实现自己的java即时编译器来响应Hotspot虚拟机编译的请求

允许编译器访问中与即时编译相关的数据结构，如性能监控信息，类，方法等

提供Hotspot代码缓存的java端抽象表示，允许编译器部署编译完成的二进制机器码

#### JVMCI编译接口

![image-20210127155550958](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127155550958.png)

输入字节码，输出二进制码

![image-20210127155617293](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127155617293.png)

使用Graal编译器需要为HotSpot指定该编译器的位置以及要使用该编译器

![image-20210127155716524](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127155716524.png)

![image-20210127155752644](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127155752644.png)

#### 代码的中间表示

通过可视化工具Ideal Graph Visualizer可以看出在理想图上翻译的过程，编译器内部过程是 字节码->理想图->优化->机器码

理想图是一个有向图，用结点来表示程序中元素，比如变量，操作符等，而边来表示数据流或者控制流

![image-20210127160047133](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127160047133.png)

![image-20210127160110765](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127160110765.png)

虚线代表数据流，但是还必须要考虑方法调用的顺序，使用实现来表示

![image-20210127160314007](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127160314007.png)

可以看到蓝色线是代表数据楼，红色线代表控制流

![image-20210127160416036](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127160416036.png)

![image-20210127160519470](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127160519470.png)

![image-20210127160526767](https://sober-feng.oss-cn-shanghai.aliyuncs.com/learning/pictures/image-20210127160526767.png)

至于代码生成，Graal并不是直接由理想图转换为机器码，而是和其他编译器一样，会先产生低级中级表示LIR，然后由HotSpot后端来生成机器码

# 高效并发

