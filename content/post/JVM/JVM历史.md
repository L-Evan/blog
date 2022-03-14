--- 

title: "JVM历史" # 标题
date: 2022-03-14T17:36:27+08:00    # 创建时间
lastmod: 2022-03-14T17:36:27+08:00 # 最后修改时间
tags: ['JVM']
categories: ['JVM']  # 分类

---


# JVM历史

## 历史

> 其他历史：<https://blog.csdn.net/weixin_37098404/article/details/102706160>
>
> 来源：<https://blog.csdn.net/oneby1314/article/details/107850002>
>
> 总结： <https://www.cnblogs.com/coderxz/p/14519037.html>

### Java 重大事件

1990年，在Sun计算机公司中，由Patrick Naughton、MikeSheridan及James Gosling领导的小组Green Team，开发出的新的程序语言，命名为Oak，后期命名为Java
1995年，**Sun**正式发布Java和HotJava产品，**Java首次公开亮相**。
1996年1月23日Sun Microsystems发布了JDK 1.0。
1998年，JDK1.2版本发布。同时，Sun发布了JSP/Servlet、EJB规范，以及将Java分成了J2EE、J2SE和J2ME。这表明了Java开始向企业、桌面应用和移动设备应用3大领域挺进。
2000年，JDK1.3发布，Java HotSpot Virtual Machine正式发布，成为Java的默认虚拟机。
2002年，JDK1.4发布，古老的Classic虚拟机退出历史舞台。
2003年年底，Java平台的scala正式发布，同年Groovy也加入了Java阵营。
2004年，JDK1.5发布。同时JDK1.5改名为JavaSE5.0。
2006年，JDK6发布。**同年，Java开源并建立了OpenJDK**。顺理成章，Hotspot虚拟机也成为了OpenJDK中的默认虚拟机。
2007年，Java平台迎来了新伙伴Clojure。
2008年，**oracle收购了BEA**，得到了**JRockit虚拟机**。
2009年，Twitter宣布把后台大部分程序从Ruby迁移到Scala，这是Java平台的又一次大规模应用。
2010年，**Oracle收购了Sun**，获得Java商标和最真价值的HotSpot虚拟机。此时，Oracle拥有市场占用率最高的两款虚拟机HotSpot和JRockit，并计划在未来对它们进行整合：HotRockit
2011年，JDK7发布。在JDK1.7u4中，正式启用了新的垃圾回收器G1。
2017年，JDK9发布。将G1设置为默认GC，替代CMS
同年，**IBM的J9开源**，形成了现在的Open J9社区
2018年，Android的Java侵权案判决，Google赔偿Oracle计88亿美元
同年，Oracle宣告JavagE成为历史名词JDBC、JMS、Servlet赠予Eclipse基金会
同年，JDK11发布，LTS版本的JDK，发布革命性的ZGC，调整JDK授权许可
2019年，JDK12发布，加入RedHat领导开发的Shenandoah GC

 ![image-20200727123230088](https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/61f6ba77dbb5ad5c383c090214bcbcfe.png)

### 区别

> <https://www.zhihu.com/question/327162941/answer/730989088>
>
> jdk11后基本相同，主要在版权上

- **JFR是一种分析工具，用于从正在运行的Java应用程序中收集诊断信息和分析数据。**
  - 作为oracle jdk商业组件（jdk11后开源）
- Oracle JDK版本将**每三年发布一次**，Oracle为其版本提供**长期支持**。（商业解决问题）
- OpenJDK版本**每三个月发布一次**，且仅支持对发布的更改，直到下一个版本发布。（商业解决问题jdk11后有）
- openJDK开源所以允许你自己加东西，发行的软件中直接内置，如JB家

## 虚拟机变动

### 主要的虚拟机

> <https://segmentfault.com/a/1190000040137862?sort=newest>

1、 classic VM （jdk1.0  =》 sun公司）

