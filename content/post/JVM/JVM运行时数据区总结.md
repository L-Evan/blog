--- 

title: "JVM运行时数据区总结" # 标题
date: 2022-03-15T01:55:34+08:00    # 创建时间
lastmod: 2022-03-15T01:55:34+08:00 # 最后修改时间
tags: ['JVM']
categories: ['JVM']  # 分类

---

# JVM运行时数据区

## Java 8 虚拟机范围

1. 程序计数器（Program Counter Register）  （线程独有）
2. Java虚拟机栈（Java Virtual Machine Stacks）（线程独有）
3. 本地方法栈（Native Method Stack）（线程独有）
4. Java堆（Java Heap）（线程共有）
5. 方法区（Methed Area）（类信息）（线程共有）

### 分类

- 线程独有：虚拟机栈，本地方法栈，程序计数器（PC寄存器） 
- 线程共有：方法区，堆 （堆外内存  ->（元空间/本地内存/永久代/代码缓存））
- 不会产生OOM：只有PC寄存器
- 产生SOF： 虚拟机栈
- 不会垃圾回收：虚拟机栈，本地方法栈

## 组成成分

### pc寄存器（java模拟的运行代码偏移）

对物理PC寄存器的一种抽象模拟，一块较小的内存空间, 字节码的行号指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器。

- 如果是遇到本地方法（native 方法），这个方法**不是 JVM 来具体执行**，所以**程序计数器不需要记录**了
  - 这个是因为在**操作系统层面也有一个程序计数器**，这个 会记录本地代码的执行的地址，所以在执行 native 方法时，JVM 中程序计数器的值为空(Undefined)。
  - 对字节码位置的存储，所以是模拟的寄存器
- 记录指令运行位置，用于保存上下文，恢复现场和执行控制
- 唯一一个不会发生OOM的地方

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527170601938.png" alt="image-20210527170601938" style="zoom:50%;" />

### 虚拟机栈

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527184017793.png" alt="image-20210527184017793" style="zoom:50%;" />

#### 特点

- 指令精简：方便实现，不过**同样功能得更多指令运行**
  - 为了编译器容易实现
- 跨平台性：更加兼容，因为不是所有平台，都有配套寄存器
  - 所以采用**栈兼容一些复杂指令** ，处理也导致速度慢了一些
- **内存私有，它的生命周期和线程相同**
- 是可以**动态扩展的**，如果扩展时无法申请到足够的内存就会抛出OutOfMemoryError异常
  - 可以jvm调优
- 不进行垃圾回收

#### 看栈帧大小

> java.tools.main tools.java.xss

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527190916159.png" alt="image-20210527190916159" style="zoom: 50%;" />

- IDEA查看栈帧

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527190824556.png" alt="image-20210527190824556" style="zoom: 33%;" /><img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527191636217.png" alt="image-20210527191636217" style="zoom: 33%;" />

#### 栈帧组成

**每个方法在执行的同时都会创建一个线帧**（Stack Frame），可以理解栈的进出单位是栈帧，而栈帧单位是运行代码。

- **局部变量表**：局部变量引用，方法参数，返回值
  - 类似指针数组，大小在编译的时候确认，看方法体运行时最大有多少临时变量
- **异常表**：异常跳转位置
- **操作数栈**（对表达式的运算栈）
- **动态链接**
- **方法返回地址**

##### 局部变量表

- 在存储时，一行一个索引，32位以下都会存储一个slot（行），**long和double 会存在2个slot**
  - 并且对于变量表的生成，是固定的所以jvm会**根据作用域去复用变量的slot**

- 在构造方法和实例方法中，**0号索引变量为this**
  - 静态方法不是，因为无需this

###### 性能调优注意点

1. 局部变量表大小固定，对于常量等如果考虑复用slot，可以有一定优点
2. 局部变量表中的引用丢失，会触发堆（栈本身的不会）的回收
   1. 所以局部变量表的直接或间接引用都会导致不被回收。
      1. 所以设null加速回收

###### 从字节码文件看

- 数组的表示形式

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527201208016.png" alt="局部变量表图" style="zoom: 25%;" />

- 槽位长度

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527201256203.png" alt="局部变量表大小" style="zoom:50%;" />

- 槽位复用

  ![image-20210527210803659](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527210803659.png)

##### 操作数栈

栈的大小，一样在编译确定大小，因为运行带字节码代码，push和load的次数可以算出最大多大。

