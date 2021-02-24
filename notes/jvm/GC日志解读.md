# GC日志解读和分析

## 目标
* 并行GC演示和分析
* 串行GC演示和分析
* CMSGC演示和分析
* G1GC演示和分析

## 如下是模拟内存溢出的Java代码
>案例代码存在生产垃圾对象，当垃圾对象超过最大堆内存时，就造成内存溢出

``` Java
import java.util.Random;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.LongAdder;
/*
演示GC日志生成与解读
*/
public class GCLogAnalysis {
    private static Random random = new Random();
    public static void main(String[] args) {
        // 当前毫秒时间戳
        long startMillis = System.currentTimeMillis();
        // 持续运行毫秒数; 可根据需要进行修改
        long timeoutMillis = TimeUnit.SECONDS.toMillis(1);
        // 结束时间戳
        long endMillis = startMillis + timeoutMillis;
        LongAdder counter = new LongAdder();
        System.out.println("正在执行...");
        // 缓存一部分对象; 进入老年代
        int cacheSize = 2000;
        Object[] cachedGarbage = new Object[cacheSize];
        // 在此时间范围内,持续循环
        while (System.currentTimeMillis() < endMillis) {
            // 生成垃圾对象
            Object garbage = generateGarbage(100*1024);
            counter.increment();
            int randomIndex = random.nextInt(2 * cacheSize);
            if (randomIndex < cacheSize) {
                cachedGarbage[randomIndex] = garbage;
            }
        }
        System.out.println("执行结束!共生成对象次数:" + counter.longValue());
    }

    // 生成对象
    private static Object generateGarbage(int max) {
        int randomSize = random.nextInt(max);
        int type = randomSize % 4;
        Object result = null;
        switch (type) {
            case 0:
                result = new int[randomSize];
                break;
            case 1:
                result = new byte[randomSize];
                break;
            case 2:
                result = new double[randomSize];
                break;
            default:
                StringBuilder builder = new StringBuilder();
                String randomString = "randomString-Anything";
                while (builder.length() < randomSize) {
                    builder.append(randomString);
                    builder.append(max);
                    builder.append(randomSize);
                }
                result = builder.toString();
                break;
        }
        return result;
    }
}
```
## 演示GC日志生成与解读
* 当时在win下编译的，需要加参数-encoding UTF-8，linux不需要  （win默认使用GBDK字符，如果文件是UTF-8不能编译）
>D:\F盘\JavaCourseCodes\01jvm>javac -encoding UTF-8 GCLogAnalysis.java
*  执行GCLogAnalysis文件，打印的GC日志生产到当前目录下demo.log
>D:\F盘\JavaCourseCodes\01jvm>java -Xloggc:gc.demo.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps
> -XX:MaxHeapSize=4244185088,如果没有设置最大堆内存，默认是服务器的最大内存的1/4大小 ，默认采用UseParallelGC 并行GC
```log
Java HotSpot(TM) 64-Bit Server VM (25.201-b09) for windows-amd64 JRE (1.8.0_201-b09), built on Dec 15 2018 18:36:39 by "java_re" with MS VC++ 10.0 (VS2010)
Memory: 4k page, physical 16578848k(2260340k free), swap 30350536k(4206764k free)
CommandLine flags: -XX:InitialHeapSize=265261568 -XX:MaxHeapSize=4244185088 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC
//-XX:MaxHeapSize=4244185088,如果没有设置最大堆内存，默认是服务器的最大内存的1/4大小 ，默认采用UseParallelGC 并行GC

2021-02-23T07:34:15.501+0800: 0.274: [GC (Allocation Failure) [PSYoungGen: 65024K->10747K(75776K)] 65024K->23007K(249344K), 0.0072693 secs] [Times: user=0.03 sys=0.08, real=0.01 secs] 
//65024K->10747K(75776K)] 65024K->23007K(249344K), 0.0072693 secs] :young区从65M >10M，Old区从65M > 23M  ，old区减到23M，说明此时部分对象从young提升到old区
//Times: user=0.03   用户线程耗时  sys=0.08   系统耗时, real=0.01 secs STW暂停时间
2021-02-23T07:34:15.531+0800: 0.304: [GC (Allocation Failure) [PSYoungGen: 75204K->10744K(140800K)] 87463K->42977K(314368K), 0.0097131 secs] [Times: user=0.03 sys=0.05, real=0.01 secs] 
2021-02-23T07:34:15.610+0800: 0.382: [GC (Allocation Failure) [PSYoungGen: 140564K->10747K(140800K)] 172797K->88342K(314368K), 0.0153475 secs] [Times: user=0.03 sys=0.08, real=0.02 secs] 
2021-02-23T07:34:15.664+0800: 0.436: [GC (Allocation Failure) [PSYoungGen: 140795K->10749K(270848K)] 218390K->127729K(444416K), 0.0157620 secs] [Times: user=0.02 sys=0.08, real=0.02 secs] 
2021-02-23T07:34:15.810+0800: 0.583: [GC (Allocation Failure) [PSYoungGen: 270845K->10747K(270848K)] 387825K->204045K(465408K), 0.0291165 secs] [Times: user=0.03 sys=0.19, real=0.03 secs] 
2021-02-23T07:34:15.839+0800: 0.612: [Full GC (Ergonomics) [PSYoungGen: 10747K->0K(270848K)] [ParOldGen: 193297K->170244K(330240K)] 204045K->170244K(601088K), [Metaspace: 2628K->2628K(1056768K)], 0.0301415 secs] [Times: user=0.16 sys=0.00, real=0.03 secs] 
//经过五次youngGC（每次GC耗时递增），old区越来越大，此时进行fullGC一次，， 0.0301415 secs对比youngGC，fullGC耗时非常多
//fungGC时，young区直接减到0，old区从193M > 170M
2021-02-23T07:34:15.941+0800: 0.713: [GC (Allocation Failure) [PSYoungGen: 260096K->78120K(546816K)] 430340K->248364K(877056K), 0.0249608 secs] [Times: user=0.02 sys=0.06, real=0.03 secs] 
2021-02-23T07:34:16.189+0800: 0.961: [GC (Allocation Failure) [PSYoungGen: 546600K->101880K(588800K)] 716844K->353984K(919040K), 0.0510303 secs] [Times: user=0.11 sys=0.22, real=0.05 secs] 
2021-02-23T07:34:16.240+0800: 1.013: [Full GC (Ergonomics) [PSYoungGen: 101880K->0K(588800K)] [ParOldGen: 252103K->266559K(460800K)] 353984K->266559K(1049600K), [Metaspace: 2628K->2628K(1056768K)], 0.0374054 secs] [Times: user=0.20 sys=0.00, real=0.04 secs] 
2021-02-23T07:34:16.398+0800: 1.171: [GC (Allocation Failure) [PSYoungGen: 486345K->142934K(982016K)] 752905K->409493K(1442816K), 0.0431602 secs] [Times: user=0.05 sys=0.22, real=0.04 secs] 
Heap
 PSYoungGen      total 982016K, used 176414K [0x000000076bb00000, 0x00000007b5680000, 0x00000007c0000000)
  eden space 821760K, 4% used [0x000000076bb00000,0x000000076dbb2160,0x000000079dd80000)
  from space 160256K, 89% used [0x00000007a8e00000,0x00000007b1995908,0x00000007b2a80000)
  to   space 180736K, 0% used [0x000000079dd80000,0x000000079dd80000,0x00000007a8e00000)
 ParOldGen       total 460800K, used 266559K [0x00000006c3000000, 0x00000006df200000, 0x000000076bb00000)
  object space 460800K, 57% used [0x00000006c3000000,0x00000006d344fda0,0x00000006df200000)
 Metaspace       used 2634K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 291K, capacity 386K, committed 512K, reserved 1048576K

执行结束!共生成对象次数:8885

```
* 设置初始最小和最大对大小一样时，
 >共生成对象次数:12035,性能对没有设置初始化最小堆时提升39%，解读：Xms设置与-Xmx相同，以避免每次垃圾回收完成后JVM重新分配内存
