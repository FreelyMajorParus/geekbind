[toc]

## 4.（必做）根据上述自己对于 1 和 2 的演示，写一段对于不同 GC 和堆内存的总结，提交到 GitHub。

串行GC ：包括Serial New和Serial Old，分别用于新生代和老年代的回收，在单核并且内存资源比较紧凑的环境下，非常适合垃圾回收，因为单核环境下，由单个线程来执行垃圾回收，可以避免多线程上下文切换带来的性能开销，并且，在内存资源较紧凑的情况下，即使是单线程执行垃圾回收，上百毫秒通常不会被一些客户端应用程序感知到。

并行GC : 以ParNew（适用于新生代收集）, Parallel Scavenge(适用于新生代收集)，Parallel Scavenge（适用于老年代收集）为代表，在多核的环境下，单线程收集已经达不到目标吞吐量，而且在内存资源较大时，单线程回收导致GC停顿时间更长。这个时候多线程收集就能大幅度提升吞吐量了，而且多线程收集带来的GC停顿时间更短。像Web应用的服务端对应的新生代一般使用并行GC，首先新生代的空间比老年代小，且多核环境下，新生代的并行GC也更快一些。

CMS GC : 当并行GC可以达到更高的吞吐量时，我们将目标转移至GC的停顿时间， 因为对于Web服务端来说，GC停顿时间较长相当于更高的吞吐量而言，更加重要。这个时候CMS从GC停顿时间出发，将老年代的回收过程打散了，因为并行收集老年代时，收集的整个过程还可以细分成多个阶段。就像多线程执行同步代码块，也许真正同步的只有部分代码行，其他代码行是可以并行执行的。通过将老年代的GC过程分成: 初始标记、并发标记、并发预清理、最终标记、清除等步骤，其中初始标记以及最终标记才是需要STW的步骤，而其他大部分阶段占用的时间，都是可以和应用线程并行执行的。也就是说STW只存在于小阶段中，这样就可以大幅减少STW的时间，从而做到牺牲少部分吞吐量达到更低的GC停顿。

G1 GC : G1作为CMS的升级版本，它的目的是专注于可预测的GC暂停，也就是将GC的暂停时间做到可控制的范围内。如果像新生代和老年代一样将内存划分成2大块，在GC收集时，要么收集其中的新生代，要么收集老年代，而这两部分的任意一部分回收时间，相对于一些高性能Web服务端而言可容忍的50-100毫秒，都是比较长的。G1提出了颠覆之前内存划分策略，将整块内存划分成大小相同的很多块，这些小块可以是老年代，也可以是新生代。通过这种方式，就很方便知道哪块分配了多少对象，而收集哪块可以达到最好的回收，最终就可以根据用户配置的预期GC时间来决定具体要回收哪些块(Region)。

ZGC & ShennandoahGC: 由于G1和CMS最大的差异是在堆空间划分上，所以当我们有更大的内存资源时，我们依然要考虑在CMS或者G1在回收时，同步进行的代码块该如何优化，而像ZGC等新的垃圾收集器就更加关注于GC如何与应用线程达到最大化的并发执行，以更低的GC停顿时间，达到更高的吞吐量，更高效的垃圾收集。

堆内存: JVM在最初将堆内存分为新生代和老年代是因为，新生代的对象存活率较低，所以新生代使用复制算法转移少量的对象可以达到更加高效的垃圾收集，而老年代由于大部分的对象存活率都比较久，使用复制算法会涉及到大部分对象的拷贝，所以使用标记-清除算法。CMS之后，为了更加高效的垃圾收集，将堆空间的划分的更加小粒度，这种方式有助于可预期的回收, 但是CMS垃圾收集不涉及到内存整理，也就是还会产生内存碎片。在G1中，因为基于本身的垃圾回收算法，堆内存被打散成了很多个小块，在垃圾不断回收的过程中，也类似内存碎片不断生成，但是G1垃圾收集为堆空间分配了“巨型”对象专用的空间，这样也避免一些内存碎片带来的问题。

## 6.（必做）写一段代码，使用 HttpClient 或 OkHttp 访问  http://localhost:8801 ，代码提交到 GitHub.