- push进栈，load加载局部变量表的 进栈，istore_1  出栈进局部变量表
- ireturn 是**返回栈顶元素**，所以前面会进行iload吧返回值进栈
  - 注意try{return}finaly{return } 的时候，只进行finaly是因为ireturn 仅仅**只是返回栈顶元素**，终究看栈顶
- HotSpot 虚拟机采用了栈顶缓存技术，栈顶存到真实的寄存器（待了解）

###### 从字节码看

- 栈高度

![操作数栈的深度](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527212342543.png)

- 操作流程（下面指操作数栈）

1. bipush  数字进栈
2. istore_1  存进局部变量表索引1，并且出栈
3. iload _1  加载局部变量索引1进栈
4. iadd  进行相加

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527213008047.png" alt="image-20210527213008047" style="zoom:50%;" />





##### 动态链接

动态链接是链接的一种（静态链接在类加载的时候链接）

1. java源码转成字节码文件的时候，会将变量和方法作为**符号引用**（#n）存储在**字节码文件中**的常量池（也就是以后的运行时常量池），包括返回值类型，使用到的类等。

2. 然后在类加载的时候对字节码文件的**符号引用转为栈帧引用**

- 栈帧引用是实现代码数据复用的关键，指向与运行时常量池的**对应关系**， 而运行时常量池是在jvm虚拟机的方法区位置。

> 动态链接一般属于动态多分派，下面方法调用介绍

###### 字节码文件中

- 符号引用

![常量池图](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527221122842.png)

- 动态链接流程

  <img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210527221733559.png" alt="image-20210527221733559" style="zoom:50%;" />

###### 方法调用

**一些概念区分**

静/动态链接：**符号引用转直接引用**的时间，是否在编译期间确定位置、是否在链接阶段确定方法（链接中的解析阶段）完成

- 是否在方法运行时确定。

