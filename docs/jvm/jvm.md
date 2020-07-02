##JVM知识点

###1、java内存分区
* java虚拟机栈，线程独享的区域，以栈帧方式（局部变量表，操作数栈，动态连接，返回地址），动态连接->指向运行时常量区的指针，该引用可以支持方法调用过程中的动态连接，符号引用等；
* 程序计数器
* 本地方法栈，调用本地方法使用的栈
* 堆，新生代和老年代，eden区，to/from区，老年代等
* 方法区（运行时常量区），永久代，后面成了元数据去metadata
* 直接内存，堆外内存

###2、java对象的创建
* 加载，类加载机制，双亲委派模型
* 解析，验证--准备--解析。验证-->
* 初始化，对象头markword（64位8字节），类型指针（4字节，指向类型信息，通过类型指针可以知道这个实例对象是属于哪个类的）；

###3、jvm堆内分区
* 新生代Eden区，s0和s1。young-gc的区域，新生的对象会放到这里，当不可达以后就会被回收，当时如果是大对象，会直接放到老年代。每次gc对象年纪会增加，同时采用复制算法，gc回收时会从eden区和s-from区将存活对象考到to区域，如果再次逃过gc，达到一定年限后就会复制到老年代中去。eden区又会为每个线程分配tlab区域，线程缓冲区。垃圾收集器有serial-new，parnew，parallelScavenge（吞吐量保证的收集器）
* 老年代。大对象直接分配到老年代，达到年龄的新生代的对象也会拷贝到老年代，老年代会为新生代对象分配提供担保，如果不满足担保条件会触发full-gc，老年代内存不够用也会触发Full-gc，老年代采用标记清除算法。垃圾收集器serial-old，cms，G1等收集器
* 永久代。早期实现，jdk8以后就改成了metadata区，元数据区域，也会被回收
* 堆外内存。DirectBuffer直接使用堆外内存分配，只在full-gc的时候会被回收，所以一定要慎重。
* 对象是否可回收--->可达，从GC-Root可达，哪些gc-root呢？线程栈上引用的对象，方法去中的静态变量引用的对象，方法区常量引用的对象，本地方法栈中引用的对象。
* 为什么不用计数法--->无法解决循环引用计数问题；

###4、gc调优思路
* 明确需求，需要性能满足什么条件？
* 知道现存引用的性能情况，查找瓶颈
* gc工具是否满足条件


###5、java内存模型，happens-before，内存屏障
* 保证多线程可见性的一种机制
* jvm优化会做重排序，编译器优化重排序，指令级别重排序，内存重排序
* 处理器重排序和内存屏障
	* JMM内存屏障指令可以防止处理器重排序，主要是因为缓冲区的影响

* happens-before
	* 一个线程中的操作happens-before线程后续的操作
	* 对monitor的解锁，happens-before对其后续加锁
	* volatile，对一个volatile的写happens-before后续对这个volatile读
	* A happens-before B， B happens-before C，则A happens-before C。

* 顺序一致性
	* 如果发生数据竞争，会出现无法预料的情况
	* 如果正确同步以后就会得到预料中的结果
	* 需要monitor保证顺序一致性
	* Long在操作时可能是线程不安全的，因为会拆分成两个32位的操作

* Volatile语义
	* volatile可以保证内存可见性
	* 当volatile写入后可以保证对所有线程可见，但是不能保证线程安全
	* 对volatile的读写都具有原子性，但是volatile的复合操作是不具有原子性的
	* volatile的写和锁具有相同的内存语义，volatile的读和锁的获取具有相同的内存语义
	* volatile写变量的时候，JMM会把线程对应的缓冲区的值刷新到内存中去
	* volatile在读的时候，JMM会把线程本地的缓冲置为无效，会直接从内存中读取变量，这样就保证了可见性
	* JMM为了实现volatile的内存语义，会限制两种类型的重排序
		* 当第二个操作是volatile写的时候，不能重排序
		* 如果第一个是volatile的读，第二个操作不能重排序
		* 当第一个是volatile的写，第二个是volatile的读的时候不能重排序
		* 实现语义
			* volatile写之前会插入storestore的内存屏障
			* volatile写之后会插入storeload屏障
			* volatile读后会插入loadload屏障
			* volatile读之后会插入loadstore屏障
* 锁的内存语义
	* 线程锁释放时，JMM会把线程对应的本地内存中的共享变量刷新到主存中
	* 线程锁获取时，会把线程本地缓存置为无效，从而必须从主存中去获取值
	* ReentrantLock
		* 公平锁在释放锁最后写volatile变量，获取锁之前先读volatile变量。根据volatile的内存语义，
		* 非公平锁使用unsafe的cas去更改state

	* 并发包concurrent包的实现
		* 首先声明共享变量为volatile
		* 然后，使用cas的原子语义来更新线程之间的同步
		* 以volatile的读写和cas所具有的volatile的读写内存语义来实现线程通信
* final的内存语义
	* 构造函数内对final的写入，和把构造的对象引用赋给另一个引用之间不能重排序
	* 初次读取包含final域的对象引用，和随后初次读final域这两个操作之间不能重排序
	* 禁止编译器把final写重排序到构造函数之外
	* 在final域写之后，构造函数return之前加入storestore屏障

###6、安全代码实践
* 注意数值型防范溢出
* 敏感信息不要序列换，用transient


