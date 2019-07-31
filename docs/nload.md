nload 监控网口流量
==============================
网络监控工具很多，但是一直想找针对指定网口流量的监控工具。


1. Overall bandwidth - nload, bmon, slurm, bwm-ng, cbm, speedometer, netload

2. Overall bandwidth (batch style output) - vnstat, ifstat, dstat, collectl

3. Bandwidth per socket connection - iftop, iptraf, tcptrack, pktstat, netwatch, trafshow

4. Bandwidth per process - nethogs

参考文档 [[https://www.binarytides.com/linux-commands-monitor-network/]](https://www.binarytides.com/linux-commands-monitor-network/)

# 安装
```
yum install nload iftop nethogs htop
```

# 查看整个系统的流量。dstat
```
[root@localhost ~]# dstat
You did not select any stats, using -cdngy by default.
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw
  0   0  99   0   0   0|  16k   22k|   0     0 |  14k   20k|  10k 8862
  1   0  99   0   0   0|   0     0 |2476k  476M|   0     0 |  22k   11k
  1   0  99   0   0   0|   0     0 |2236k  400M|   0     0 |  20k   11k
  0   0  99   0   0   0|   0  2968k|2112k  418M|   0     0 |  20k   11k
  1   0  99   0   0   0|   0     0 |2646k  499M|   0     0 |  24k   13k
  1   0  99   0   0   0|   0     0 |2494k  446M|   0     0 |  23k   11k
  1   0  99   0   0   0|   0     0 |2333k  445M|   0     0 |  22k   11k
  1   0  99   0   0   0|   0     0 |2890k  531M|   0     0 |  25k   11k
  1   0  99   0   0   0|   0     0 |2743k  481M|   0     0 |  24k   11k
[root@localhost ~]#

```

# 查看指定网络端口的流量。nload
```
nload -m
```
```
Device em1 [192.168.100.118] (1/9):
==============================================================================================================================
Incoming:                                                      Outgoing:
Curr: 2.12 kBit/s                                              Curr: 28.61 kBit/s
Avg: 4.32 kBit/s                                               Avg: 49.00 kBit/s
Min: 0.00 Bit/s                                                Min: 0.00 Bit/s
Max: 7.30 kBit/s                                               Max: 100.50 kBit/s
Ttl: 84.48 MByte                                               Ttl: 95.69 MByte

Device em2 (2/9):
==============================================================================================================================
Incoming:                                                      Outgoing:
Curr: 0.00 Bit/s                                               Curr: 0.00 Bit/s
Avg: 0.00 Bit/s                                                Avg: 0.00 Bit/s
Min: 0.00 Bit/s                                                Min: 0.00 Bit/s
Max: 0.00 Bit/s                                                Max: 0.00 Bit/s
Ttl: 0.00 Byte                                                 Ttl: 0.00 Byte

Device em3 (3/9):
==============================================================================================================================
Incoming:                                                      Outgoing:
Curr: 0.00 Bit/s                                               Curr: 0.00 Bit/s
Avg: 0.00 Bit/s                                                Avg: 0.00 Bit/s
Min: 0.00 Bit/s                                                Min: 0.00 Bit/s
Max: 0.00 Bit/s                                                Max: 0.00 Bit/s
Ttl: 0.00 Byte                                                 Ttl: 0.00 Byte

Device em4 (4/9):
==============================================================================================================================
Incoming:                                                      Outgoing:
Curr: 0.00 Bit/s                                               Curr: 0.00 Bit/s
Avg: 0.00 Bit/s                                                Avg: 0.00 Bit/s
Min: 0.00 Bit/s                                                Min: 0.00 Bit/s
Max: 0.00 Bit/s                                                Max: 0.00 Bit/s
Ttl: 0.00 Byte                                                 Ttl: 0.00 Byte
```

# 查看进程的流量 nethogs
```
NetHogs version 0.8.5

    PID USER     PROGRAM                         DEV        SENT      RECEIVED
 155017 root     fio                             p7p2    40193.922     269.434 KB/sec
 155035 root     fio                             p7p2    42799.801     249.772 KB/sec
 155065 root     fio                             p7p2    27634.619     180.794 KB/sec
 155057 root     fio                             p7p2    29825.311     165.916 KB/sec
 155079 root     fio                             p7p2    30595.211     162.005 KB/sec
 155009 root     fio                             p7p2    22149.711     134.591 KB/sec
 155059 root     fio                             p7p2     5550.278      32.793 KB/sec
 155069 root     fio                             p7p2     5945.441      31.159 KB/sec
 158413 root     sshd: root@pts/1                em1         4.339       0.245 KB/sec
 155027 root     fio                             p7p2        0.119       0.089 KB/sec

```


# 查看socket连接流量 iftop
也可以很很清楚的看出主机到主机的流量。
```

                         12.5Kb                   25.0Kb                   37.5Kb                   50.0Kb              62.5Kb
└────────────────────────┴────────────────────────┴────────────────────────┴────────────────────────┴─────────────────────────
localhost.localdomain:ssh                         => 115.171.85.202:51346                              32.1Kb  27.2Kb  23.1Kb
                                                  <=                                                   1.77Kb  1.38Kb   828b
localhost.localdomain:ssh                         => 192.168.100.12:41678                              2.28Kb  1.73Kb  2.09Kb
                                                  <=                                                    208b    208b    379b
255.255.255.255:bootps                            => 0.0.0.0:bootpc                                       0b      0b      0b
                                                  <=                                                      0b    266b     66b
localhost.localdomain:54269                       => public1.114dns.com:domain                            0b     59b     15b
                                                  <=                                                      0b     87b     22b
localhost.localdomain:33555                       => public1.114dns.com:domain                            0b      0b     13b
                                                  <=                                                      0b      0b     20b
```





# 

ubuntu

```
me@ubuntu:~$ ifstat -a
        lo             enahisic2i0         enahisic2i1         enahisic2i2         enahisic2i3
 KB/s in  KB/s out   KB/s in  KB/s out   KB/s in  KB/s out   KB/s in  KB/s out   KB/s in  KB/s out
    0.00      0.00      0.12      0.12      0.00      0.00      0.00      0.00      0.00      0.00
    0.00      0.00      0.24      0.11      0.00      0.00      0.00      0.00      0.00      0.00
    0.00      0.00      0.12      0.10      0.00      0.00      0.00      0.00      0.00      0.00
```