HttpClient(引入的版本: )示例如下: 
```java
public class HttpClientTest {
    static CloseableHttpClient httpClient = HttpClientBuilder.create()
            .setConnectionTimeToLive(5, TimeUnit.SECONDS).build();
 
    public static void main(String[] args) throws Exception {
        String result = new HttpClientTest().sendGet("http://localhost:8088/api/hello");
        System.out.println(result);
    }
 
    private String sendGet(String address) {
        HttpGet httpGet = new HttpGet(address);
        try (CloseableHttpResponse response = httpClient.execute(httpGet)){
            HttpEntity entity = response.getEntity();
            InputStream content = entity.getContent();
            int bytesAvailable = content.available();
            byte[] bytes = new byte[bytesAvailable];
            content.read(bytes, 0, bytesAvailable);
            return  JSONObject.toJSONString(new String(bytes));
        } catch (Exception ignore) {
        }
        return "empty";
    }
}
```
OkHttp 示例如下:
```java
public class OkHttpClientTest {
    public static void main(String[] args) throws IOException {
        OkHttpClient client = new OkHttpClient.Builder()
                .connectTimeout(5, TimeUnit.SECONDS)
                .build();
        Request request = new Request.Builder()
                .url("http://localhost:8088/api/hello")
                .build();
        Response response = client.newCall(request).execute();
        if (response.isSuccessful()) {
            ResponseBody body = response.body();
            if (null == body) {
                System.out.println("empty");
                return;
            }
            System.out.println(body.string());
        }
    }
}
```

## 2.（选做）使用压测工具（wrk 或 sb），演练 gateway-server-0.0.1-SNAPSHOT.jar 示例。

```
--------40个并发，测试30s
admin@admindeMacBook-Pro-3 ~ % wrk -c40 -d30s http://localhost:8088/api/hello
Running 30s test @ http://localhost:8088/api/hello
  2 threads and 40 connections
  Thread Stats   Avg(平均值)  Stdev(标准值)   Max(最大值)   +/- Stdev
    Latency(延迟)     7.95ms   51.21ms 639.20ms   97.95%
    Req/Sec(每秒请求数)    24.07k     8.44k   44.41k    67.46%
  1416614 requests in 30.03s, 169.13MB read
Requests/sec:  47180.25
Transfer/sec:      5.63MB
 
--------10w并发，测试30s 
admin@admindeMacBook-Pro-3 ~ % wrk -c100000 -d30s http://localhost:8088/api/hello
Running 30s test @ http://localhost:8088/api/hello
  2 threads and 100000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    87.01ms   96.49ms   1.95s    94.90%
    Req/Sec    17.49k    12.61k   51.76k    59.83%
  423560 requests in 30.05s, 50.54MB read
  Socket errors: connect 95143, read 27404, write 0, timeout 3759
Requests/sec:  14095.31
Transfer/sec:      1.68MB

--------10w并发，开启10个线程，测试30s
admin@admindeMacBook-Pro-3 ~ % wrk -c100000 -d30s -t10  http://localhost:8088/api/hello
Running 30s test @ http://localhost:8088/api/hello
  10 threads and 100000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    84.25ms  125.10ms   1.93s    98.77%
    Req/Sec     4.40k     3.10k   47.91k    81.58%
  992804 requests in 30.12s, 118.50MB read
  Socket errors: connect 95151, read 15383, write 1, timeout 3623
Requests/sec:  32962.12
Transfer/sec:      3.93MB

```