###7、垃圾收集器
* serial
* serialOld
* ParNew
* ParallelScavenge。吞吐量优先的新生代的垃圾收集器，需要跟parallelOld或者serailOld配合使用，并行的复制算法的垃圾收集器。
	* cms等收集器关注的是尽量缩短垃圾收集线程造成的用户线程的停顿时间
	* parallelscaveng是达到可控的吞吐量，即cpu用于运行用户线程的时间和cpu总消耗时间的比值。
	* 停顿时间越短越适合交互式应用，而吞吐量优先的，更看中cpu的利用率，尽快完成程序的运算任务，适合后台运算而不需要太多交换的任务。
* Parallel old
* cms
	* 以尽量缩短垃圾回收停顿时间的垃圾收集器
	* 初始标记，暂停
	* 并发标记，跟用户线程一起运行
	* 重新标记，更正并发过程中的一些变化
	* 并发清除，跟用户线程一起清理
* G1
	* 将内存化为一些region，新生代和老年代就是一些region的集合
	* 可以实现可预测的停顿，他在垃圾回收的时候可以有计划的避免全部区域的垃圾回收。只对垃圾收集价值最大的region进行回收，后台维护region的回收队列，可以保证在有限的时间内获取尽可能多的收集效率
	* 

###8、对象创建过程
* 检查对应的类有没有被加载进去，如果没有被加载，就加载进jvm
	* 加载的时候类加载的过程
	* 双亲委派模型。加载器自己先不尝试加载，将类加载的任务委托给自己的父类加载器，只有当父类加载器加载不了的时候才尝试自己加载
	* 类加载过程，验证、准备、解析
	* 验证。将字节流读入jvm，并进行校验
	* 准备。准备阶段是为类变量分配内存并设置初始值，方法区（1.8以后再meta区）分配
	* 解析。将常量池中的符号引用替换成直接引用
* 如果类被加载进去了，知道类的信息，可以知道一个实例的具体大小，为类分配内存空间
* 指针的方式分配：内存分配过程中可能出现并发问题，造成内存分配重叠出现问题，这里内部使用cas来保证更新操作的原子性
* 线程不同空间划分：每个线程在堆中有TLAB线程本地缓冲区
* 内存分配好了就对内存清空置为0
* 对象进行必要设置
	* 对象头信息（8+4）
	* markword有8字节，类型指针有4字节，一共12字节
	* markword有对象hash码，gc分代年龄，偏向锁等信息

###9、类加载
* 验证
* 准备
* 解析
* 初始化
	* 初始化时类加载的最后一步
	* 初始化时执行类构造器的<cinit>()方法
	* <cinit>()方法是编译器根据所有类变量的赋值动作和静态语句块的语句合并而生成的
	* <cinit>()方法更类的构造函数是不相同的，可有可无
	* 父类的<cinit>()方法优先于子类执行，所以父类的静态语句的执行时优先于子类的静态语句的而执行的
	* <cinit>()方法可有可无，只有内部有变量的复制动作和静态语句的时候才可能
* 类加载器
	* 比较两个类是不是相等，判断其是不是由同一个类加载器加载的，同时类名称是不是相等的
	* 类加载器--->双亲委派模型
		* 启动类加载器
		* 拓展类加载器
		* 应用程序类加载器
		* 自定义类加载器

###10、字节码执行引擎
* 运行时栈帧结构。
	* 局部变量表。存放方法参数，方法内部局部变量，是线程安全的
	* 操作数栈。
	* 动态连接。一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程中的动态连接
	* 方法返回地址
* 方法调用
	* 一切方法调用在class里面都是一个符号引用，而不是真正执行时内存布局的入口地址。
	* 解析
		* 类加载阶段会把部分符号引用转换成直接引用
		* 符号引用是一组用来描述锁引用的目标的，其可以是任何形式的字面量
	* 分派
		* 静态分派和动态分派
		* 重载是以参数的静态类型判断而不是实际类型做判断，并且静态类型是在编译期可知的
		* 静态分派发生在方法的编译阶段，静态分派实际上并不是有虚拟机来指定的
		* 静态方法也是有重载版本的，选择重载版本也是通过静态分派来完成的
		* 动态分派跟--->重写，密切相关的
		* 动态分派是通过运行期的实际类型来确定方法的具体执行版本的
* 动态代理，spring内部就是通过动态代理的方式对Bean进行增强

###11、编译器优化
（1）早期优化

* 编译器优化
* 常量折叠、变量检查
* 数据控制，局部变量声明为final
* 解语法糖
* java语法糖
	* 泛型和类型擦除，泛型技术是java的语法糖，泛型的类型擦除使得泛型后方法无法用类型进行区分
	* 自动装箱、拆箱
	* 循环遍历foreach
	* 条件编译，条件常量直接完成编译，if一直是真值，后续else代码就不会编译
	* 注解处理器，利用反射

（2）运行期优化

* 热点代码即时编译
	* c1编译器
	* c2编译器
	* 热点代码-->基于采样决定是否热点代码或者基于计数决定是否热点代码
* 公共子表达式消除
* 方法内联
* 逃逸分析
	* 栈上分配，如果确定一个对象不会逃出方法之外，就可以在栈上分配，可以快速消失回收
	* 同步消除。如果确定变量不会逃出线程，就没必要给变量加上同步
	* 标量替换。如果一个对象不会被外部访问，而且该对象可以被裁开，那么久不会创建这个变量，而是直接创建其相关的成员变量来替换。
	