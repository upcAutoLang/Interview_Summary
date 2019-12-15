# 服务假死问题解决过程实记——Tomcat 与数据库连接池配置引发的血案

## 一、前言

自三月六日起，笔者所在业务组的开发环境上出现了若干次服务假死，页面请求无响应的现象。由于笔者在三月六日之前，对 JVM, Tomcat，以及数据库连接池没有丝毫调优经验，所以从三月六日开始的所有与解决该问题的过程，都会记录到本文，以记录并纪念笔者的第一次服务调优经历。

## 二、03.06 记 Tomcat 的一次假死问题解决经历

> 注：2019.03.30 记，该现象的源头是因为 C3P0 参数配置问题，现已解决该数据库连接问题。但这只是问题解决过程中顺手解决的另一个问题而已，服务假死的原因应该是因为其他原因，该问题并非源头。

### 1. 测试环境服务假死

现象：未知具体操作，但出现 Tomcat 假死情况，无法使用 jmap, jstat, jstack 指令以及 jvisualVM 工具，且使用 netstat -ano | findstr "12808" 指令后，出现七八个 CLOSE\_WAIT 连接。

猜测原因：

1. ClassLoader 的更换，导致方法区溢出？
2. 反射使用太多？

待解决方法：

1. 重启服务，待正常运行过程中使用 JVisualVM 监控，看方法区以及堆区的大小变化；
2. 如果还需要重新启动服务，加入虚拟机 GC 参数：-XX:HeapDumpOnOutOfMemoryError；
3. 如果确定是 ClassLoader 堆积的问题，则加入类卸载参数：-Xnoclassgc；  

