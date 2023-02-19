# 一文掌握 Linux 性能分析之 I/O 篇

这是 Linux 性能分析系列的第三篇，前两篇分别讲了 CPU 和 内存，本篇来看 IO。

IO 和 存储密切相关，存储可以概括为磁盘，内存，缓存，三者读写的性能差距非常大，磁盘读写是毫秒级的（一般 0.1-10ms），内存读写是微妙级的（一般 0.1-10us），cache 是纳秒级的（一般 1-10ns）。但这也是牺牲其他特性为代价的，速度快的，价格越贵，容量也越小。

IO 性能这块，我们更多关注的是读写磁盘的性能。首先，先了解下磁盘的基本信息。

## 磁盘基本信息

### fdisk

查看磁盘信息，包括磁盘容量，扇区大小，IO 大小等信息，常用 `fdisk -l` 查看

```bash
[root@zhimin serops]# fdisk -l

Disk /dev/sda: 85.9 GB, 85899345920 bytes, 167772160 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00045e8a

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048   167772159    83373056   8e  Linux LVM

Disk /dev/mapper/centos-root: 32.2 GB, 32208060416 bytes, 62906368 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 8589 MB, 8589934592 bytes, 16777216 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-apps: 44.6 GB, 44568674304 bytes, 87048192 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

可以看到 /dev/ 下有一个 80.9G 的硬盘，一共 167772160个扇区，每个扇区 512字节，IO 大小也是 512 字节。

###df

查看磁盘使用情况，通常看磁盘使用率

```bash
[root@zhimin serops]# df -h

Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   30G  7.1G   23G  24% /
devtmpfs                 4.8G     0  4.8G   0% /dev
tmpfs                    4.9G     0  4.9G   0% /dev/shm
tmpfs                    4.9G  479M  4.4G  10% /run
tmpfs                    4.9G     0  4.9G   0% /sys/fs/cgroup
/dev/mapper/centos-apps   42G  4.3G   38G  11% /apps
/dev/sda1                497M  125M  372M  26% /boot
10.10.2.140:/data/ECC    541G  242G  272G  48% /data/ECC
tmpfs                    984M     0  984M   0% /run/user/0
tmpfs                    984M     0  984M   0% /run/user/1002
```

## 磁盘性能分析
主要分析磁盘的读写效率（IOPS：每秒读写的次数；吞吐量：每秒读写的数据量），IO 繁忙程度，及 IO 访问对 CPU 的消耗等性能指标。

### vmstat

第一个较为常用的还是这个万能的 vmstat，`vmstat 1`为每隔1秒刷新一次；

```bash
[root@zhimin serops]# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0 117592 580920     28 1952392    0    0     0     2    0    0  0  0 99  0  0
 0  0 117592 581152     28 1952424    0    0     0     0  313  543  0  0 100  0  0
 0  0 117592 581028     28 1952424    0    0     0     4  305  520  0  0 100  0  0
 0  0 117592 581028     28 1952424    0    0     0     0  315  545  0  0 100  0  0
 0  0 117592 581152     28 1952424    0    0     0     0  311  549  0  0 100  0  0
 0  0 117592 581152     28 1952424    0    0     0     0  414  632  0  0 100  0  0
 0  0 117592 581028     28 1952428    0    0     0     0  460  716  0  0 99  0  0
 0  0 117592 581028     28 1952428    0    0     0     4  270  486  0  0 100  0  0
 0  0 117592 581152     28 1952428    0    0     0    24  323  542  0  0 100  0  0
 0  0 117592 580904     28 1952432    0    0     0     0  359  607  0  0 100  0  0