``` Java
D:\F盘\JavaCourseCodes\01jvm>java  -Xms4g -Xmx4g  -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis
正在执行...
2021-02-24T06:48:52.516+0800: [GC (Allocation Failure) [PSYoungGen: 1048576K->174589K(1223168K)] 1048576K->235252K(4019712K), 0.0441413 secs] [Times: user=0.02 sys=0.20, real=0.04 secs]
2021-02-24T06:48:52.718+0800: [GC (Allocation Failure) [PSYoungGen: 1223165K->174591K(1223168K)] 1283828K->357504K(4019712K), 0.1024186 secs] [Times: user=0.23 sys=0.52, real=0.10 secs]
2021-02-24T06:48:52.978+0800: [GC (Allocation Failure) [PSYoungGen: 1223167K->174577K(1223168K)] 1406080K->473105K(4019712K), 0.4661695 secs] [Times: user=0.27 sys=2.50, real=0.47 secs]
执行结束!共生成对象次数:12035
Heap
 PSYoungGen      total 1223168K, used 216650K [0x000000076ab00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 1048576K, 4% used [0x000000076ab00000,0x000000076d416238,0x00000007aab00000)
  from space 174592K, 99% used [0x00000007aab00000,0x00000007b557c750,0x00000007b5580000)
  to   space 174592K, 0% used [0x00000007b5580000,0x00000007b5580000,0x00000007c0000000)
 ParOldGen       total 2796544K, used 298527K [0x00000006c0000000, 0x000000076ab00000, 0x000000076ab00000)
  object space 2796544K, 10% used [0x00000006c0000000,0x00000006d2387f40,0x000000076ab00000)
 Metaspace       used 2634K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 291K, capacity 386K, committed 512K, reserved 1048576K

D:\F盘\JavaCourseCodes\01jvm>
```

* 图解说明并行GC的内存变化
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210223230157116.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)
>每次youngGC，young区空间减少很多，Eden和S0全部清空，部分对象**复制**到S1区，old区越来越大，然后进行fullGC，下一次youngGC会把Eden和S1清空 ，
>>延伸问题：年轻代和老年代的垃圾回收都会触发STW 事件。在年轻代使用标记-复制（mark-copy）算法，在老年代使用标记-清除-整理（mark-sweepcompact）算法。-XX：ParallelGCThreads=N 来指定GC 线程数， 其默认值为CPU 核心数。
>
>fullGC，会把Eden ，S0，S1区全部清空
>默认yungGC次数15次，超过15次还存活的对象采用**移动方式**晋升到老年代区，老年代能减少些但不多