> 相关参考连接：  
> 
> 1. 关于 CLOSE\_WAIT 状态，来自于关闭 TCP 连接的四次握手：  
>   - [《【服务器】一次对Close_Wait 状态故障的排查经历
》](https://www.cnblogs.com/DannielZhang/p/8000496.html)
>   - [《TCP的三次握手与四次分手》](https://www.cnblogs.com/leezhxing/p/4524176.html)
> 2. [《netstat监控大量ESTABLISHED连接数和TIME_WAIT连接数题解决》](https://blog.csdn.net/bluetjs/article/details/80965967)


### 2. 再次假死，并成功定位问题

由于昨天有了一次假死，且假死过程中已经不能使用 JVisualVM 连接 Tomcat 服务，所以在服务重启之前，我就已经打开了 JVisualVM 远程监控。但到了下午 17:24 再次假死。

#### (1) 第一个排查方向：方法区与堆内存

师父一直怀疑是因为业务原因，使用过自定义的 ClassLoader，并存在稍微频繁加载类的可能性，所以他的检测重点在于方法区。但开 JVisualVM 发现方法区大小一直稳定在 150+M，从始至终很稳定的很小幅度增长，远没有到我们设置的 MaxPermSize = 500M。此外给虚拟机设置的 -Xmx:4096M，但实际使用堆内存只有 800M 左右。

#### (2) 第二个重点排查方向：激增的线程数量

我的监控重点，在于观察到 **17:24 线程数飙升，从 240 个线程瞬间涨到了 400+ 个**。打开 JVisualVM 的线程监控界面，发现几乎同一时间 pool-22, pool-37 创建了大量线程。使用线程 Dump，读取 Dump 文件，发现大部分 BLOCK 的线程都是汇总数据的内容，以及在 save, update, delete 方法上加的 After AOP 中的异步日志记录方法。所以线程暴增的原因，肯定和 17:24 出现的某种 save 操作有关。果然在我写的记录 save 方法操作记录的日志文件，结合项目日志，了解到 17:24 分有测试人员调用 XML 解析入库的方法，且调用了十几种不同的表的插入操作，估算了一下自己写的代码，大概一次 save 操作应该会加入十一二个 task 到阻塞任务队列中，所以线程数量暴增的问题定位到了。

但是，**以前测试人员也做过很多次类似的解析 XML 入库的操作，并没有引发过 Tomcat 假死的错误。**我感觉很奇怪，所以还是只把线程数量激增的情况当成了一种现象，并未完全它是问题的源头。

#### (3) 第三个重点排查方向：CLOSE\_WAIT 数量

上网搜 "Tomcat 假死"，帖子 [《tomcat假死之谜？》](https://bbs.csdn.net/topics/392190187?list=lz) 介绍，会出现 100+ 个 CLOSE\_WAIT 的现象，导致了 Tomcat 崩溃。那时候老子还不会使用 netstat 指令，甚至连 CLOSE\_WAIT 都是前一天晚上临时抱佛脚学的，就依葫芦画瓢输了个 netstat -ano | findstr "CLOSE\_WAIT"，结果出现只有 10 个左右的 CLOSE\_WAIT 是和我们的服务有关的，虽然有了找到错误的那么点儿意思，但 10 个 CLOSE\_WAIT 也太没牌面了吧…… 

#### (4) 第四个重点排查方向：TIME\_WAIT 数量

大概有一个多小时纠结在 CLOSE\_WAIT 上面，后来还是尝试着按照帖子的做法来做，开始监控 TIME\_WAIT。改了个 netstat 指令：**netstat -ano | findstr "TIME\_WAIT"**。<font color=red>**结果发现有 200+ 个 TIME\_WAIT**！！</font>

> ”终于上量级了！！有牌面了啊妈蛋！！“——笔者的内心世界

TIME\_WAIT 的内容是啥呢？大致内容都是：

```txt
......
tcp    16.12.104.133:55177    16.12.181.151:1521   TIME_WAIT   
tcp    16.12.104.133:56430    16.12.181.151:1521   TIME_WAIT   
tcp    16.12.104.133:55659    16.12.181.151:1521   TIME_WAIT   
tcp    16.12.104.133:55141    16.12.181.151:1521   TIME_WAIT   
tcp    16.12.104.133:56582    16.12.181.151:1521   TIME_WAIT   
tcp    16.12.104.133:55240    16.12.181.151:1521   TIME_WAIT   
tcp    16.12.104.133:54811    16.12.181.151:1521   TIME_WAIT   
tcp    16.12.104.133:55772    16.12.181.151:1521   TIME_WAIT   
......
```

重点来了，虽然不懂 TIME\_WAIT，但**<font color=red>大片的 1521</font>** 也太醒目了吧啊喂！！

1521 是确实是 Oracle 默认的连接端口号，而且对应的 16.12.181.151 也确实是我们要连的数据库，但是**我们的服务又不是直连数据库的**，怎么会出现这么多的 1521 端口的 TIME\_WAIT 呢？莫非是 RPC？不对，我们的 RPC 应该是直连 DAO 服务的，<font color=red>**DAO 服务才会连接数据库**</font>，所以肯定也不会是 RPC 的原因。

忽然我想到了一个很重要的事情：由于我们的资源有限，只分配了两台虚拟机，对于我们这个有四个服务 (两个应用服务，两个 DAO 服务) 的业务组，我们把应用服务和 DAO 服务交叉配置，所以在我们服务部署的 16.12.104.133 虚拟机上，**确实是有一个 DAO 服务的，而且和我们的应用服务部署在同一个 Tomcat 下**！

#### (5) 确定问题所在的证据

**首先，我先检查其他业务组的正常 DAO 服务。**远程连到其他正常 DAO 服务的虚拟机上输入同样的指令 **netstat -ano | findstr "TIME\_WAIT"**，以及 **netstat -ano | findstr "CLOSE\_WAIT"**，都不会出现几行。所以可以确定这个 DAO 服务肯定是有异常的。

**然后，端口数量。**还是刚才 16.12.133.104 上连接数据库的端口数，是很明显的 +1 递增趋势，最大的端口数基本上是 65400+ 左右。我专门查了一下**端口最大连接数量**，知道了 TCP 最大连接数量是 65536 个，所以这种占据了大量端口的现象，一定是异常！！

确认了异常的原因，是**<font color=red>部署在 16.12.104.133 上同 Tomcat 下的 DAO 层服务的数据库连接异常</font>**。这样也可以解释，为什么 JVM 监控正常，GC 情况无异常，且在假死现象出现后甚至无法使用 JVisualVM 连接服务的诸多情况都可以解释了。

> 注：  
> 值得一说的是，对于基于 TCP 的 HTTP 协议，关闭 TCP 连接的是 Server 端，这样，Server 端会进入 TIME\_WAIT 状态，可想而知，对于访问量大的 Web Server，会存在大量的 TIME\_WAIT 状态，假如 server 一秒钟接收 1000 个请求，那么就会积压 240*1000=240000 个 TIME\_WAIT 的记录，维护这些状态给 Server 带来负担。当然现代操作系统都会用快速的查找算法来管理这些 TIME\_WAIT，所以对于新的 TCP 连接请求，判断是否 hit 中一个 TIME\_WAIT 不会太费时间，但是有这么多状态要维护总是不好。  
> HTTP 协议 1.1 版规定 default 行为是 Keep-Alive，也就是会重用 TCP 连接传输多个 request/response，一个主要原因就是发现了这个问题。


## 三、03.20 Tomcat 假死后续

开发环境假死多次，出现的现象有：

1. (03.20, 15:20) jmap -dump 指令，出现错误：36604: Insufficient memory or insufficient privileges to attach; 加 -F 指令可以 dump，但无法加载到 JVisualVM 中；
2. (03.20, 15:40) jmap -heap 指令，老年代部分 concurrent mark-sweep generation，显示异常，出现极大的值，且使用率达到 -5E11%；
	- 已知是 bug，已在 Java9 中修复；
	- [《JDK-8033440 : jmap reports unexpected used/free size of concurrent mark-sweep generation》](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=8033440)
3. (03.20, 17:20) 张磊服务有一次无法登陆，现象与服务器上的服务相同：不能通过首页登陆，但可以通过 URL 查询到某些数据并响应；报错如下：
	- org.springframwork.....InternalAuthenticationServiceException: UserDetailsService returned null, which is an interface contract violation;
4. (03.07, 18:00; 03.20, 15:40) 无法使用 JVisialVM 连接，jmap -dump -F 无法 dump 成功；jstat 无法连接，jstack 可以使用；
5. (03.20, 15:40) 服务器上，多次使用 jstack，发现随着中间多次尝试访问服务，但 jstack 结果中文件内容增多，线程堆积，且后面出现线程池拒绝的现象；thread Dump 内容出现频率最高的内容：
	- 线程池满；
	- thymeleaf 渲染； 
	- native 方法：getDeclaredConstructors0(boolean);
		- Google 之后，好几个关于 getDeclaredConstructors0 的 BLOCKED 状态是因为 **Debug 端口打开**的原因，是否如此？参考链接如下：
		- [《Why java newInstance hang at getDeclaredConstructors0?》](https://stackoverflow.com/questions/37564701/why-java-newinstance-hang-at-getdeclaredconstructors0)
		- [《getDeclaredConstructors0(boolean)- line not available during tomcat startup in debug mode》](https://stackoverflow.com/questions/27114001/getdeclaredconstructors0boolean-line-not-available-during-tomcat-startup-in-d)

<font color=red>**注：一篇 Tomcat 配置讲的很好的文章[《对tomcat来说，每一个进来的请求(request)都需要一个线程，直到该请求结束。》](https://www.cnblogs.com/softidea/p/5750791.html)**</font>

## 四、03.30 Tomcat 假死后续——C3P0 连接池参数配置问题

昨晚上正在看有关 B+Tree 相关的内容，收到业务组的微信消息：

> 最帅气的大龙龙：现场数据库连接不上，他们排查问题，怀疑与连接池或者日志有关系，最后发现从昨天下午到现在产生 30 万条日志，其中我们就有 22 万条，明天查一下我们服务 @琦小虾
 
好吧，那就和师父一起查问题好了。第二天早上，果然数据库组的同事过来和我们说了说情况，说现场传来的具体情况：现场忽然之间所有业务都不能连接 Oracle，后来查询了下原因，看到 Oracle 的**监听日志过大**，导致所有业务不能连接数据库。后来通过某些手段打开 Oracle 监听日志 (listener.log)，发现总共产生了 30 万条日志，我们业务组相关的日志占了 20+ 万条。所以建议我们检查一下数据库连接池相关的参数。

> 注：Oracle 监听日志文件过大导致无法数据库无法连接的相关问题参考连接：  
> [《ORACLE的监听日志太大，客户端无法连接 BUG:9879101》](https://blog.csdn.net/u013770979/article/details/50968612)  
> [《ORACLE清理、截断监听日志文件（listener.log）》](http://www.cnblogs.com/kerrycode/p/4227579.html)

数据库组老大专门来我**师父的机器**上的 C3P0 的数据库连接池相关参数，大佬感觉没什么问题（然而是个小坑）。那是为什么呢？最好的方法还是调过来开发环境的 Oracle 监听日志看看吧。  
经过我一番猛如虎的操作，我们把日志分析的准备工作做好了：

1. 安装 XShell 用 sftp 连接 Oracle 所在的 CentOS 服务器，把数据库监听日志 listener.log 宕到本机；
2. 监听日志记录了两个月的日志信息，大小大概有 5 个多 G；记事本与 NotePad 都不能打开这么大的日志文件；
3. 由于不能连接外网下载第三方工具，我在网上找了个 Java 方法，用 NIO 的方法把 5G 的日志文件分成了 200 个文件，这样就可以进行分析了。

```java
public static void splitFile(String filePath, int fileCount) throws IOException {
    FileInputStream fis = new FileInputStream(filePath);
    FileChannel inputChannel = fis.getChannel();
    final long fileSize = inputChannel.size();
    long average = fileSize / fileCount;//平均值
    long bufferSize = 200; //缓存块大小，自行调整
    ByteBuffer byteBuffer = ByteBuffer.allocate(Integer.valueOf(bufferSize + "")); // 申请一个缓存区
    long startPosition = 0; //子文件开始位置
    long endPosition = average < bufferSize ? 0 : average - bufferSize;//子文件结束位置
    for (int i = 0; i < fileCount; i++) {
        if (i + 1 != fileCount) {
            int read = inputChannel.read(byteBuffer, endPosition);// 读取数据
            readW:
            while (read != -1) {
                byteBuffer.flip();//切换读模式
                byte[] array = byteBuffer.array();
                for (int j = 0; j < array.length; j++) {
                    byte b = array[j];
                    if (b == 10 || b == 13) { //判断\n\r
                        endPosition += j;
                        break readW;
                    }
                }
                endPosition += bufferSize;
                byteBuffer.clear(); //重置缓存块指针
                read = inputChannel.read(byteBuffer, endPosition);
            }
        }else{
            endPosition = fileSize; //最后一个文件直接指向文件末尾
        }

        FileOutputStream fos = new FileOutputStream(filePath + (i + 1));
        FileChannel outputChannel = fos.getChannel();
        inputChannel.transferTo(startPosition, endPosition - startPosition, outputChannel);//通道传输文件数据
        outputChannel.close();
        fos.close();
        startPosition = endPosition + 1;
        endPosition += average;
    }
    inputChannel.close();
    fis.close();

}

public static void main(String[] args) throws Exception {
    long startTime = System.currentTimeMillis();
    splitFile("/Users/yangpeng/Documents/temp/big_file.csv",5);
    long endTime = System.currentTimeMillis();
    System.out.println("耗费时间： " + (endTime - startTime) + " ms");
}
```

好吧，终于可以分析日志了。

既然所有业务组都在和这个 Oracle 连接，那么就统计一下几个流量比较大的服务的 IP 出现频率吧。随手打开分割的 200 个中随便一个日志 (25M)，首先用 NotePad 统计了一下所有业务都会访问的共享数据 DAO 服务 IP，总共 300+ 的频率。  
我们业务 DAO 服务几个 IP 的频率呢？**52796 + 140293 + 70802 + <font color=red>142</font> = 264033** 次……

文件里看到了满屏熟悉的 IP…… 没错，这些 IP 就是我们曾经以及正在运行过 DAO 服务的四台主机 IP 地址…… (其中 28.1.25.91 就是区区在下臭名昭著的开发机 IP 地址)

> ...
> 22-MAR-2019 13:23:41 * (CONNECT_DATA=(SERVICE_NAME=DBdb)(CID=()(HOST=SC-201707102126)(USER=Administrator))) * (ADDRESS=(PROTOCOL=tcp)(HOST=**28.1.25.91**)(PORT=53088)) * establish * DBdb * 0
> 22-MAR-2019 13:23:41 * (CONNECT_DATA=(SERVICE_NAME=DBdb)(CID=()(HOST=SC-201707102126)(USER=Administrator))) * (ADDRESS=(PROTOCOL=tcp)(HOST=**28.1.25.91**)(PORT=53088)) * establish * DBdb * 0
> ...

**“完了，这次真的要背血锅了。哈哈哈哈哈。”**这是我第一反应，发现了服务的坑，我竟然这么兴奋哇哈哈哈哈～

随手选了一个文件就是这样，检查了一下其他分割的日志文件，也全都是这种情况。但为什么这个日志文件里，我们四个不同的服务地址总共出现了 26 万次 IP 地址，其中一个只有 142 次，和其他三个 IP 频率差了这么多？  
原来这个 142 次的 IP 是我师父的开发机 IP，而且他自己也说不清楚什么时候出于什么思考，把数据库的连接池给改小了（就是数据库组大佬亲自检查后说没有问题的那组参数），然而我和其他小伙伴的 DAO 服务 C3P0 参数没有改过，所以只有我们三台服务的 IP 地址显得那么不正常。哈哈哈我师父果然是个心机 BOY～

> 注：关于 C3P0 参数设置的相关内容笔者总结到了另一篇博客里：[《C3P0 连接池相关概念》](https://blog.csdn.net/ajianyingxiaoqinghan/article/details/88931960)

之前的 C3P0 参数是这样的：

```css
# unit:ms
cpool.checkoutTimeout=60000
cpool.minPoolSize=200
cpool.initialPoolSize=200
cpool.maxPoolSize=500
# unit:s
cpool.maxIdleTime=60
cpool.maxIdleTimeExcessConnections=20
cpool.acquireIncrement=5
cpool.acquireRetryAttempts=3
```

修改后的 C3P0 参数是这样的：

```css
# unit:ms
cpool.checkoutTimeout=5000
cpool.minPoolSize=5
cpool.maxPoolSize=500
cpool.initialPoolSize=5
# unit:s
cpool.maxIdleTime=0
cpool.maxIdleTimeExcessConnections=200
cpool.idleConnectionTestPeriod=60
cpool.acquireIncrement=5
cpool.acquireRetryAttempts=3
```

> **04.21 记**：cpool.maxIdleTimeExcessConnections=200 依旧是个大坑，后续解析。

可以看出来，差距最大的几个参数是：

- **cpool.checkoutTimeout**: 60000 -> 5000 (单位 ms)
- **cpool.minPoolSize**: 200 -> 5
- **cpool.initialPoolSize**: 200 -> 5
- **cpool.maxIdleTime**: 60 -> 0 (单位 s)

所以，**<font color=red>可以总结现场出现的现象如下：</font>**

1. 我们的 DAO 服务由于设置了 initialPoolSize 的值为 200，所以 DAO 服务在一开始启动的时候，就已经和 Oracle 建立了 200 个连接；
2. 由于服务大部分时间都不会有太多人使用，所以运行过程中每超过 maxIdleTime 的时间即 60 秒后，没有被使用到的数据库连接被释放。一般释放的连接数量大约会在 195 ~ 200 个左右；
3. 刚刚释放了大量的数据库连接（数量计作 size），由于 minPoolSize 设置为 200，所以立即又会发起 size 个数据库连接，使数据库连接数量保持在 minPoolSize 个；
4. 每 60s (maxIdleTime) 重复 2~3 步骤；

所以现场的 Oracle 的监听日志也会固定每 60 秒 (maxIdleTime) 添加约 200 条，运行了一段时间后，就出现了 Oracle 监听日志过大（一般情况下指一个 listener.log 监听文件大于 4G），Oracle 数据库无法被连接的情况。

所以，前面三月六日我发现的大量出现 1521 端口的 TIME\_WAIT，就应该是 DAO 服务端检测到有 200 个空闲连接，便为这些连接向数据库发送关闭请求，然后这些连接在等待 maxIdleTime 时间的过程中就进入了 TIME\_WAIT 状态。释放这些连接后由于 minPoolSize 设置值为 200，所以又重新发起了约 200 个新的数据库连接。<font color=red>所以我如果在 cmd 中随一定时间周期 (每 60s) 输入 netstat -ano 
| findstr "1521" 的指令，列出来的与数据库 1521 端口应该是变动的。</font>

![TCP 四次分手示意图](https://images0.cnblogs.com/blog2015/545411/201505/231512425935841.png)

至此，数据库连接池的问题应该是解决了。但我认为服务假死问题应该不是出在这里。目前怀疑的问题，有因为虚拟机开启了 -XDebug, -Xrunjdwp 参数，也可能是由于我们使用线程池的方式有误。还是需要继续进一步检查啊。

> PS: 这次问题之后，总结了一下我和我师父的工作日常：
> 
> 1. 师父随意黑科技挖坑
> 2. 业务组一脸懵逼入坑
> 3. 笔者陪师父填坑
> 4. 师徒组 LEVEL UP
> 
> 今日份问题：C3P0连接池参数设置优化 get√

------------

## 五、04.15 100 插入并发假死问题——C3P0 连接池参数配置问题

> 参考地址：  
> [《c3p0 不断的输出debug错误信息》](https://blog.csdn.net/greensurfer/article/details/7496283)  

很长一段时间里，在忙一些其他杂事，没有时间开发。终于把杂事忙完之后，笔者和师父在修正了 C3P0 参数之后，开始尝试测试并发性能。  
用 LoadRunner 写了一个脚本，同时 50 个用户并发插入一条数据，无思考时间的插入一分钟。脚本跑起来之后，很快服务就出现了问题。

首先，DAO 服务直接完全假死。而且由于笔者在虚拟机参数中添加了 -XX:+PrintGCDetails 参数，观察到打印出来的 GC 日志，竟然有一秒钟三到四次的 FullGC！而且虚拟机的旧生代已经完全被填满，每次 FullGC 几乎完全没有任何的释放。此外，DAO 服务也会偶尔报出 OutofMemoryError，只是没有引起虚拟机崩溃而已。   
当然，软件服务也由于大量的插入无响应，报出了大量的 Read Time Out 错误。

开始分析问题的时候，笔者也是一脸懵逼。打开 JVisualVM 监控 Java 堆，反复试了多次，依旧是长时间的内存不释放的现象。正当有一次对着 JVisualVM 监控画面发呆，发呆到执行并发脚本几分钟之后，忽然我看到有一次 FullGC 直接令 **<font color=red>Java 堆有了一次断崖式的下降</font>**，堆内存直接下降了 80%！！  
我当时就意识到这就是问题的突破点。所以由重新跑了一次并发脚本复现问题。再次卡死时，我用 jmap 指令把堆内存 Dump 下来，加载到前几天准备好的 Eclipse 插件 Memory Analyse Tool (MAT) 中进行分析。  
果然看到了很异常的 HeapDump 饼图：1.5G 的堆内存，有 70%-80% 的容量都在存着一个名为  **newPooledConnection** 的对象，这种对象的数量大概有 60 个，每个对象大小 20M 左右。这个对象是在 c3p0 的包里，所以用脚指头想就知道，肯定是我们的 C3P0 配置还有问题。

查了一下 C3P0 的配置参数，观察到有一条信息：

- <font color=red>**maxIdelTimeExcessConnections**</font>: 这个配置主要是为了快速减轻连接池的负载，比如连接池中连接数因为某次数据访问高峰导致创建了很多数据连接，但是后面的时间段需要的数据库连接数很少，需要快速释放，必须小于 maxIdleTime。其实这个没必要配置，maxIdleTime 已经配置了。

而此时我看了一眼我们的 C3P0 参数，有这样两个参数：

```properties
cpool.maxIdleTime=0
cpool.maxIdleTimeExcessConnections=200
```

所以由于 cpool.maxIdleTimeExcessConnections=200 这个参数，在并发发生之后，C3P0 持续持有并发后产生的数据库连接，直到 200s 之内没有再复用到这些连接，才会将其释放。所以我之前发呆后忽然的断崖式内存释放，肯定就是因为这个原因……

果然把 maxIdleTime, maxIdleTimeExcessConnections 都设置为 0，并发插入立即变得顺滑了很多。  
至此，DAO 服务最重要的问题找到，对它的优化过程基本告一段落。但我们的服务依旧有很多待优化的点，也有很多业务逻辑可以优化，这是后面一段时间需要考虑的问题。

## 六、04.17—04.21 缓存逻辑修正

这段时间我一直在优化服务的性能，主要是从分布式缓存和业务逻辑修正两个角度出发进行的。首先是将我们的缓存逻辑给修正了一下。

关于缓存，我们业务存在两个重要问题：

- 集群部署的情况下，每个服务都用了很多本地 ConcurrentHashMap 缓存；
- 在业务逻辑计算出结果之后，直接将计算出的结果存在了本地缓存中（即缓存过程与业务逻辑紧密耦合）；

对于第一个问题，主要有两个隐患：首先集群部署，也就意味着为了提高服务的性能，环境中有多台服务，所以对于相同的数据，每个服务都要自己记录一份缓存，这样**对内存是很大的浪费**。其次多台服务的**缓存也很容易出现不同步**的问题，极易出现数据脏读的现象。  
对于第二个问题，将结果存放到缓存中，本身与业务并没有关系，不管是否置入缓存，都不会对业务结果不会有影响。但如果将缓存的一部分放在业务逻辑中，就相当于缓存被强行的绑在了业务逻辑之中。所以对这个问题进行优化，就是将缓存从业务逻辑中解耦。  

笔者是先解决了后一个问题，然后再解决前一个问题。

### 1. 切面思想的体会

我认为将缓存从业务逻辑中解耦，这种工作交给 AOP 后置增强是最合适的。所以我就开始对业务代码进行一通分析，提取出来他们的共同点，将置入缓存的逻辑从业务代码中拆了出来，放到了一个后置切面中。具体思路就是这样，过程不表。  
笔者之前只是会使用 AOP 切面，但在这个过程中，笔者切实的加深了对 AOP 的理解。代码抽取过程中，同事也问我这样做有什么好处，对性能有什么优化？我想了一下，回答：这对性能没有任何优化。同事问我做 AOP 切面的意义，我开了个脑洞，用这个例子给出了一个比较通俗易懂的解释：  

> **问：**把大象放在冰箱里总共分几步？  
> **答：**分三步。第一步把冰箱门打开，第二步把大象给塞进去，第三步把冰箱门关上。

这个经典段子在笔者看来，很有用 AOP 思路分析的价值。首先，我们的目的是**<font color=red>把大象放进冰箱里</font>**，这就是我们的业务所在。但是要放大象进去，**开冰箱门**和**关冰箱门**可以省略吗？不能。那这两者和塞大象的业务有关吗？没有。  
所以**<font color=red>与业务无关</font>**，但又必须做的工作（或者优化的工作），就是切面的意义所在了。缓存的加入，优化了数据的读取，但如果去掉了缓存，业务依旧可以正常工作，只是效率低一点而已。所以把缓存从业务代码中拿出来，就实现了解耦。

### 2. AOP 的代理思想

> 参考地址：  
> [《Spring AOP 的实现原理》](http://www.importnew.com/24305.html)  
> [《Spring service 本类中方法调用另一个方法事务不生效问题》](https:// blog.csdn.net/dapinxiaohuo/article/details/52092447)  

另外在该过程中，笔者也终于理解了**<font color=red>代理</font>**的意义。  
首先叙述一下问题：笔者有一次在 A 类的 a 方法上加入了后置切面方法后，用 A 类的 b 方法调用了自身的 a 方法，但多次测试发现怎么也不会进后置切面方法。经过好长时间的加班折腾，笔者终于发现了一个问题：**自身调用方法，是不会进入切面方法的**。  

AOP 的基本是使用代理实现的。通常使用的是 AspectJ 或者 Spring AOP 切面。  
AspectJ 使用静态编译的方式实现 AOP 功能。对于一个写好的类，对其编写 aspectj 脚本，然后对该 \*.java 文件进行编译指令，如 <code>ajc -d . Hello.java TxAspect.aj</code>，即可编译生成一个类，该类会比原先的类多一些内容，通过这种方式实现切面。

原始类：

```java
public class Hello {
    public void sayHello() {
        System.out.println("hello");
    }
 
    public static void main(String[] args) {
        Hello h = new Hello();
        h.sayHello();
    }
}
```

编写的 aspectj 语句：

```java
public aspect TxAspect {
    void around():call(void Hello.sayHello()){
        System.out.println("开始事务 ...");
        proceed();
        System.out.println("事务结束 ...");
    }
}
```

执行 aspectj 语句 <code>ajc -d . Hello.java TxAspect.aj</code> 编译后生成的类：

```java
public class Hello {
    public Hello() {
    }
 
    public void sayHello() {
        System.out.println("hello");
    }
 
    public static void main(String[] args) {
        Hello h = new Hello();
        sayHello_aroundBody1$advice(h, TxAspect.aspectOf(), (AroundClosure)null);
    }
}
```

Spring AOP 是通过**<font color=red>动态代理</font>**的形式实现的，其中又分为通过 **JDK 动态代理**，以及 **CGLIB 动态代理**。

- **JDK 动态代理**：使用反射原理，对实现了接口的类进行代理；
- **CGLIB 动态代理**：字节码编辑技术，对没有实现接口的类进行代理；

主要原因笔者后续也终于分析理解了：由于笔者虽然使用的是 @AspectJ 注解，但实际上使用的依旧是 Spring AOP。

如果使用 Spring AOP，使用过程中可能会出现一个问题：<font color=red>**自身调用切面注解方法，切面失效**</font>。这是因为 AOP 的实现是通过代理的形式实现的，所以自身调用方法不满足代理调用的条件，所以不会执行切面。切面的调用流程如下文链接所示，文中以事务出发，讲解了 AOP 的实现原理 (注：事务的实现原理也是切面)：

![Spring AOP 调用流程](http://dl.iteye.com/upload/attachment/0066/6245/a7d8d493-e387-34e9-a637-8a4d8d438602.jpg)

所以，对于笔者这种自身调用切面的情况，可以**<font color=red>改变方法的调用方式</font>**：改变调用自身方法的方式，使用调用**代理**方法的形式。笔者在 Spring 的 XML 中对 aop 进行配置：

```xml
<!—- 注解风格支持 -->
<aop:aspectj-autoproxy expose-proxy="true"/>
<!—- xml 风格支持 -->
<aop:config expose-proxy="true"/>
```

然后在方法中通过 Spring 的 AopContext.currentProxy 获取代理对象，然后通过代理调用方法。例如有自身方法调用如下：

```java
this.b();
```

变为：

```java
((AService) AopContext.currentProxy()).b();
```

笔者又开了一次脑洞，用娱乐圈明星和代理人之间的关系来类比理解了一下代理模式。作为一个代理人，目的是协助明星的工作。明星主要工作，就是唱，跳，RAP 之类的，而代理人，就是类似于在演出开始之前找厂商谈出场费，演出之后找厂商结账，买热搜，或者发个律师函之类的。总之不管好事儿坏事儿，代理干的事儿都贼 TM 操心，又和明星的演出工作没有直接的关系。  
数据库事务也是一样的道理。增删改查，是 SQL 语句关心的核心业务，SQL 语句只要按照语句执行就顺利完成了任务。由于事务的原子性，一个事务内的所有执行完毕后，**事务一起提交结果**。如果执行过程中出现了意外呢？那么**事务就把状态回滚到最开始的状态**。事务依旧做着处理后续工作，还有帮人擦屁股的工作，而且还是和业务本身没有关系的事儿，这和代理人是一样的命啊……  
这样，AOP 和代理思想，笔者用一头大象，还有一个明星经纪人的例子便顿悟了。

### 3. 分布式缓存问题（缓存雪崩，缓存穿透，缓存击穿）

> 参考地址：  
> [《缓存穿透，缓存击穿，缓存雪崩解决方案分析》](https://blog.csdn.net/zeb_perfect/article/details/54135506)  
> [《缓存穿透、缓存击穿、缓存雪崩区别和解决方案》](https://blog.csdn.net/kongtiao5/article/details/82771694)

好的，把缓存逻辑从业务代码逻辑揪了出来，后一个问题就解决了，现在解决前一个问题：将集群中所有服务的缓存从本地缓存转为分布式缓存，降低缓存在服务中占用的资源。  
由于业务组只有 Memcache 缓存集群，并没有搭起来 Redis，所以笔者还是选了 Memcache 作为分布式缓存工具。笔者用了一天时间封装了我们服务自己用的 MemcacheService，把初始化、常用的 get, set 方法封装完毕，测试也没有问题。由于具体过程只是对 Memcache 的 API 进行简单封装，故具体过程不表。但是进行到这里，笔者也只是简单的封装完毕，仍然有可以优化的空间。  
集群服务的缓存，有三大问题：**<font color=red>缓存雪崩、缓存穿透、缓存击穿</font>**。在并发量高的时候，这三个缓存问题很容易引起服务与数据库的宕机。虽然我们的小服务并不存在高并发的场景，但既然要做性能优化，就要尽量做到最好，所以笔者还是在我这小小的服务上事先了这几个缓存问题并加以解决。

#### (1) 缓存雪崩

缓存雪崩和缓存击穿都和分布式缓存的**缓存过期时间**有关。  
缓存雪崩，指的是对于某些热点缓存，如果都设置了**相同的过期时间**，在过期时间范围之内是正常的。但等到经过了这个过期时间之后，大量并发再访问这些缓存内容，会因为缓存内容已经过期而失效，从而大量并发短时间内涌向数据库，很容易造成数据库的崩溃。  
这样的情况发生的主要原因，在于热点数据设置了相同的过期时间。解决的方案是对这些热点数据设置**<font color=red>随机的过期时间</font>**即可。比如笔者在封装 Memcache 接口的参数中有过期时间 int expireTime，并设置了默认的过期时间为 30min，这样的缓存策略确实容易产生缓存雪崩现象。此后笔者在传入的 expireTime 值的基础上，由加上了一个 0~300 秒的随机值。这样所有缓存的过期时间都有了一定的随机性，从而避免了缓存雪崩现象。

#### (2) 缓存击穿

假设有某个热点数据，该数据在数据库中存在该值，但缓存中不存在，那么如果同一时间大量并发查询该缓存，则会由于缓存中不存在该数据，从而将大量并发释放，大量并发涌向数据库，容易引起数据库的宕机。  
看到这里也可以体会到，前面的缓存雪崩与缓存击穿有很大的相似性。缓存雪崩针对的是对**一批**在数据库中存在，但在缓存中不存在的数据；而缓存击穿针对的是**一个**数据。  
在[《缓存穿透，缓存击穿，缓存雪崩解决方案分析》](https://blog.csdn.net/zeb_perfect/article/details/54135506)一文中提到了四种方式，笔者采用了类似于第一种方式的解决方法：使用互斥锁。由于这里的环境是分布式环境，所以这里的互斥锁指的其实是**<font color=red>分布式锁</font>**。笔者又按照[《缓存穿透、缓存击穿、缓存雪崩区别和解决方案》](https://blog.csdn.net/kongtiao5/article/details/82771694)一文中的思路，以业务组的 Zookeeer 集群为基础实现了分布式锁，解决了缓存击穿的问题。伪代码如下：

```java
    public Object getData(String key) {
        // 1. 从缓存中读取数据
        Object result = getDataFromMemcache(key);
        // 2. 如果缓存中不存在数据，则从数据库中 (或者计算) 获取
        if (result == null) {
            InterProcessMutex lock = new InterProcessMutex(client, "/service/lock/test1");
            // 2.1 尝试获取锁
            try {
                if (lock.acquire(10, TimeUnit.SECONDS)) {
                    // ※ 2.1.1 尝试再次获取缓存，如果获取值不为空，则直接返回
                    result = getDataFromMemcache(key);
                    if (result != null) {
                        log.info("获取锁后再次尝试获取缓存，缓存命中，直接返回");
                        return result;
                    }

                    // 2.1.2 从数据库中获取原始数据 (或者计算获取得到数据)
                    result = queryData(key);

                    // 2.1.3 将结果存入缓存
                    setDataToMemcache(key, result);
                }
                // 2.2 获取锁失败，暂停短暂时间，尝试再次重新获取缓存信息
                else {
                    TimeUnit.MILLISECONDS.sleep(100);
                    result = getData(key);
                }
            } catch (Exception e) {
                e.printStackTrace();
            } 
            // 2.3 退出方法前释放分布式锁
            finally {
                if (lock != null && lock.isAcquiredInThisProcess()) {
                    lock.release();
                }
            }
        }
        
        return result;
    }
```

笔者解决缓存击穿的思路，是集群中服务如果同时处理大量并发，且尝试获取同一数据时，所有并发都会尝试获取 InterProcessMutex 的分布式锁。这里的 InterProcessMutex，是 Curator 自带的一个分布式锁，它基于 Zookeeper 的 Znode 实现了分布式锁的功能。在 InterProcessMutex 的传参中，需要传入一个 ZNode 路径，当大量并发都尝试获取这个分布式锁时，只有一个锁可以获得该锁，其他锁需要等待一定时间 (acquire 方法中传入的时间)。如果经过这段时间仍然没有获得该锁，则 acquire 方法返回 false。  

笔者解决缓存击穿的逻辑伪代码如上所示。逻辑比较简单，但其中值得一提的是，在 2.1.1 中，对于已经获取了分布式锁的请求，笔者又重新尝试获取一次缓存。这是因为 Memcache 缓存的存入与读取可能会**<font color=red>不同步</font>**的情况。假想一种情况：对于尝试获取分布式锁的请求 req1, req2，如果 req1 首先获取到了锁，且将计算的结果存入了 Memcache，然后 req2 在等待时间内又重新获取到了该锁，如果直接继续执行，也就会重新从数据库中获取一次 req1 已经获取且存入缓存的数据，这样就造成了重复数据的读取。所以需要在获取了分布式锁之后重新再获取一次缓存，判断在争抢分布式锁的过程中，缓存是否已经处理完毕。

#### (3) 缓存穿透

缓存穿透，指的是当数据库与缓存中都没有某数据时，该条数据就会成为漏洞，如果有人蓄意短时间内大量查询这条数据，大量连接就很容易穿透缓存涌向数据库，会造成数据库的宕机。针对这种情况，比较普遍的应对方法是使用**布隆过滤器 (Bloom Filter)**进行防护。

![布隆，来干活了！（弗雷尔卓德之心——Bloom Filter）](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1556687431324&di=751f296a74fabdedf75bc9b63d788e7b&imgtype=0&src=http%3A%2F%2Fi1.17173cdn.com%2F2fhnvk%2FYWxqaGBf%2Fcms3%2FPDuCdAbksCFdafl.jpg)

布隆过滤器和弗雷尔卓德之心有一些相似的地方，它的防御不是完全抵挡的，是不准确的。换句话说，**针对某条数据，布隆过滤器<font color=red>只保证在数据库中一定没有该数据，不能保证一定有这条数据</font>**。  
布隆过滤器的最大的好处是，判断简单，消耗空间少。通常如果直接使用 Map 访问结果来判断是否存在数据是否存在，虽然可以实现，但 Map 通常的内存利用率不会太高，对于几百万甚至几亿的大数据集，太浪费空间。而布隆过滤器本身是一个 bitmap 的结构（笔者个人理解基本是一个很大很大的 0-1 数组），初始状态下全部为 0。当有值存入缓存时，使用多个 Hash 函数分别计算对应 Key 值的结果，结果转换为 bitmap 指定的位数，对应位上置 1。这样，越来越多的值存入，bitmap 上也填充了越来越多的 1。  
这样如果有请求查询某个数据是否存在，则依旧利用相同的 Hash 函数计算结果，并在 bitmap 上查找计算结果的位置上是否全部为 1。**只要有一个位置不为 1，缓存中就必然没有该数据**。但是如果所有位置都为 1，那么也不能说明缓存中一定有这条数据。因为随着越来越多的数据存入缓存，布隆过滤器 bitmap 中的 1 值也越来越多，所以即使计算结果中所有位数的值都为 1，也有可能是其他若干计算结果将这些位置上的 1 给占据了。布隆过滤器虽然有误判率，但是有文章指出布隆过滤器的误判率在合适的参数设置之下会变得很低。具体可以见文章[《使用BloomFilter布隆过滤器解决缓存击穿、垃圾邮件识别、集合判重》](https://blog.csdn.net/tianyaleixiaowu/article/details/74721877)。  
除了不能判断数据库中一定存在某条数据之外，布隆过滤器还有一个问题，在于它不能删除某个值填充在 bitmap 中的结果。

笔者本来想用 guava 包中自带的 BloomFilter 来实现 Memcache 的缓存穿透防护，本来都已经研究好该怎么加入布隆的大盾牌了，但是后来一想，布隆过滤器应该是在 Memcache 端做的事情，而不是在我集群服务这里该做的。如果每个服务都建一个 BloomFilter，这几个过滤器的值肯定是不同步的，而且会造成大量的空间浪费，所以最后并没有付诸实践。

## 七、04.17—04.25 业务逻辑修正

与解决技术层面同步进行的，是对于业务逻辑的修正。修正的主要思路是调整消息订阅后的处理方式，以及方法、缓存的粒度调整（从粗粒度调整到细粒度）。涉及具体的业务逻辑，此处不表。


## 结语

经过一段长时间的奋战，我们的并发效率提升了二到三倍。  
但笔者并不是感觉我们做的很好，笔者更认为这是项目整个过程中的问题爆发。由于去年项目赶的太紧，三个月下来几乎天天 9107 的节奏，小伙伴们都累的没脾气，自然而然产生了抵触心理，代码质量与效率也自然下降。整个过程下来，堆积的坑越攒越多，最终到了某个时间不得不改。  
看着这些被修改的代码，有一部分确实都是自己的手笔，确实算是段悲伤的黑历史了。但历史已不再重要了，而是在这段解决问题的过程中积累学习的经验，是十分宝贵的。希望以后在工作中能够不再出现类似的问题吧。  

------------------------

本文于 2019.03.06 始，于 2019 五一劳动节终。