```
对于 IO，我们常关注三个部分：
* b 值：表示因为 IO 阻塞排队的任务数
* bi 和 bo 值：表示每秒读写磁盘的块数，bi（block in）是写磁盘，bo（block out）是读磁盘
* wa 值：表示因为 IO 等待（wait）而消耗的 CPU 比例

一般这几个值偏大，都意味着系统 IO 的消耗较大，对于读请求较大的服务器，b、bo、wa 的值偏大，而写请求较大的服务器，b、bi、wa 的值偏大。

### iostat

vmstat 虽然万能，但是它分析的东西有限，iostat 是专业分析 IO 性能的工具，可以方便查看 CPU、网卡、tty 设备、磁盘、CD-ROM 等等设备的信息，非常强大，总结下来，共有以下几种用法：

**1）**`iostat -c `查看部分 CPU 使用情况

```bash
[root@zhimin serops]# iostat -c
Linux 3.10.0-327.el7.x86_64 (ECC_Prod_2_203) 	06/24/2021 	_x86_64_	(8 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.47    0.00    0.23    0.00    0.00   99.30
```

这里显示的是多个 CPU 的平均值，每个字段的含义我就不多解释了，我一般会重点关注 `%iowait` 和 `%idle`。

- `%iowait` ：表示 CPU 等待 IO 完成时间的百分比

- `%idle`：表示CPU 空闲时间百分比

如果 %iowait 较高，则表明磁盘存在 IO 瓶颈，如果 %idle 较高，则 CPU 比较空闲，如果两个值都比较高，则有可能 CPU 在等待分配内存，瓶颈在内存，此时应该加大内存，如果 %idle 较低，则此时瓶颈在 CPU，应该增加 CPU 资源。

**2）**`iostat -d` 查看磁盘使用情况，主要是显示 IOPS 和吞吐量信息（-k : 以 KB 为单位显示，-m：以 M 为单位显示）

```bash
[root@zhimin serops]# iostat -d
Linux 3.10.0-327.el7.x86_64 (ECC_Prod_2_203) 	06/24/2021 	_x86_64_	(8 CPU)

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
fd0               0.00         0.00         0.00          4          0
sda               1.91         0.18        16.04    7031355  637851973
dm-0              1.75         0.16        14.81    6251049  588911652
dm-1              0.02         0.02         0.07     603084    2616684
dm-2              0.23         0.00         1.17     147801   46321556
```

其中，几个参数分别解释如下：

- `tps`：设备每秒的传输次数（transfers per second），也就是读写次数。
- `kB_read/s`：每秒读磁盘的数据量
- `kB_wrtn/s`：每秒写磁盘的数据量
- `kB_read`：读取磁盘的数据总量
- `kB_wrtn`：写入磁盘的数据总量

**3）**`iostat -x` 查看磁盘详细信息

```bash
[root@zhimin serops]# iostat -x
Linux 3.10.0-327.el7.x86_64 (ECC_Prod_2_203) 	06/24/2021 	_x86_64_	(8 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.47    0.00    0.23    0.00    0.00   99.30

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
fd0               0.00     0.00    0.00    0.00     0.00     0.00     8.00     0.00   63.00   63.00    0.00  63.00   0.00
sda               0.00     0.09    0.01    1.90     0.18    16.04    17.02     0.00    0.54    3.42    0.53   0.32   0.06
dm-0              0.00     0.00    0.01    1.74     0.16    14.81    17.15     0.00    0.58    3.34    0.57   0.31   0.05
dm-1              0.00     0.00    0.00    0.02     0.02     0.07     8.00     0.00    8.27    3.77    9.31   0.15   0.00
dm-2              0.00     0.00    0.00    0.23     0.00     1.17    10.08     0.00    0.32    3.66    0.31   0.32   0.01
```

其中，几个参数解释如下：
- `rrqm/s` 和 `wrqm/s`：分别每秒进行合并的读操作数和写操作数，这是什么意思呢，合并就是说把多次 IO 请求合并成少量的几次，这样可以减小 IO 开销，buffer 存在的意义就是为了解决这个问题的。
- `r/s` 和 `w/s`：每秒磁盘读写的次数。这两个值相加就是 tps。
- `rkB/s` 和 `wkB/s`：每秒磁盘读写的数据量，这两个值和上面的 kB_read/s、kB_wrnt/s 是一个意思。
- `avgrq-sz`：平均每次读写磁盘扇区的大小。
- `avgqu-sze`：平均 IO 队列长度。队列长度越短越好。
- `await`：平均每次磁盘读写的等待时间（ms）。
- `svctm`：平均每次磁盘读写的服务时间（ms）。
- `%util`：一秒钟有百分之多少的时间用于磁盘读写操作。

以上这些参数太多了，我们并不需要每个都关注，可以重点关注两个：

* 1）`%util：衡量 IO 的繁忙程度`

这个值越大，说明产生的 IO 请求较多，IO 压力较大，我们可以结合 %idle 参数来看，如果 %idle < 70% 就说明 IO 比较繁忙了。也可以结合 vmstat 的 b 参数（等待 IO 的进程数）和 wa 参数（IO 等待所占 CPU 时间百分比）来看，如果 wa > 30% 也说明 IO 较为繁忙。

* 2）`await：衡量 IO 的响应速度`

通俗理解，await 就像我们去医院看病排队等待的时间，这个值和医生的服务速度（svctm）和你前面排队的人数（avgqu-size）有关。如果 svctm 和 await 接近，说明磁盘 IO 响应时间较快，排队较少，如果 await 远大于 svctm，说明此时队列太长，响应较慢，这时可以考虑换性能更好的磁盘或升级 CPU。

**4）**`iostat -dxkt 1 2`间隔刷新查看磁盘使用情况和磁盘详细信息，1定时1s显示，2显示2条信息，iostat -dxkt interval count

```bash
[root@zhimin serops]# iostat -dxkt 1 2
Linux 3.10.0-327.el7.x86_64 (ECC_Prod_2_203) 	06/24/2021 	_x86_64_	(8 CPU)

06/24/2021 04:14:31 PM
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
fd0               0.00     0.00    0.00    0.00     0.00     0.00     8.00     0.00   63.00   63.00    0.00  63.00   0.00
sda               0.00     0.09    0.01    1.90     0.18    16.04    17.02     0.00    0.54    3.42    0.53   0.32   0.06
dm-0              0.00     0.00    0.01    1.74     0.16    14.81    17.15     0.00    0.58    3.34    0.57   0.31   0.05
dm-1              0.00     0.00    0.00    0.02     0.02     0.07     8.00     0.00    8.27    3.77    9.31   0.15   0.00
dm-2              0.00     0.00    0.00    0.23     0.00     1.17    10.08     0.00    0.32    3.66    0.31   0.32   0.01

06/24/2021 04:14:32 PM
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
fd0               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
sda               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-0              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-2              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
```

## 进程 IO 性能分析

有了以上两个命令，基本上能对磁盘 IO 的信息有个全方位的了解了。但如果要确定具体哪个进程的 IO 开销较大，这就得借助另外的工具了。

`iotop`这个命令类似 top/htop，可以显示每个进程的 IO 情况，有了这个命令，就可以定位具体哪个进程的 IO 开销比较大了。

```bash
Total DISK READ :	0.00 B/s | Total DISK WRITE :       0.00 B/s
Actual DISK READ:	0.00 B/s | Actual DISK WRITE:       0.00 B/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND                                                                        
    1 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % systemd --switched-root --system --deserialize 21
    2 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [kthreadd]
    3 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % [ksoftirqd/0]
```

最后还是总结下，`fdisk -l` 和 `df -h` 查看磁盘基本信息，`iostat -dxkt 1 2` 查看磁盘 IOPS 和吞吐量，`iostat -x` 结合 `vmstat` 查看磁盘的繁忙程度和处理效率。



[参考]

* https://segmentfault.com/a/1190000018499770

