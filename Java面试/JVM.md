###  其他JVM 参数

- **verbose:gc**：启动jvm的时候，输出jvm里面的gc信息
- **-XX:+printGC**：打印的GC信息
- **-XX:+PrintGCDetails**：打印GC的详细信息
- **-XX:+PrintGCTimeStamps**：打印GC发生的时间戳
- **-X:loggc:log/gc.log**：指定输出gc.log的文件位置
- **-XX:+PrintHeapAtGC**：表示每次GC后，都打印堆的信息
- **-XX:+TraceClassLoading**：监控类的加载
- **-XX:+PrintClassHistogram**：跟踪参数
- **-XX:+HeapDumpOnOutMemoryError**：发生OOM时，导出堆的信息到文件
- **-XX:+HeapDumpPath**：导出堆信息的文件路径
- **-XX:+OnOutOfMemoryError**：当系统产生OOM时，执行一个指定的脚本，这个脚本可以是任意功能的。比如生成当前线程的dump文件，或者是发送邮件和重启系统。
- **-XX:+Xss**：设置栈的大小。栈都是每个线程独有一个，所有一般都是几百k的大小
- **-XX:PermSize， -XX:MaxPermSize**：设置永久区的内存大小和最大值；

## GC调优
+ 减少对象的创建
+ 在平时写代码时，多使用StringBuilder和StringBuffer对象替代String，减少不必要的日志输出
+ 降低移动到老年代的对象数量
+ 缩短Full GC的执行时间

### GC调优需要关注的选项
<table>
  <tr>
    <td>堆空间</td>
    <td>-Xms</td>
    <td>启动JVM时的初始堆空间大小</td>
  </tr>
  <tr>
    <td></td>
    <td>-Xmx</td>
    <td>堆空间最大值</td>
  </tr>
  <tr>
    <td>新生代空间</td>
    <td>-XX:NewRatio</td>
    <td>新生代与老年代的比例</td>
  </tr>
  <tr>
    <td></td>
    <td>-XX:NewSize</td>
    <td>新生代大小</td>
  </tr>
  <tr>
    <td></td>
    <td>-XX:SurvivorRatio</td>
    <td>Eden区与Survivor区的比例</td>
  </tr>
</table>

### 1. 监控GC状态
### 2.分析监控数据并决定是否需要GC调优
使用**jstat**命令监控Web应用的GC运行状态。

如果GC执行时间满足以下判断条件，那么GC调优并没那么必须，ps：括号内的值并非绝对：
+ Minor GC执行迅速(50毫秒以内)
+ Minor GC执行不频繁(间隔10秒左右一次)
+ Full GC执行迅速(1秒以内)
+ Full GC执行不频繁(间隔10分钟左右一次)

**在校验GC状态时，不要只关心Minor GC和Full GC的耗时，GC执行次数也同样重要**。如果新生代太小，Minor GC就会频繁执行(甚至每间隔1秒就要执行一次)。另外，新生代太小导致转移到老年代的对象增多，也会引起Full GC的频繁执行。**因此使用'-gccapacity'配合jstat命令，以检查内存空间的使用情况。**

### 3. 设置GC类型和内存大小
#### 设置GC类型
例如通常Full GC中使用CMS GC会执行更快，如果CMS GC的并发模式失败，则会出现比Parallel GCs慢的情况。
所谓**并发模式是指CMS由于采用的是标记-清除算法，会产生碎片空间。所以可能会出现老年代还有100M的空闲空间，却不能分配10M的连续空间**。这时便会发生并发模式失败的警告，并触发内存压缩。如果使用CMS GC，在内存压缩过程中可能会比Parallel GCs更为耗时，也可能会带来其他问题。

#### 调整内存大小
内存大小与GC执行次数、每次GC耗时之间的关系：
+ 大内存
    - 会降低GC执行次数
    - 相应的会增加GC执行耗时
+ 小内存
    - 会缩短单次GC耗时
    - 相应的会增加GC执行次数

### 4. 分析GC调优结果
### 5. 如果结果可接受，则对所有服务应用调优选项并停止调优
