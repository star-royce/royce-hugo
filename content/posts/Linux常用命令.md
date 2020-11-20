---
title: "Linux常用命令"
date: 2020-11-20
slug: "linux order"
draft: false
tags:
- TECH
- Linux
categories:
- Linux
---



## 主机名

1. 查看主机名， `hostname` 
2. 修改主机名， `hostname xxx` 

## 查看进程

- `ps -aux` 

- - **STAT:** S表示正在休眠；s表示主进程；Z表示僵尸进程。
  - ![image-20201120101422075](https://i.loli.net/2020/11/20/71nHCMkTvbVlaR3.png)

- `ps -elf` 

- - ![image-20201120101431042](https://i.loli.net/2020/11/20/duyrvNmci97WqsV.png)

- `lsof -i:8080` 

- - 根据端口查看进程pid

## 高cpu占用率的进程和线程

1. top

1. 1. 实时显示系统中各个进程的资源占用状况，类似于Windows的任务管理器。
   2. 内容主要关注点

1. 1. 1. **进程的状态(S)**，其中S表示休眠，R表示正在运行，Z表示僵死状态，N表示该进程优先值是负数
      2. %CPU：该进程占用的CPU使用率。
      3. %MEM：该进程占用的物理内存和总内存的百分比。
      4. pid: 进程号

1. 1. 图示

1. 1. 1. ![image-20201120101440442](https://i.loli.net/2020/11/20/XUqmnJ8cekQ5ME3.png)

1. 1. 加上-H参数，则可以查看线程情况。 `top -H` 

1. ps

1. 1. 也可以用  `ps -eo pid,pcpu | sort -n -k 2 ` 查看进程cpu使用情况
   2. 同样的，也可以用 `ps H -eo pid,tid,pcpu | sort -n -k 3` 查看线程cpu使用情况
   3. 查看该进程下所有的线程  `ps -efL | grep pid` 

1. cat

1. 1. 查看线程的详细信息： `cat /proc/进程号/task/线程号/status` 

1. vmstat

1. 1. 大多跟top重叠，主要看下system这一栏，in线程中断，cs线程上下文切换是否有异常，还有io这一栏。对top是一个非常好的补充。



## 查看系统负载

1. 实时负载

1. 1. 使用w命令或者uptime命令查看系统负载
   2. top可以看， `load averge` 就是，代表 5、10、15分钟的负载

1. 历史负载

1. 1. `sar -q -f /var/log/sa/sa22`  #查看22号的系统负载

## 查看端口

1. 都开了哪些端口？

1. 1. `netstat -lnp` 或者 `netstat -a` 
   2. ![image-20201120101449500](https://i.loli.net/2020/11/20/sGTpQOJv2fH4b9M.png)

1. 网络连接状态

1. 1. `netstat -an` 
   2. ![image-20201120101512109](https://i.loli.net/2020/11/20/jehL7xPbcpWEkuC.png)

## 内存查看



1. 查看内存使用量和交换区使用量

1. 1. `free -m` 

1. 查看内存总量

1. 1. `grep MemTotal /proc/meminfo` 

1. 查看空闲内存量

1. 1. `grep MemFree /proc/meminfo` 

## 磁盘查看

1. 查看各分区使用情况

1. 1. `df -h`    

1. 查看指定目录的大小

1. 1. `du -sh <目录名>` 
   2. 或者直接在要查看的目录下执行，`du -sh`

## 定时脚本

1. 利用linux crontab 命令

1. 1. [基础概念](https://www.runoob.com/w3cnote/linux-crontab-tasks.html)， `定时纬度为 分 时 日 月 年` 

1. 1. 1. 每分钟执行一次， ``*/1 * * * * {{需执行脚本路径/名}}`
      2. 采用此语法写好文件 `crontest.cron`

1. 1. 1. 1. ![image-20201120101522270](https://i.loli.net/2020/11/20/jvq3JrUBGMAQy6W.png)

1. 1. 加入linux crontab调度队列， `crontab crontest.cron` 
   2. 查看crontab调度队列， `crontab -l` 
   3. 编辑调度队列， `crontab -e` 

## Linux环境抓包

1. `tcpdump`  

1. 1. 只过滤出访问http服务的，目标ip为192.168.0.111，一共抓1000个包，并且保存到1.cap文件中
   2. `tcpdump -nn -s0 host 192.168.0.111 and port 80 -c 1000 -w 1.cap` 
   3. -s0 为网卡

## 文本内容替换

1. vim 直接修改
2. perl命令替换

1. 1. `perl -p -i.bak -e 's/\bfoo\b/bar/g' *.c` ，将所有C程序中的foo替换成bar，旧文件备份成.bak。(原地替换文件，并将旧文件用指定的扩展名备份。不指定扩展名则不备份。)
   2. `perl -p -i -e "s/shan/hua/g" ./lishan.txt ./lishan.txt.bak` , 将当前文件夹下lishan.txt和lishan.txt.bak中的“shan”都替换为“hua”。(-e  执行指定的脚本。可以用正则表达式)

1. sed命令下批量替换文件内容  ，支持正则表达式

1. 1. `sed -i "s/shan/hua/g"  lishan.txt` ，把当前目录下lishan.txt里的shan都替换为hua

1. 1. 1. -i 表示inplace edit，就地修改文件
      2. -r 表示搜索子目录
      3. -l 表示输出匹配的文件名

## AWK命令使用

1. 概念: 简单来说awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理。
2. 举个栗子，每行按空格或TAB分割，输出文本中的1、4项

1. 1. `awk ‘{print $1,$4}’ log.txt` 

## 查找文件内容

### 1. 查看test1.txt文本内容

1. 1. 输入: `cat test1.txt` 
   2. 输出

1. 1. 1. ![image-20201120101530989](https://i.loli.net/2020/11/20/eRmaM3Tj6ntWCgl.png)

### 2. 从文本中匹配包含某段字符串的记录

1. 1. 输入: `grep -i "teacher" test1.txt` 
   2. 输出

1. 1. 1. ![image-20201120101539873](https://i.loli.net/2020/11/20/luToQsOCVaZrdS9.png)

## 访问权限

> 用数字添加权限, r=4，w=2，x=1 。
>
> a,b,c各为一个数字，分别代表User、Group、及Other的权限。
>
> 若要同时设置 rwx (可读写运行） 权限则将该权限位 设置 为 4 + 2 + 1 = 7
>
> 若要同时设置 rw- （可读写不可运行）权限则将该权限位 设置 为 4 + 2 = 6
>
> 若要同时设置 r-x （可读可运行不可写）权限则将该权限位 设置 为 4 +1 = 5

```
#权限范围：u(拥有者)g(郡组)o(其它用户)， 权限代号：r(读权限/4)w(写权限/2)x(执行权限/1)
#给文件拥有者增加test.sh的执行权限
chmod u+x test.sh
#给文件拥有者增加test目录及其下所有文件的执行权限
chmod u+x -R test

# 设置所有人可以读写及执行
chmod 777 test.sh
# 仅拥有者可读写
chmod 600 test.sh
```

## 压缩和解压

```
#打包test目录为test.tar.gz文件，-z表示用gzip压缩
tar -zcvf test.tar.gz ./test
#解压test.tar.gz文件
tar -zxvf test.tar.gz
#在不解压tar包的前提下，查看包里的内容
#‘t’(显示内容)，‘v’（详细报告tar处理的文件信息），‘f’（使用档案文件或者设备）
tar -tvf
```

## 光标移动

1. 左右上下，对应hlkj
2. 插入

1. 1. i --> 在光标处插入
   2. a --> 在光标下一个位置插入
   3. o --> 在光标所在行的下一行插入

1. 删除

1. 1. x --> 删除光标所在位置的字符
   2. dw --> 删除光标所在位置的一个单词
   3. d0 --> 删除从首字符到当前光标位置的字符
   4. D(shift + d) --> 删除从当前光标位置到行末尾的字符
   5. dd --> 删除整行（实际上是做了剪切）
   6. u --> 撤销

1. 复制

1. 1. yy --> 复制当前行
   2. nyy --> 复制从当前行开始的n行

1. 粘贴

1. 1. p --> 粘贴在当前光标所在行的下一行
   2. P --> 粘贴在当前光标所在行的上一行



## 分层复制(文件备份)

`cpio` 命令可以把文件按其目录结构给归档起来，同时也提供了解压功能

1. 将/etc下的所有普通文件都备份到/opt/etc.cpio

1. 1. `find /etc –type f | cpio –ocvB >/opt/etc.cpio` 

1. 将备份包还原到相应的位置，如果有相同文件进行覆盖

1. 1. `cpio –icduv < /opt/etc.cpio` 