## 串行GC演示（ -XX:+UseSerialGC）
* 当前最大堆内存512M
``` Java
D:\F盘\JavaCourseCodes\01jvm>java -XX:+UseSerialGC -Xms512m -Xmx512m  -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis
正在执行...
2021-02-23T22:35:13.453+0800: [GC (Allocation Failure) 2021-02-23T22:35:13.453+0800: [DefNew: 139776K->17472K(157248K), 0.0324775 secs] 139776K->48832K(506816K), 0.0330806 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
2021-02-23T22:35:13.521+0800: [GC (Allocation Failure) 2021-02-23T22:35:13.521+0800: [DefNew: 157127K->17471K(157248K), 0.0427665 secs] 188488K->98605K(506816K), 0.0434445 secs] [Times: user=0.03 sys=0.00, real=0.04 secs]
2021-02-23T22:35:13.607+0800: [GC (Allocation Failure) 2021-02-23T22:35:13.608+0800: [DefNew: 157247K->17471K(157248K), 0.0323721 secs] 238381K->140525K(506816K), 0.0331904 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
2021-02-23T22:35:13.682+0800: [GC (Allocation Failure) 2021-02-23T22:35:13.682+0800: [DefNew: 157247K->17471K(157248K), 0.0337171 secs] 280301K->184841K(506816K), 0.0344639 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]
2021-02-23T22:35:13.757+0800: [GC (Allocation Failure) 2021-02-23T22:35:13.758+0800: [DefNew: 156573K->17472K(157248K), 0.0318421 secs] 323942K->226784K(506816K), 0.0327063 secs] [Times: user=0.01 sys=0.02, real=0.03 secs]
2021-02-23T22:35:13.831+0800: [GC (Allocation Failure) 2021-02-23T22:35:13.832+0800: [DefNew: 157248K->17471K(157248K), 0.0357931 secs] 366560K->273563K(506816K), 0.0364656 secs] [Times: user=0.02 sys=0.02, real=0.04 secs]
2021-02-23T22:35:13.909+0800: [GC (Allocation Failure) 2021-02-23T22:35:13.910+0800: [DefNew: 157098K->17471K(157248K), 0.0321501 secs] 413190K->312630K(506816K), 0.0330849 secs] [Times: user=0.02 sys=0.02, real=0.03 secs]
2021-02-23T22:35:13.981+0800: [GC (Allocation Failure) 2021-02-23T22:35:13.981+0800: [DefNew: 157247K->17470K(157248K), 0.0342253 secs] 452406K->354246K(506816K), 0.0349803 secs] [Times: user=0.02 sys=0.02, real=0.04 secs]
2021-02-23T22:35:14.053+0800: [GC (Allocation Failure) 2021-02-23T22:35:14.054+0800: [DefNew: 157246K->157246K(157248K), 0.0002210 secs]2021-02-23T22:35:14.054+0800: [Tenured: 336776K->273961K(349568K), 0.0635833 secs] 494022K->273961K(506816K), [Metaspace: 2628K->2628K(1056768K)], 0.0645362 secs] [Times: user=0.06 sys=0.00, real=0.07 secs]
2021-02-23T22:35:14.155+0800: [GC (Allocation Failure) 2021-02-23T22:35:14.155+0800: [DefNew: 139776K->17471K(157248K), 0.0137345 secs] 413737K->324486K(506816K), 0.0144490 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
2021-02-23T22:35:14.207+0800: [GC (Allocation Failure) 2021-02-23T22:35:14.207+0800: [DefNew: 157247K->157247K(157248K), 0.0002225 secs]2021-02-23T22:35:14.207+0800: [Tenured: 307014K->301297K(349568K), 0.0715632 secs] 464262K->301297K(506816K), [Metaspace: 2628K->2628K(1056768K)], 0.0724522 secs] [Times: user=0.06 sys=0.00, real=0.07 secs]
执行结束!共生成对象次数:6288
Heap
 def new generation   total 157248K, used 126441K [0x00000000e0000000, 0x00000000eaaa0000, 0x00000000eaaa0000)
  eden space 139776K,  90% used [0x00000000e0000000, 0x00000000e7b7a608, 0x00000000e8880000)
  from space 17472K,   0% used [0x00000000e9990000, 0x00000000e9990000, 0x00000000eaaa0000)
  to   space 17472K,   0% used [0x00000000e8880000, 0x00000000e8880000, 0x00000000e9990000)
 tenured generation   total 349568K, used 301297K [0x00000000eaaa0000, 0x0000000100000000, 0x0000000100000000)
   the space 349568K,  86% used [0x00000000eaaa0000, 0x00000000fd0dc620, 0x00000000fd0dc800, 0x0000000100000000)
 Metaspace       used 2634K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 291K, capacity 386K, committed 512K, reserved 1048576K

D:\F盘\JavaCourseCodes\01jvm>
```
* 当前最大堆内存128M时，出现内存溢出异常了
> fullGC对垃圾回收几乎没有作用，垃圾一直占的内存空间
``` Java
D:\F盘\JavaCourseCodes\01jvm>java -XX:+UseSerialGC -Xms128m -Xmx128m  -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis
正在执行...
2021-02-23T22:42:07.153+0800: [GC (Allocation Failure) 2021-02-23T22:42:07.154+0800: [DefNew: 34915K->4351K(39296K), 0.0105513 secs] 34915K->12378K(126720K), 0.0115875 secs] [Times: user=0.02 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.182+0800: [GC (Allocation Failure) 2021-02-23T22:42:07.182+0800: [DefNew: 39197K->4349K(39296K), 0.0123236 secs] 47223K->22965K(126720K), 0.0138073 secs] [Times: user=0.00 sys=0.02, real=0.01 secs]
2021-02-23T22:42:07.213+0800: [GC (Allocation Failure) 2021-02-23T22:42:07.214+0800: [DefNew: 38618K->4343K(39296K), 0.0091259 secs] 57234K->33897K(126720K), 0.0100570 secs] [Times: user=0.01 sys=0.02, real=0.01 secs]
2021-02-23T22:42:07.237+0800: [GC (Allocation Failure) 2021-02-23T22:42:07.238+0800: [DefNew: 39146K->4351K(39296K), 0.0104840 secs] 68701K->45751K(126720K), 0.0111278 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.265+0800: [GC (Allocation Failure) 2021-02-23T22:42:07.266+0800: [DefNew: 38929K->4338K(39296K), 0.0086908 secs] 80329K->55306K(126720K), 0.0096184 secs] [Times: user=0.02 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.288+0800: [GC (Allocation Failure) 2021-02-23T22:42:07.288+0800: [DefNew: 39269K->4348K(39296K), 0.0137101 secs] 90238K->68836K(126720K), 0.0144757 secs] [Times: user=0.00 sys=0.02, real=0.01 secs]
2021-02-23T22:42:07.313+0800: [GC (Allocation Failure) 2021-02-23T22:42:07.313+0800: [DefNew: 39292K->4349K(39296K), 0.0142091 secs] 103780K->83540K(126720K), 0.0150465 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
2021-02-23T22:42:07.344+0800: [GC (Allocation Failure) 2021-02-23T22:42:07.350+0800: [DefNew: 39008K->39008K(39296K), 0.0011520 secs]2021-02-23T22:42:07.352+0800: [Tenured: 79190K->87419K(87424K), 0.0237885 secs] 118199K->90743K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0361671 secs] [Times: user=0.02 sys=0.00, real=0.04 secs]
2021-02-23T22:42:07.394+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.396+0800: [Tenured: 87419K->87397K(87424K), 0.0191946 secs] 126695K->99685K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0212197 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
2021-02-23T22:42:07.429+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.429+0800: [Tenured: 87397K->87232K(87424K), 0.0221254 secs] 126683K->106532K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0232227 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
2021-02-23T22:42:07.462+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.463+0800: [Tenured: 87304K->87268K(87424K), 0.0306981 secs] 126540K->108960K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0328040 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
2021-02-23T22:42:07.504+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.506+0800: [Tenured: 87268K->87268K(87424K), 0.0079315 secs] 126362K->113405K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0106992 secs] [Times: user=0.00 sys=0.02, real=0.01 secs]
2021-02-23T22:42:07.520+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.529+0800: [Tenured: 87268K->87268K(87424K), 0.0050639 secs] 126526K->116875K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0144243 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
2021-02-23T22:42:07.541+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.550+0800: [Tenured: 87268K->87398K(87424K), 0.0116193 secs] 126410K->118705K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0215769 secs] [Times: user=0.00 sys=0.00, real=0.02 secs]
2021-02-23T22:42:07.569+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.574+0800: [Tenured: 87398K->87249K(87424K), 0.0315818 secs] 126427K->116175K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0369570 secs] [Times: user=0.03 sys=0.00, real=0.04 secs]
2021-02-23T22:42:07.613+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.616+0800: [Tenured: 87249K->87249K(87424K), 0.0091864 secs] 126101K->119359K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0126861 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.634+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.636+0800: [Tenured: 87249K->87249K(87424K), 0.0047839 secs] 126005K->121203K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0077264 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.651+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.651+0800: [Tenured: 87249K->87249K(87424K), 0.0049760 secs] 126470K->123275K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0061458 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.669+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.680+0800: [Tenured: 87357K->87423K(87424K), 0.0342295 secs] 126618K->121943K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0459635 secs] [Times: user=0.03 sys=0.00, real=0.05 secs]
2021-02-23T22:42:07.717+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.717+0800: [Tenured: 87423K->87423K(87424K), 0.0109540 secs] 126119K->123382K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0122313 secs] [Times: user=0.02 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.737+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.742+0800: [Tenured: 87423K->87423K(87424K), 0.0046415 secs] 126694K->124383K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0101329 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.753+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.768+0800: [Tenured: 87423K->87423K(87424K), 0.0114733 secs] 126437K->124440K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0268546 secs] [Times: user=0.00 sys=0.00, real=0.03 secs]
2021-02-23T22:42:07.786+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.794+0800: [Tenured: 87423K->87362K(87424K), 0.0319215 secs] 126709K->124372K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0403032 secs] [Times: user=0.03 sys=0.00, real=0.04 secs]
2021-02-23T22:42:07.831+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.832+0800: [Tenured: 87362K->87362K(87424K), 0.0037068 secs] 126024K->125099K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0073521 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.840+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.842+0800: [Tenured: 87362K->87362K(87424K), 0.0046366 secs] 125758K->125758K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0095181 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-23T22:42:07.851+0800: [Full GC (Allocation Failure) 2021-02-23T22:42:07.866+0800: [Tenured: 87362K->87279K(87424K), 0.0246226 secs] 125758K->125675K(126720K), [Metaspace: 2628K->2628K(1056768K)], 0.0399895 secs] [Times: user=0.01 sys=0.00, real=0.04 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        at GCLogAnalysis.generateGarbage(GCLogAnalysis.java:48)
        at GCLogAnalysis.main(GCLogAnalysis.java:25)
Heap
 def new generation   total 39296K, used 38845K [0x00000000f8000000, 0x00000000faaa0000, 0x00000000faaa0000)
  eden space 34944K, 100% used [0x00000000f8000000, 0x00000000fa220000, 0x00000000fa220000)
  from space 4352K,  89% used [0x00000000fa660000, 0x00000000faa2f408, 0x00000000faaa0000)
  to   space 4352K,   0% used [0x00000000fa220000, 0x00000000fa220000, 0x00000000fa660000)
 tenured generation   total 87424K, used 87279K [0x00000000faaa0000, 0x0000000100000000, 0x0000000100000000)
   the space 87424K,  99% used [0x00000000faaa0000, 0x00000000fffdbe90, 0x00000000fffdc000, 0x0000000100000000)
 Metaspace       used 2658K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 294K, capacity 386K, committed 512K, reserved 1048576K

D:\F盘\JavaCourseCodes\01jvm>
```