> [早/晚绑定](https://www.cnblogs.com/liujunj/p/13381658.html)：同上，只是叫法问题，（面向过程基本都是早起绑定）
>
> 静/动（前/后）绑定：针对的是方法和属性

虚/非虚方法：虚方法是可能会变化的，产生变化的方法

> final 是非虚的，不变化的，可是使用的是虚指令invokevirtual，是历史设计遗留原因，但是类加载的时候仍然可以确定并类加载直接绑定

静/动态分派：多态的一种表现，**分派这个词本身就是动态**的只是翻译问题。重载，重写方法调用  （深入虚拟机 8.3） 

- 静态分派，更多的是达到**方法重载**（参数不同，）

  - 对于Java来说是**静态多分派**，有多个选择的静态绑定(按**调用时参数类型**直接静态匹配对应方法)
  - 在调用的时候，如果没有一样的类型，那么由**数据类型优先级**分派，也就是由基础类型向类继承到实现接口转化，通过多省略类型，进行按**优先级分派**
- 动态分派，更多是**方法重写**（详见invokevirtual虚指令**检查实际类型**）
  - 依托虚指令的方法追寻，由**数据本身实际类型再到继承关系，由下往上追寻合适的方法**，从而达到多态的方法重写。（**因为静态多分派原因**，参数和实际类型不一致，所以分派是动态的**运行时确定**）
  - 对于Java来说是**动态单分派**，调用方法的时候，根据实际类型去匹配调用的方法（包括父类调用方法也会向子类执行），属性是静态的

###### 实例上区分

1、 符号引用转换成直接引用的过程区别

1.1 静态链接：**编译期间就可以**获得【等同于】早期绑定

早期绑定： **确定的方法**（面向过程基本都是）

- 只有**父类方法、final(非虚方法可是用虚指令), static、private** 和 **构造方法** 是静态绑定，多态的调用会失效

~~~java
   public static void main(String[] args) {
          Father son = new Son();
          System.out.println(son.str);//father
          System.out.println(son.getStr());//son
   } 
~~~

1.2 动态链接：编译期间**不能**获得 【等同于】 晚期绑定： **接口和抽象**（类比虚函数，运行的时候**根据引用**去实际调用）

2、 指令层次区别静态和动态链接

- **invokedynamic指令**（jd7新增，只通过外部工具生成，jdk8开始出现lamdal）让java由静态语言新加了动态语言的特性(待了解)

  > tips 增加对动态和静态语言的解释

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210528160504941.png" alt="image-20210528160504941" style="zoom:80%;" />

###### **方法重写的本质**

- 动态单分派

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210528164348772.png" alt="image-20210528164348772" style="zoom:50%;" />

**会出现的几个运行时异常**

- IllegalAcessError, 没权限访问，一般是访问非虚方法导致（maven重复加载jar，缺少jar之类的）？
- 继承树中没找到方法或者类，也就是符号引用有，而找不到实际的方法引用，会导致爆AbstructMethorError？

**虚方法表**

为了解决方法重写的本质，在动态链接的过程，所以建立了**虚方法表**（类的方法区），在类的解析阶段创建

- 虚方法表： 记录**虚方法调用的是哪个类的方法**。

  - 每个类都有自己的虚方法表，表示了方法是调用 哪里的方法，这样让查找更快了，节省查找的过程！
  - 此处是虚方法代表是真正的虚方法，而不是只说虚指令，final采用虚指令可是属于虚方法

  <img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210528170341733.png" alt="虚方法表" style="zoom:50%;" />

###### 总结

本人理解是： ctrl+点击方法位置，一般就是编译期确定的位置，也就是**静态多分派**后的位置。而**解析阶段**对符号链接尝试转为直接链接时，通过对其**字节码指令**的不同，如果是**非虚方法**，可以直接进行**静态链接**，如果是虚方法进行**动态链接**运行时确定，采用的是**动态多分派**。

- 此处是虚方法代表是真正的虚方法，而不是只说虚指令，final采用虚指令可是属于虚方法

##### 异常表

在表中记录，方法运行在try范围内，内部产生异常后，此栈帧运行位置的行数。

###### 从字节码文件

- 对异常捕获的时候会在异常表索引

  - 异常表参数解释：标明try区域是from到to也就是  **4-8**  ， 出现异常是IOException 就跳到target也就是 **11（carch io）**进行处理

  记得一点：无论哪种方式反回，结束后，**pc寄存器存的都是之前调用方法（之前栈帧）的下一条指令的地址**。

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210528185513492.png" alt="image-20210528185513492" style="zoom:50%;" />



### 本地方法栈

Java虚拟机规范中对于本地方法栈（服务native）没有特殊的要求，虚拟机可以自由的实现它，因此在Sun HotSpot虚拟机直接把**本地方法栈和虚拟机栈合二为一**了.并且也是**线程私有**的，**栈长度有限制**可动态扩展基本和java栈一样

- 本地方法可以通过本地方法接口来**访问虚拟机内部的运行时数据区**。
- 它甚至可以直接使用**本地处理器中的寄存器**
- 直接从本地内存的**堆中分配任意数量的内存**。
- 核心用来做一些底层操作，和调用外部本地接口

**并不是所有的JVM都支持本地方法**。

> 原理是通过执行引擎去调用本地方法接口，也就是对应注入的本地方法

![image-20220314201753771](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20220314201753771.png)



### 方法区

#### 方法区内容

![image-20210529221219448](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210529221219448.png)

##### 类信息

1. 类型信息 

   类的名称：包名和全类名

   类名对应的继承实现链名

   类对应的修饰符

   实现接口的列表（列表存储接口）

   - 域（属性）信息：private修饰符，名称，类型
   - 方法信息：方法名，返回，参数（类型和数量顺序），修饰符，**方法实体（字节码）**对应有他的局部变量表，操作数栈大小等，异常表（异常的范围和对应的类和索引位置）
   - 问题：异常表和虚方法表的位置
   - 局部变量表大小是怎么确定的？

2. 静态变量，jit编译后的代码缓存
3. 运行时常量

- 常量final static 基本数据类型转字节码时直接写出来了
- static在**类加载的准备阶段**就初始化**default 值**（0、0.0、null），初始化阶段赋值（就是生成的init方法内，也就是静态块执行顺序）
- final static的图例

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210529223930517.png" alt="final static的图例" style="zoom: 50%;" /><img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210529223939327.png" alt="image-20210529223939327" style="zoom:33%;" />

##### [常量池](https://www.jianshu.com/p/cf78e68e3a99)

编译阶段会将常量对于的以**符号引用存储于文件**中

###### **Class文件常量池**

- 字符串、类引用、数值、字段、方法（开始加载的时候加载）

1、**编译期明确的字面量**

- **文本字符串**

  ~~~
   #9 = Utf8               s
   #3 = String             #31            // abc
   #31 = Utf8              abc
  ~~~

-  **用final修饰的**成员变量，包括**静态变量**、**实例变量**和**局部变量**

~~~
 #11 = Utf8               f
 #12 = Utf8               ConstantValue
 #13 = Integer            257
~~~

- 一般除去表达式,给变量赋值时**,等号右边都可以认为是字面量**。 （也就是字符串）
- 字面量分为字符串字面量(string literal )、数组字面量(array literal)和对象字面量(object literal), 另外还有函数字面量(function literal)

2 、符号引用字符串

如**类和接口**的**全限定名**，也就是`Ljava/lang/String;`

###### **运行时常量池**

1、运行时变量

​	包含多种变量，不是**符号引用字符串地址**，而是真实地址，并可动态改变

- 也就是运行中，**解析后的方法，解析后的字段引用**

###### [字符串常量池](https://blog.51cto.com/u_15009384/2563484)

 **jvm优化**

**-XX:StringTableSize=60013**

- JDK1.6默认为1009（受到PermGen固定大小的限制），JDK1.7之后默认为60013（60013也是质数散列好），**字符串常量池底层为HashTable**，合理增大常量池大小会解决[Hash](https://so.csdn.net/so/search?q=Hash&spm=1001.2101.3001.7020)冲突问题
  - 60013 =》 大概30000个不同的字符串到常量池
- JDK1.8开始1009是可以设置的最小值

> 注意：虽然hashtable被废弃了，并不影响string pool的实现。因为底层是c++的，问题是后面的原理还是不是hashtable待了解
>
> https://stackoverflow.com/questions/35498336/how-does-java-implement-string-pooling 

#### JVM规范

- 方法区包含编译代码缓存，静态变量，运行时常量池（类信息，方法，字段，常量池表->存字面量和符号引用**包括字符串常量池**）
- 方法区应该是属于堆的一部分的
  - 方法区和堆理论上是一样的（jvm规范中），**HotSop虚拟机将他们区分开了而已**，独立堆的内存空间

#### 特点

- 可能oom场景：加载太多jar，tomcat太多工程，太多反射类

- 方法区是虚拟机的一个规范，具体实现如hotSop是采用永久代实现，8之后是元空间实现
  - J9（媲美hotSop内存优化好）和JR（被hotsop收购）本来就是元空间

#### HotSop实现

**永久代**：非堆，方法区不等于永久代，逻辑是不同的，**兼容分代回收机制**（fullGC，非常难回收）

- 也就是为兼容实现对方法区的分代回收，而创出**永久代存方法区**
- 在触发full gc的情况下，永久代也会被进行垃圾回收。永久代的内存溢出也就是 pergen space

**元空间**：存在本地内存，存永久代中**除了字符串常量池和静态变量的信息**

> 同样的元空间的大小和初始大小建议**设置相同**，因为每次调整都是一次fullGC**卸载没用的类**

#### 历史变化

> [常量池介绍](https://blog.csdn.net/zm13007310400/article/details/77534349)

JDK6： 方法区在永久代， 此时运行时常量池（含字符串常量池和静态变量），代码缓存等都在方法区

JDK7：方法区在永久代，不过**开始去永久代**，此时**字符串常量池和静态变量**移动到**堆的老年代**， 其他运行时常量池仍在方法区

- 字符串常量池和静态变量**逻辑上属于方法区**，但是**实际（静态变量）存放在堆**内存中

JDK8：方法区在元空间，去除永久代转移到**新出现的元空间**，老年区不变

- 元空间让方法区从**虚拟机内存转本地内存**

#### 方法区数据注意（重要）

1. 元空间并不是类加载的完全存储在本地内存

   1. **加载类型信息等公共信息放在方法区**后
   2. 同时在堆区中创建类对象引用，将静态变量存在那个类的堆区

2. 静态属性在new对象时，无论1.6-1.8都是在堆，而引用所占的8字节，在方法区或者堆区（我觉得引用在方法区随着跑元空间）

   1. 一般说变量在方法区还是在堆区还是在元空间，**一般说的是他的引用变量**而不算说他new出实体数据，对于new的实体数据都在堆（1.6-1.8都是**1.6字符串用intent后也在永久代也在堆**）。

3. Java 对象的大小默认是按照 **8 字节对齐**，也就是说 Java 对象的大小必须是 [8 字节的倍数](https://www.cnblogs.com/rickiyang/p/14206724.html#:~:text=%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%BC%95%E7%94%A8%E7%B1%BB%E5%9E%8B%EF%BC%9A64,%E5%8E%8B%E7%BC%A9%E5%90%8E16%20%E5%AD%97%E8%8A%82%E3%80%82)。

4. 字符串常量池在堆是为了方便GC（[intern](https://www.jianshu.com/p/2dc4c3dd5376)意思是返回常量池中的引用，没有就吧自己加进去）

   JDK6 字符串实体是存在方法区的，调用intern会吧**堆中字符串的完整复制到方法区字符串常量池也就是永久代**。

   - 宽松意义直接进老年代

   JDK7、8中intern是不会将字符串复制到常量池，而只将该字符串的引用保存到常量池。也就是保存到老年代的常量池

   - 分代机制迟早进老年代

#### 图例

- 历史变化图

  <img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210605155542402.png" alt="image-20210605155542402" style="zoom:50%;" />

  <img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210605163849202.png" alt="image-20210605163849202" style="zoom:50%;" />

  

  <img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210605164027072.png" alt="image-20210605164027072" style="zoom:50%;" />

  <img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210605164109126.png" alt="image-20210605164109126" style="zoom:50%;" />

  ![image-20210529213210839](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210529213210839-1647276784567.png)

- 永久代图例

![img](https://img2020.cnblogs.com/blog/1534147/202003/1534147-20200327123751777-1521977355.png)

#### 方法区测试类加载

- 看看会不会oom爆炸

<img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210529215850984.png" alt="image-20210529215850984" style="zoom:50%;" />



# tips

## Jvm一些参数

>  JVM虚拟机规范：https://docs.oracle.com/javase/specs/index.html
>
> jvm一些tools参数：https://docs.oracle.com/en/java/javase/11/tools/
>
> javap -v *.class  查看文件解析



## 动态和静态语言

> 科普动态和静态语言（弱类型，编译不确定类型为动态语言）

**lamdal表达式生成的地方会出现invkedynamic指令**，支持了函数编程也让java可以由此去兼容各个语言，并在平台上运行

- 改变的是jvm的规范而不是java规范？ 
- 简单说就是通过值去确定类型，在go的:= 方法很像

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210221152028185.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTkxMDc3OQ==,size_16,color_FFFFFF,t_70)



## 永久代为什么会被替换

![image-20210605171941075](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210605171941075.png)

## 字符串常量池为啥要迁移

![image-20210605173324701](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210605173324701.png)

## 思考

### 逃逸分析、标量替换、栈上分配、 同步消除（文章待补充）

> https://juejin.cn/post/6992440698993639431

HotSop虚拟机的**逃逸分析**具体实现在于**标量替换**，是对new出来的**对象的成员变量属性**的引用值，**存储在栈上**，引用对应new出来的对象**仍然需要分配在堆**区（只是省去了多一个类，多一个类头等）。

~~~java
// 替换前(hotsop 栈上分配具体实现好像是标量替换，而栈上分配的前提是逃逸分析)
public void personTest(){
    Person person=new Person();
    person.id=01;
    person.age=22;
}
class Person{
     int id; 
     int age;
} 
// 替换后
public void personTest(){
    Person person=new Person();
    person.id=01;
    person.age=22;
    //开启标量替换后
    int id =01;
    int age=22;
} 
~~~

## 其他

- 操作数栈是数组实现的，而非链表，方便方法代码直接取出编号（忘记哪里看的了，我自己觉得应该是方便限制大小）
- 程序计数器是当前运行的序号，**取到指令后会立马+1**，所以你取到指令就在运行你当前代码，运行了当前代码证明pc那时候已经加过1了所以是下条指令的地址

- 类加载
  - 类加载器并不需要等到某个类被“首次主动使用”时再加载它，JVM规范允许类加载器在**预料某个类将要被使用时**就预先加载它
    - 也就是**可以提前加载**
  - 如果在预先加载的过程中遇到了.class文件缺失或存在错误，类加载器必须在程序**首次主动使用该类时才报告错误**（LinkageError错误）
    - 如果这个类一直没有被程序主动使用，那么类加载器就不会报告错误
    - **提前加载不到，不会提前报错**



[早晚概念解释]: https://blog.csdn.net/weixin_45910779/article/details/113910933