发现当不知道具体请求的端口号时，将gateway-server-0.0.1-SNAPSHOT.jar解压之后，可以从yaml中的配置得知。而想要知道请求的路径，根据大概猜测的HelloController.class执行javap -c -verbose命令，在对应的方法字节码中，其实是可以看见请求的路径，因为这类注解通常是需要保留到运行时，所以数据肯定是存在字节码文件中的.
```
public java.lang.String sayHello(javax.servlet.http.HttpServletRequest);
    descriptor: (Ljavax/servlet/http/HttpServletRequest;)Ljava/lang/String;
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=2, args_size=2
         0: ldc           #2                  // String hello world
         2: areturn
      LineNumberTable:
        line 16: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       3     0  this   Lio/github/kimmking/gateway/server/HelloController;
            0       3     1 request   Ljavax/servlet/http/HttpServletRequest;
    MethodParameters:
      Name                           Flags
      request
    RuntimeVisibleAnnotations:
      0: #18(#19=[s#20])
        org.springframework.web.bind.annotation.GetMapping(
          value=["/api/hello"]
        )
```
## 3.（选做）如果自己本地有可以运行的项目，可以按照 2 的方式进行演练。
```
Running 20s test @ http://localhost:8080/xxxxx
  10 threads and 10000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   618.07ms  372.17ms   1.20s    83.33%
    Req/Sec    18.67     13.48    70.00     75.67%
  812 requests in 20.10s, 4.39MB read
  Socket errors: connect 5151, read 4988, write 0, timeout 806
Requests/sec:     40.39
Transfer/sec:    223.42KB

```
对自己的一个Web系统对应的会员中心页面进行本地测试, 平均下来每秒40个请求。

## 1.（选做）使用 GCLogAnalysis.java 自己演练一遍串行 / 并行 /CMS/G1 的案例。

### 单线程收集器 + 指定内存1G做分析
```
admin@admindeMacBook-Pro-3 01jvm环境准备说明 % java -XX:+PrintGCDetails -XX:+UseSerialGC -Xms1g -Xmx1g GCLogAnalysis
正在执行...
[GC (Allocation Failure) [DefNew: 279616K->34944K(314560K), 0.0398939 secs] 279616K->80499K(1013632K), 0.0399204 secs] [Times: user=0.02 sys=0.01, real=0.04 secs]
[GC (Allocation Failure) [DefNew: 314560K->34943K(314560K), 0.0586362 secs] 360115K->160178K(1013632K), 0.0586636 secs] [Times: user=0.04 sys=0.02, real=0.06 secs]
[GC (Allocation Failure) [DefNew: 314559K->34943K(314560K), 0.0461874 secs] 439794K->238092K(1013632K), 0.0462137 secs] [Times: user=0.03 sys=0.02, real=0.05 secs]
[GC (Allocation Failure) [DefNew: 314559K->34944K(314560K), 0.0499400 secs] 517708K->322280K(1013632K), 0.0500078 secs] [Times: user=0.03 sys=0.02, real=0.05 secs]
[GC (Allocation Failure) [DefNew: 314560K->34943K(314560K), 0.0428847 secs] 601896K->394818K(1013632K), 0.0429122 secs] [Times: user=0.03 sys=0.01, real=0.04 secs]
[GC (Allocation Failure) [DefNew: 314559K->34944K(314560K), 0.0458776 secs] 674434K->471770K(1013632K), 0.0459052 secs] [Times: user=0.02 sys=0.02, real=0.05 secs]
[GC (Allocation Failure) [DefNew: 314560K->34944K(314560K), 0.0456483 secs] 751386K->547590K(1013632K), 0.0456731 secs] [Times: user=0.03 sys=0.02, real=0.05 secs]
[GC (Allocation Failure) [DefNew: 314560K->34943K(314560K), 0.0461142 secs] 827206K->627261K(1013632K), 0.0461386 secs] [Times: user=0.02 sys=0.02, real=0.04 secs]
[GC (Allocation Failure) [DefNew: 314119K->34944K(314560K), 0.0514847 secs] 906436K->712087K(1013632K), 0.0515110 secs] [Times: user=0.03 sys=0.02, real=0.05 secs]
[GC (Allocation Failure) [DefNew: 314560K->314560K(314560K), 0.0000108 secs][Tenured: 677143K->381686K(699072K), 0.0568577 secs] 991703K->381686K(1013632K), [Metaspace: 2582K->2582K(1056768K)], 0.0569054 secs] [Times: user=0.05 sys=0.00, real=0.06 secs]
执行结束!共生成对象次数:11529
Heap
 def new generation   total 314560K, used 260925K [0x0000000780000000, 0x0000000795550000, 0x0000000795550000)
  eden space 279616K,  93% used [0x0000000780000000, 0x000000078fecf478, 0x0000000791110000)
  from space 34944K,   0% used [0x0000000793330000, 0x0000000793330000, 0x0000000795550000)
  to   space 34944K,   0% used [0x0000000791110000, 0x0000000791110000, 0x0000000793330000)
 tenured generation   total 699072K, used 381686K [0x0000000795550000, 0x00000007c0000000, 0x00000007c0000000)
   the space 699072K,  54% used [0x0000000795550000, 0x00000007aca0d880, 0x00000007aca0da00, 0x00000007c0000000)
 Metaspace       used 2589K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 278K, capacity 386K, committed 512K, reserved 1048576K

```
总结: 
在单线程收集器中，新生代叫做DefNew, 如图所示，共发生了10多次YoungGC，GC暂用的时间就是STW时间，即40-50ms左右。