- 只支持**纯解释器**运行（了解python的解释，**翻译与执行一次性完成**,启动快）
- 条件编译智能用外挂（Sun wjit）。解释器和编译器**不能配合工作**。
- 内部工作原理十分简单

2、[Exact VM](https://blog.csdn.net/qq_41813208/article/details/108570189)虚拟机 （jdk1.2 ）

- 解决热点代码探索问题（值或引用，在堆是否移动）
- JIT（解释缓存优化）和解释模式一起使用（混用）  
  - jit 运行频繁时就会认定为热点代码直接转机器语言，启动快还能运行快
- 准确式内存管理，每次定位对象都少了一次间接查找的开销

> 可惜只在Solaris使用**准确式内存管理**，其他平台是classic vm 基于句柄（Handle）的对象查找方式

**2、 开始[Hotspt](https://blog.51cto.com/u_15119353/3304004)称霸** （jdk1.3）

- 小公司设计，**1997被sun收购**，2009被甲骨文收购。

- **计数器去判断jit热点**，到java区域的方法区，并且**兼容**在嵌入式  服务器  移动端  桌面端等应用
  - 一样混用解释和JIT，只是思想上新不同于Exact
  - sun采用Hotspt替代exact**不是真正的技术打败**
  - 高频代码的标准**即使编译和栈上替换**（重要）

**意义**

- HotRocket：jdk8的Hotspot和JRocket进行合并
- 实际效果并不好，JRocket的很多特性没有发挥出来。

3、 Bea的**JRockit虚拟机**（JDK 6）

- **专用于服务端**（不包含解释模式） 08年被oracal收购

- 对敏感应用提供，RealTime毫秒级的JVM响应时间。

- 提供**服务套件**，JMC监控内存泄漏应用生产环境！（对服务端重要出现了监控）
- JRockit内部不包含解释器实现，全部代 码**都靠即时编译器编译后执行**

> **IBM的J9**也是运用服务端块，局限在IBM家的产品。后17由开源部分openj9给了eclipse也就是Eclipse OpenJ9
>
> openj9 是hotspt的一个实现，[内存优化上更好](https://medium.com/criciumadev/new-open-source-jvm-optimized-for-cloud-and-microservices-c75a41aa987d)，CPU还是hotspt好，更合适在[容器和微服务使用](https://www.zhihu.com/question/65452135)

**意义**

- 在JDK1.8当中oracle整合JRockit到HotSpot虚拟机上

---------------------

### 一些小品众的虚拟机

1. CDC/CLDC  塞班系统年代。 2. KVM 轻量，可移植高=》老年机

2. Azul 虚拟机，与硬件强耦合高性能管理多CPU多内存巨大范围回收内存的方案。

   2010转软件在x86实现了ZingJVM快速预热场景表现好

3. LiquidVM（已暂停 BEA家 强耦合可操作系统）

4. Apache Harmony jdk1.5和1.6 由IBM和Inter 联合开发，受到openjdk限制，并且sun不让加入JCP组织导致停止；=》后期大量90%规范引入了Android

5. Microsoft JVM  也就浏览支持javaApplets，后被sun告了，97年抹去

   <img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210520015513973.png" alt="image-20210520015513973" style="zoom:33%;" />  

6. Dalvik Vm  在**Android上基于寄存器的**，不是class而是class可以转到的dex格式，不遵循jvm规范。**一般APK都是**，可支持大部分javaapi当然有的不能。Android 5.0用可提前编译AOT的[ART JVM替换DalviK](https://juejin.cn/post/6844903748058218509)

   > Dalvik 2010andor2.2 加入jit，不过运行时比较耗电 而且需要重新编译
   >
   > ART 2013 Android4.4 aot jit 共存 Android5.0 替代， AOT 是静态编译无需重新细节暂时不做了解

7. 较为新的GraalVM 对比Dcoker

   <img src="https://pig-blog-1252563418.cos.ap-chengdu.myqcloud.com/img/blog/image-20210520020801897.png" alt="image-20210520020801897" style="zoom:50%;" />