* 当前最大堆内存2048M时，
 >此时GC次数只有两次了。但是GC耗时很久，对比设置最大堆内存512M时产生的对象减少了20%，也就是说性能反而较少，Xmx不是越大越好
``` Java
D:\F盘\JavaCourseCodes\01jvm>java -XX:+UseSerialGC -Xms2048m -Xmx2048m  -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis
正在执行...
2021-02-23T22:47:56.035+0800: [GC (Allocation Failure) 2021-02-23T22:47:56.035+0800: [DefNew: 559232K->69888K(629120K), 0.1193025 secs] 559232K->157347K(2027264K), 0.1198591 secs] [Times: user=0.06 sys=0.05, real=0.12 secs]
2021-02-23T22:47:56.304+0800: [GC (Allocation Failure) 2021-02-23T22:47:56.304+0800: [DefNew: 629120K->69888K(629120K), 0.1559310 secs] 716579K->289236K(2027264K), 0.1568711 secs] [Times: user=0.09 sys=0.06, real=0.16 secs]
执行结束!共生成对象次数:4917
Heap
 def new generation   total 629120K, used 291713K [0x0000000080000000, 0x00000000aaaa0000, 0x00000000aaaa0000)
  eden space 559232K,  39% used [0x0000000080000000, 0x000000008d8a07a8, 0x00000000a2220000)
  from space 69888K, 100% used [0x00000000a2220000, 0x00000000a6660000, 0x00000000a6660000)
  to   space 69888K,   0% used [0x00000000a6660000, 0x00000000a6660000, 0x00000000aaaa0000)
 tenured generation   total 1398144K, used 219348K [0x00000000aaaa0000, 0x0000000100000000, 0x0000000100000000)
   the space 1398144K,  15% used [0x00000000aaaa0000, 0x00000000b80d53e8, 0x00000000b80d5400, 0x0000000100000000)
 Metaspace       used 2634K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 291K, capacity 386K, committed 512K, reserved 1048576K

D:\F盘\JavaCourseCodes\01jvm>
```