新生代总大小为: 341560,大约为333MB.第一次新生代GC时，新生代由279616K回收至34944K的使用量，大概被回收了238MB，而堆内存由279616K至80499K，被回收了194MB，那么238 - 194 = 44，大概有44MB的对象被转移到了老年代。 

另外发生了一次Old GC,即老年代的GC, 老年代在单线程收集器中被叫做Tenured，最后一次GC为新生代和老年代GC, 老年代GC的效果很明显，即677143K- 381686K= 288MB。Metaspace区域并没有发生变化。老年代发生了的一次GC为60毫秒左右。 

由最后一次老年带回收信息可知，其老年代一共被分配到682MB，与新生代总大小333，大约是2:1。

### 单线程收集器 + 未指定内存1G做分析
```
admin@admindeMacBook-Pro-3 01jvm环境准备说明 % java -XX:+PrintGCDetails -XX:+UseSerialGC -XX:-UseAdaptiveSizePolicy GCLogAnalysis
正在执行...
[GC (Allocation Failure) [DefNew: 69952K->8704K(78656K), 0.0127925 secs] 69952K->24298K(253440K), 0.0128213 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
[GC (Allocation Failure) [DefNew: 78624K->8703K(78656K), 0.0209974 secs] 94218K->53454K(253440K), 0.0210237 secs] [Times: user=0.02 sys=0.01, real=0.02 secs]
[GC (Allocation Failure) [DefNew: 78655K->8702K(78656K), 0.0148888 secs] 123406K->75401K(253440K), 0.0149150 secs] [Times: user=0.01 sys=0.01, real=0.02 secs]
[GC (Allocation Failure) [DefNew: 78654K->8702K(78656K), 0.0143347 secs] 145353K->96982K(253440K), 0.0143630 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
[GC (Allocation Failure) [DefNew: 78644K->8703K(78656K), 0.0175214 secs] 166923K->122998K(253440K), 0.0175518 secs] [Times: user=0.01 sys=0.01, real=0.01 secs]
[GC (Allocation Failure) [DefNew: 78655K->8702K(78656K), 0.0149137 secs] 192950K->144548K(253440K), 0.0149420 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
[GC (Allocation Failure) [DefNew: 78654K->8701K(78656K), 0.0144353 secs] 214500K->164535K(253440K), 0.0144658 secs] [Times: user=0.01 sys=0.01, real=0.01 secs]
[GC (Allocation Failure) [DefNew: 78653K->8702K(78656K), 0.0154744 secs][Tenured: 177234K->159088K(177280K), 0.0305100 secs] 234487K->159088K(255936K), [Metaspace: 2582K->2582K(1056768K)], 0.0462145 secs] [Times: user=0.04 sys=0.01, real=0.04 secs]
[GC (Allocation Failure) [DefNew: 106112K->13247K(119360K), 0.0128325 secs] 265200K->192823K(384508K), 0.0128638 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
[GC (Allocation Failure) [DefNew: 119080K->13247K(119360K), 0.0272922 secs] 298655K->224820K(384508K), 0.0273212 secs] [Times: user=0.02 sys=0.02, real=0.03 secs]
[GC (Allocation Failure) [DefNew: 118979K->13245K(119360K), 0.0218361 secs] 330552K->259965K(384508K), 0.0218634 secs] [Times: user=0.01 sys=0.00, real=0.02 secs]
[GC (Allocation Failure) [DefNew: 119357K->13244K(119360K), 0.0237939 secs][Tenured: 285295K->242404K(285308K), 0.0340404 secs] 366077K->242404K(404668K), [Metaspace: 2582K->2582K(1056768K)], 0.0581136 secs] [Times: user=0.04 sys=0.01, real=0.06 secs]
[GC (Allocation Failure) [DefNew: 161728K->20159K(181888K), 0.0164561 secs] 404132K->296399K(585896K), 0.0164839 secs] [Times: user=0.01 sys=0.01, real=0.01 secs]
[GC (Allocation Failure) [DefNew: 181887K->20159K(181888K), 0.0365883 secs] 458127K->344864K(585896K), 0.0366179 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]
[GC (Allocation Failure) [DefNew: 181887K->20157K(181888K), 0.0321157 secs] 506592K->392261K(585896K), 0.0321428 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]
[GC (Allocation Failure) [DefNew: 181885K->20158K(181888K), 0.0316204 secs][Tenured: 419092K->321362K(419180K), 0.0464079 secs] 553989K->321362K(601068K), [Metaspace: 2582K->2582K(1056768K)], 0.0782893 secs] [Times: user=0.06 sys=0.02, real=0.08 secs]
[GC (Allocation Failure) [DefNew: 214336K->26749K(241088K), 0.0232515 secs] 535698K->393574K(776696K), 0.0232794 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]
[GC (Allocation Failure) [DefNew: 241085K->26752K(241088K), 0.0301993 secs] 607910K->459357K(776696K), 0.0302297 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]
[GC (Allocation Failure) [DefNew: 241088K->26751K(241088K), 0.0374021 secs] 673693K->516342K(776696K), 0.0374301 secs] [Times: user=0.02 sys=0.01, real=0.03 secs]
[GC (Allocation Failure) [DefNew: 241087K->26752K(241088K), 0.0442099 secs][Tenured: 557430K->356049K(557612K), 0.0567246 secs] 730678K->356049K(798700K), [Metaspace: 2582K->2582K(1056768K)], 0.1010837 secs] [Times: user=0.08 sys=0.02, real=0.10 secs]
执行结束!共生成对象次数:9414
Heap
 def new generation   total 267136K, used 9766K [0x00000006c0000000, 0x00000006d21d0000, 0x0000000715550000)
  eden space 237504K,   4% used [0x00000006c0000000, 0x00000006c0989b08, 0x00000006ce7f0000)
  from space 29632K,   0% used [0x00000006ce7f0000, 0x00000006ce7f0000, 0x00000006d04e0000)
  to   space 29632K,   0% used [0x00000006d04e0000, 0x00000006d04e0000, 0x00000006d21d0000)
 tenured generation   total 593420K, used 356049K [0x0000000715550000, 0x00000007398d3000, 0x00000007c0000000)
   the space 593420K,  59% used [0x0000000715550000, 0x000000072b1047f8, 0x000000072b104800, 0x00000007398d3000)
 Metaspace       used 2589K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 278K, capacity 386K, committed 512K, reserved 1048576K
```

总结: 在未指定内存的情况下，按照第一次垃圾回收给定的新生代大小为: 253440 大约247MB ,以及第一次老年代回收时，总内存大小为173MB，JVM会在垃圾收集完毕之后，再对堆空间大小进行扩容(第一次FullGC之后，老年代扩大至278MB)。所以如果不设置-Xms和-Xmx，则有可能jvm在将堆内存(新生代和老年代)调整到默认的最大值(机器内存的1/4)，会发生比较频繁的GC, 因为刚开始，新生代和老年代都被分配的比较小，进行对象分配空间时，对象优先在新生代上分配，新生代空间不足时，则进行垃圾回收。这个时候新生代还没有扩容，当对象无法在新生代中分配大对象或者一部分对象达到默认15次收集年龄时，被分配到老年代中，由于老年代的空间也不大，所以连老年代也发生了GC。老年代发生GC之后，发现新生代和老年代均发生了扩容。
