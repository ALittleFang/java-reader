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