## CMSGC演示（ -XX:+UseConcMarkSweepGC）
* 当前最大堆内存512M
> 当发生5五次youngGC后，就进行CMSGC，
>> [GC (CMS Initial Mark) [1 CMS-initial-mark: 207528K(349568K)] 225479K(506816K), 0.0009352 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] Initial Mark(初始标记)：当前步骤需要JVM暂停STW，标记的根对象GC root
>> [CMS-concurrent-mark-start]   Concurrent Mark(并发标记)
>> [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]   Concurrent Preclean(并发预清理)
>> [CMS-concurrent-preclean-start]  Final Remark(最终标记)
>> [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]  Concurrent Sweep(并发清除)
>> [CMS-concurrent-abortable-preclean-start]   Concurrent Reset(并发重置)
>>一共6个步骤，5个步骤是并发执行,CMS 使用的并发线程数等于CPU 核心数的1/4
>
>CMS GC 的设计目标是避免在老年代垃圾收集时出现长时间的卡顿
``` Java
D:\F盘\JavaCourseCodes\01jvm>java -XX:+UseConcMarkSweepGC  -Xms512m -Xmx512m  -XX:+PrintGCDetails -XX:+PrintGCDateStamps GCLogAnalysis
正在执行...
2021-02-24T07:02:07.839+0800: [GC (Allocation Failure) 2021-02-24T07:02:07.840+0800: [ParNew: 139776K->17468K(157248K), 0.0079185 secs] 139776K->44649K(506816K), 0.0085489 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-24T07:02:07.879+0800: [GC (Allocation Failure) 2021-02-24T07:02:07.879+0800: [ParNew: 157199K->17459K(157248K), 0.0130003 secs] 184380K->97930K(506816K), 0.0133819 secs] [Times: user=0.03 sys=0.09, real=0.01 secs]
2021-02-24T07:02:07.923+0800: [GC (Allocation Failure) 2021-02-24T07:02:07.923+0800: [ParNew: 157235K->17472K(157248K), 0.0219433 secs] 237706K->138548K(506816K), 0.0223873 secs] [Times: user=0.08 sys=0.02, real=0.02 secs]
2021-02-24T07:02:07.972+0800: [GC (Allocation Failure) 2021-02-24T07:02:07.972+0800: [ParNew: 157248K->17472K(157248K), 0.0225044 secs] 278324K->181476K(506816K), 0.0231603 secs] [Times: user=0.09 sys=0.02, real=0.02 secs]
2021-02-24T07:02:08.023+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.024+0800: [ParNew: 156802K->17471K(157248K), 0.0235213 secs] 320806K->225000K(506816K), 0.0239944 secs] [Times: user=0.17 sys=0.03, real=0.02 secs]
2021-02-24T07:02:08.049+0800: [GC (CMS Initial Mark) [1 CMS-initial-mark: 207528K(349568K)] 225479K(506816K), 0.0009352 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.050+0800: [CMS-concurrent-mark-start]
2021-02-24T07:02:08.052+0800: [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.053+0800: [CMS-concurrent-preclean-start]
2021-02-24T07:02:08.055+0800: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.055+0800: [CMS-concurrent-abortable-preclean-start]
2021-02-24T07:02:08.082+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.083+0800: [ParNew: 157247K->17471K(157248K), 0.0207909 secs] 364776K->262029K(506816K), 0.0213737 secs] [Times: user=0.09 sys=0.00, real=0.02 secs]
2021-02-24T07:02:08.130+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.131+0800: [ParNew: 157247K->17470K(157248K), 0.0221870 secs] 401805K->300663K(506816K), 0.0227283 secs] [Times: user=0.17 sys=0.02, real=0.02 secs]
2021-02-24T07:02:08.176+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.176+0800: [ParNew: 157246K->17471K(157248K), 0.0268265 secs] 440439K->348831K(506816K), 0.0273241 secs] [Times: user=0.17 sys=0.01, real=0.03 secs]
2021-02-24T07:02:08.205+0800: [CMS-concurrent-abortable-preclean: 0.006/0.146 secs] [Times: user=0.51 sys=0.03, real=0.15 secs]
2021-02-24T07:02:08.206+0800: [GC (CMS Final Remark) [YG occupancy: 25957 K (157248 K)]2021-02-24T07:02:08.207+0800: [Rescan (parallel) , 0.0004543 secs]2021-02-24T07:02:08.207+0800: [weak refs processing, 0.0000867 secs]2021-02-24T07:02:08.207+0800: [class unloading, 0.0008495 secs]2021-02-24T07:02:08.208+0800: [scrub symbol table, 0.0004637 secs]2021-02-24T07:02:08.209+0800: [scrub string table, 0.0002037 secs][1 CMS-remark: 331359K(349568K)] 357316K(506816K), 0.0035973 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.210+0800: [CMS-concurrent-sweep-start]
2021-02-24T07:02:08.211+0800: [CMS-concurrent-sweep: 0.001/0.001 secs] [Times: user=0.02 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.214+0800: [CMS-concurrent-reset-start]
2021-02-24T07:02:08.222+0800: [CMS-concurrent-reset: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.234+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.234+0800: [ParNew: 157247K->17471K(157248K), 0.0133071 secs] 444418K->351340K(506816K), 0.0137356 secs] [Times: user=0.09 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.249+0800: [GC (CMS Initial Mark) [1 CMS-initial-mark: 333869K(349568K)] 351485K(506816K), 0.0008960 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.250+0800: [CMS-concurrent-mark-start]
2021-02-24T07:02:08.253+0800: [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.260+0800: [CMS-concurrent-preclean-start]
2021-02-24T07:02:08.262+0800: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.02 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.263+0800: [CMS-concurrent-abortable-preclean-start]
2021-02-24T07:02:08.263+0800: [CMS-concurrent-abortable-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.265+0800: [GC (CMS Final Remark) [YG occupancy: 79605 K (157248 K)]2021-02-24T07:02:08.265+0800: [Rescan (parallel) , 0.0004118 secs]2021-02-24T07:02:08.266+0800: [weak refs processing, 0.0001727 secs]2021-02-24T07:02:08.266+0800: [class unloading, 0.0011286 secs]2021-02-24T07:02:08.267+0800: [scrub symbol table, 0.0004355 secs]2021-02-24T07:02:08.268+0800: [scrub string table, 0.0001893 secs][1 CMS-remark: 333869K(349568K)] 413475K(506816K), 0.0038621 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.275+0800: [CMS-concurrent-sweep-start]
2021-02-24T07:02:08.278+0800: [CMS-concurrent-sweep: 0.001/0.001 secs] [Times: user=0.03 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.278+0800: [CMS-concurrent-reset-start]
2021-02-24T07:02:08.280+0800: [CMS-concurrent-reset: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.292+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.292+0800: [ParNew: 157112K->17470K(157248K), 0.0115372 secs] 387069K->288883K(506816K), 0.0123590 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.306+0800: [GC (CMS Initial Mark) [1 CMS-initial-mark: 271412K(349568K)] 289062K(506816K), 0.0008636 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.307+0800: [CMS-concurrent-mark-start]
2021-02-24T07:02:08.309+0800: [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.05 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.310+0800: [CMS-concurrent-preclean-start]
2021-02-24T07:02:08.311+0800: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.312+0800: [CMS-concurrent-abortable-preclean-start]
2021-02-24T07:02:08.334+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.335+0800: [ParNew: 156988K->17471K(157248K), 0.0125048 secs] 428400K->327382K(506816K), 0.0129594 secs] [Times: user=0.11 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.375+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.375+0800: [ParNew: 157199K->157199K(157248K), 0.0001566 secs]2021-02-24T07:02:08.376+0800: [CMS2021-02-24T07:02:08.376+0800: [CMS-concurrent-abortable-preclean: 0.003/0.060 secs] [Times: user=0.16 sys=0.00, real=0.06 secs]
 (concurrent mode failure): 309910K->289393K(349568K), 0.0562658 secs] 467109K->289393K(506816K), [Metaspace: 2628K->2628K(1056768K)], 0.0569689 secs] [Times: user=0.05 sys=0.00, real=0.06 secs]
2021-02-24T07:02:08.459+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.459+0800: [ParNew: 139776K->17472K(157248K), 0.0088130 secs] 429169K->336093K(506816K), 0.0093098 secs] [Times: user=0.09 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.470+0800: [GC (CMS Initial Mark) [1 CMS-initial-mark: 318621K(349568K)] 336237K(506816K), 0.0009292 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.471+0800: [CMS-concurrent-mark-start]
2021-02-24T07:02:08.475+0800: [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.475+0800: [CMS-concurrent-preclean-start]
2021-02-24T07:02:08.477+0800: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.479+0800: [CMS-concurrent-abortable-preclean-start]
2021-02-24T07:02:08.486+0800: [CMS-concurrent-abortable-preclean: 0.000/0.000 secs] [Times: user=0.02 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.488+0800: [GC (CMS Final Remark) [YG occupancy: 88636 K (157248 K)]2021-02-24T07:02:08.491+0800: [Rescan (parallel) , 0.0004964 secs]2021-02-24T07:02:08.492+0800: [weak refs processing, 0.0001096 secs]2021-02-24T07:02:08.492+0800: [class unloading, 0.0059392 secs]2021-02-24T07:02:08.498+0800: [scrub symbol table, 0.0006367 secs]2021-02-24T07:02:08.499+0800: [scrub string table, 0.0002811 secs][1 CMS-remark: 318621K(349568K)] 407257K(506816K), 0.0135319 secs] [Times: user=0.00 sys=0.00, real=0.02 secs]
2021-02-24T07:02:08.503+0800: [CMS-concurrent-sweep-start]
2021-02-24T07:02:08.516+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.516+0800: [ParNew: 157248K->17471K(157248K), 0.0143191 secs] 443039K->350400K(506816K), 0.0145845 secs] [Times: user=0.08 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.530+0800: [CMS-concurrent-sweep: 0.001/0.016 secs] [Times: user=0.09 sys=0.00, real=0.04 secs]
2021-02-24T07:02:08.540+0800: [CMS-concurrent-reset-start]
2021-02-24T07:02:08.547+0800: [CMS-concurrent-reset: 0.000/0.000 secs] [Times: user=0.02 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.560+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.561+0800: [ParNew: 157247K->157247K(157248K), 0.0001587 secs]2021-02-24T07:02:08.561+0800: [CMS: 328412K->312126K(349568K), 0.0522486 secs] 485660K->312126K(506816K), [Metaspace: 2628K->2628K(1056768K)], 0.0537724 secs] [Times: user=0.05 sys=0.00, real=0.05 secs]
2021-02-24T07:02:08.614+0800: [GC (CMS Initial Mark) [1 CMS-initial-mark: 312126K(349568K)] 312630K(506816K), 0.0007227 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.615+0800: [CMS-concurrent-mark-start]
2021-02-24T07:02:08.618+0800: [CMS-concurrent-mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.618+0800: [CMS-concurrent-preclean-start]
2021-02-24T07:02:08.620+0800: [CMS-concurrent-preclean: 0.001/0.001 secs] [Times: user=0.03 sys=0.00, real=0.00 secs]
2021-02-24T07:02:08.620+0800: [CMS-concurrent-abortable-preclean-start]
2021-02-24T07:02:08.625+0800: [CMS-concurrent-abortable-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.631+0800: [GC (CMS Final Remark) [YG occupancy: 88449 K (157248 K)]2021-02-24T07:02:08.632+0800: [Rescan (parallel) , 0.0004337 secs]2021-02-24T07:02:08.632+0800: [weak refs processing, 0.0014522 secs]2021-02-24T07:02:08.634+0800: [class unloading, 0.0100824 secs]2021-02-24T07:02:08.644+0800: [scrub symbol table, 0.0004037 secs]2021-02-24T07:02:08.645+0800: [scrub string table, 0.0002123 secs][1 CMS-remark: 312126K(349568K)] 400576K(506816K), 0.0142744 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
2021-02-24T07:02:08.646+0800: [CMS-concurrent-sweep-start]
2021-02-24T07:02:08.656+0800: [GC (Allocation Failure) 2021-02-24T07:02:08.663+0800: [ParNew: 139566K->139566K(157248K), 0.0001837 secs]2021-02-24T07:02:08.663+0800: [CMS2021-02-24T07:02:08.663+0800: [CMS-concurrent-sweep: 0.001/0.008 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
 (concurrent mode failure): 311478K->323317K(349568K), 0.0660418 secs] 451044K->323317K(506816K), [Metaspace: 2628K->2628K(1056768K)], 0.0729739 secs] [Times: user=0.05 sys=0.00, real=0.07 secs]
执行结束!共生成对象次数:8831
Heap
 par new generation   total 157248K, used 111872K [0x00000000e0000000, 0x00000000eaaa0000, 0x00000000eaaa0000)
  eden space 139776K,  80% used [0x00000000e0000000, 0x00000000e6d400b8, 0x00000000e8880000)
  from space 17472K,   0% used [0x00000000e9990000, 0x00000000e9990000, 0x00000000eaaa0000)
  to   space 17472K,   0% used [0x00000000e8880000, 0x00000000e8880000, 0x00000000e9990000)
 concurrent mark-sweep generation total 349568K, used 323317K [0x00000000eaaa0000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 2634K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 291K, capacity 386K, committed 512K, reserved 1048576K

D:\F盘\JavaCourseCodes\01jvm>
```

* 图解GC变化
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021022407141120.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)


## G1GC演示（-XX:+UseG1GC）
* 当前最大堆内存512M
> 当发生5五次youngGC后，就进行CMSGC，
>> 阶段1: Initial Mark(初始标记)
   阶段2: Root Region Scan(Root区扫描)
   阶段3:Evacuation Pause (mixed)(转移暂停: 混合模式)
   阶段4: Remark(再次标记)
   阶段5: Cleanup(清理)
``` Java

D:\F盘\JavaCourseCodes\01jvm>java -XX:+UseG1GC  -Xms512m -Xmx512m  -XX:+PrintGC -XX:+PrintGCDateStamps GCLogAnalysis
正在执行...
2021-02-24T07:28:03.140+0800: [GC pause (G1 Evacuation Pause) (young) 32M->11M(512M), 0.0028508 secs]       //Evacuation Pause: young(纯年轻代模式转移暂停)
2021-02-24T07:28:03.153+0800: [GC pause (G1 Evacuation Pause) (young) 39M->20M(512M), 0.0027907 secs]
2021-02-24T07:28:03.178+0800: [GC pause (G1 Evacuation Pause) (young) 67M->37M(512M), 0.0042861 secs]
2021-02-24T07:28:03.208+0800: [GC pause (G1 Evacuation Pause) (young) 102M->60M(512M), 0.0052239 secs]
2021-02-24T07:28:03.276+0800: [GC pause (G1 Evacuation Pause) (young) 214M->108M(512M), 0.0072020 secs]
2021-02-24T07:28:03.316+0800: [GC pause (G1 Evacuation Pause) (young) 242M->149M(512M), 0.0077214 secs]
2021-02-24T07:28:03.383+0800: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 342M->202M(512M), 0.0342600 secs]   //Concurrent Marking(并发标记)
2021-02-24T07:28:03.417+0800: [GC concurrent-root-region-scan-start]
2021-02-24T07:28:03.419+0800: [GC concurrent-root-region-scan-end, 0.0012973 secs]
2021-02-24T07:28:03.419+0800: [GC concurrent-mark-start]
2021-02-24T07:28:03.422+0800: [GC concurrent-mark-end, 0.0024423 secs]
2021-02-24T07:28:03.422+0800: [GC remark, 0.0017725 secs]
2021-02-24T07:28:03.424+0800: [GC cleanup 228M->228M(512M), 0.0012586 secs]
     //阶段1: Initial Mark(初始标记)
     //阶段2: Root Region Scan(Root区扫描)
     //阶段3:Evacuation Pause (mixed)(转移暂停: 混合模式)
     //阶段4: Remark(再次标记)
     //阶段5: Cleanup(清理)
2021-02-24T07:28:03.471+0800: [GC pause (G1 Evacuation Pause) (young)-- 406M->307M(512M), 0.0083992 secs]
2021-02-24T07:28:03.480+0800: [GC pause (G1 Evacuation Pause) (mixed) 310M->299M(512M), 0.0048689 secs]             //Evacuation Pause (mixed)(转移暂停: 混合模式)
2021-02-24T07:28:03.487+0800: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 303M->300M(512M), 0.0011917 secs]
2021-02-24T07:28:03.489+0800: [GC concurrent-root-region-scan-start]
2021-02-24T07:28:03.497+0800: [GC concurrent-root-region-scan-end, 0.0085547 secs]
2021-02-24T07:28:03.504+0800: [GC concurrent-mark-start]
2021-02-24T07:28:03.507+0800: [GC concurrent-mark-end, 0.0026832 secs]
2021-02-24T07:28:03.508+0800: [GC remark, 0.0113277 secs]
2021-02-24T07:28:03.520+0800: [GC cleanup 383M->383M(512M), 0.0010538 secs]
2021-02-24T07:28:03.529+0800: [GC pause (G1 Evacuation Pause) (young) 412M->332M(512M), 0.0084896 secs]
2021-02-24T07:28:03.541+0800: [GC pause (G1 Evacuation Pause) (mixed) 349M->291M(512M), 0.0047697 secs]
2021-02-24T07:28:03.552+0800: [GC pause (G1 Evacuation Pause) (mixed) 316M->278M(512M), 0.0044825 secs]
2021-02-24T07:28:03.558+0800: [GC pause (G1 Humongous Allocation) (young) (initial-mark) 283M->279M(512M), 0.0016376 secs]
2021-02-24T07:28:03.560+0800: [GC concurrent-root-region-scan-start]
2021-02-24T07:28:03.569+0800: [GC concurrent-root-region-scan-end, 0.0091800 secs]
```

* 图解GC变化
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210224072848300.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM3ODY5MjQz,size_16,color_FFFFFF,t_70)
 * 延伸知识
>通过划分多个内存区域做增量整理和回收，进一步降低延迟
>G1 的全称是Garbage-First，意为垃圾优先，哪一块的垃圾最多就优先清理它
>G1 GC 最主要的设计目标是：将STW 停顿的时间和分布，变成可预期且可配置的
>堆不再分成年轻代和老年代，而是划分为多个（通常是2048 个）可以存放对象的小块堆区域(smaller heap regions)。每个小块，可能一会被定义成Eden 区，一会被指定为Survivor区或者Old 区。在逻辑上，所有的Eden 区和Survivor区合起来就是年轻代，所有的Old 区拼在一起那就是老年