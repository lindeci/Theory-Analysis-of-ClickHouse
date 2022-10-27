- [收集系统信息](#收集系统信息)
- [xargs](#xargs)
- [sed](#sed)
- [vi](#vi)
- [bash](#bash)
- [网络调优：](#网络调优)
- [mdadm 删除 软件 RAID](#mdadm-删除-软件-raid)
- [vgextend](#vgextend)
- [lsof](#lsof)
- [cpu开启性能模式](#cpu开启性能模式)
- [查看透明页](#查看透明页)
- [lvs操作](#lvs操作)
- [硬件时间](#硬件时间)
- [拆掉raid](#拆掉raid)
- [查看是否机械硬盘](#查看是否机械硬盘)
- [查找对应线程所占的 cpu核](#查找对应线程所占的-cpu核)

# 收集系统信息
```sh
cd /tmp
localip=`ip a show up| grep 'inet '| egrep -v 'scope host|127.0.0|secondary| lo'|head -1|awk '{print $2}'| sed 's/\/.*//g'` && echo $localip
cat << "SYSEOF" |  awk -F '\n' '{if(!NF ){next}}{print "echo;echo;echo;echo;echo;echo '\''########  "$NF"'\'';"$NF}' > sysinfosh
cat /etc/redhat-release;cat /etc/system-release-cpe;uname -a;hostname;hostname -i
cat /etc/hosts
ip a;ifconfig

uptime

lscpu
cat /proc/cpuinfo

df -Th
lsblk
lsblk -ap -o NAME,KNAME,PKNAME,TYPE,FSTYPE,MOUNTPOINT,RA,RO,RM,SIZE,STATE,MODE,ALIGNMENT,MIN-IO,OPT-IO,PHY-SEC,LOG-SEC,ROTA,SCHED,RQ-SIZE,RAND,HCTL,TRAN,REV,VENDOR,MODEL
lsblk -a -o NAME,KNAME,MOUNTPOINT,ROTA,SIZE,ALIGNMENT,MIN-IO,OPT-IO,PHY-SEC,LOG-SEC,RA,RO,RM

pvdisplay
vgdisplay
lvdisplay

free -h
cat /proc/buddyinfo
cat /proc/meminfo
cat /proc/slabinfo
cat /proc/vmallocinfo
cat /proc/vmstat
cat /proc/zoneinfo
cat /proc/pagetypeinfo
numactl -H

cat /proc/cmdline


cat /etc/sysctl.conf
sysctl -a 


cat /etc/crontab
crontab -u root -l
crontab -u tdsql -l

egrep "^[^#]" /etc/security/limits.conf  /etc/security/limits.d/*
ulimit -Ha
ulimit -Sa


ls -l  /proc
find /proc -maxdepth 1  -type f -size -1k
cat /proc/fb
cat /proc/dma
cat /proc/keys
#cat /proc/kmsg
cat /proc/misc
cat /proc/mtrr
cat /proc/stat
cat /proc/iomem
cat /proc/locks
cat /proc/swaps
cat /proc/crypto
cat /proc/mdstat
cat /proc/uptime
cat /proc/vmstat
cat /proc/cgroups
cat /proc/cmdline
cat /proc/cpuinfo
cat /proc/devices
cat /proc/ioports
cat /proc/loadavg
cat /proc/meminfo
cat /proc/modules
cat /proc/version
cat /proc/consoles
#cat /proc/kallsyms
cat /proc/slabinfo
cat /proc/softirqs
cat /proc/zoneinfo
cat /proc/buddyinfo
cat /proc/diskstats
cat /proc/key-users
cat /proc/schedstat
cat /proc/interrupts
#cat /proc/kpagecount
#cat /proc/kpageflags
cat /proc/partitions
cat /proc/timer_list
cat /proc/execdomains
cat /proc/filesystems
#cat /proc/sched_debug
cat /proc/timer_stats
cat /proc/vmallocinfo
cat /proc/pagetypeinfo
#cat /proc/sysrq-trigger

SYSEOF
cat sysinfosh
bash sysinfosh &>  sysinfoout$localip     #里面命令里面不要出现单引号
ls -l sysinfoout$localip 
less sysinfoout$localip
```
----------------------------------------------------------------------------
# xargs 
```sh
ps aux | grep mysqld | grep -v grep | awk '{print $2}' | xargs -I{} echo {}
```
----------------------------------------------------------------------------
#egrep

----------------------------------------------------------------------------
# sed
```sh
###sed非常规语法
#使用sed命令在cisco行下面添加CCIE；
sed -i "/cisco/a\CCIE" 123.txt

#使用sed命令在network行上面添加一行，内容是Security；
sed -i "/network/i\Security" 123.txt

sed -i -r  's~\r$~~g' 123.txt         #\r\n 换 \n
sed -i -r  's~$~\r~g' 123.txt         #\n换\r\n

一、删除包含匹配字符串的行
## 删除包含baidu.com的所有行
sed -i '/baidu.com/d' domain.file

二、删除匹配行及后所有行
## 删除匹配20160229的行及后面所有行
sed -i '/20160229/,$d' 充值人数.log

三、删除最后3行
tac file|sed 1,3d|tac
```
----------------------------------------------------------------------------
# vi
```sh
:set ff  查看当前文本的模式类型，一般为dos,unix
:set ff=dos  设置为dos模式， 也可以用 sed -i 's/$/\r/' 
:set ff=unix  设置为unix模式，也可以用一下方式转换为unix模式:sed -i 's/.$//g'
 
:set fileencoding查看现在文本的编码
:set fenc=编码  转换当前文本的编码为指定的编码
:set enc=编码  以指定的编码显示文本，但不保存到文件中。
:set nu   显示行号
:set encoding=utf-8 #设置编码格式
:set showmatch   在vi中输入），}时，光标会暂时的回到相匹配的（，{ （如果没有相匹配的就发出错误信息的铃声），编程时很有用

vim取消自动换行
把textwidth调大：
:set textwidth=1000
vim取消自动折行
:set nowrap

Vim快速移动
1、 需要按行快速移动光标时，可以使用键盘上的编辑键Home，快速将光标移动至当前行的行首。除此之外，也可以在命令模式中使用快捷键"^"（即Shift+6）或0（数字0）。
2、 如果要快速移动光标至当前行的行尾，可以使用编辑键End。也可以在命令模式中使用快捷键"$"（Shift+4）。与快捷键"^"和0不同，快捷键"$"前可以加上数字表示移动的行数。例如使用"1$"表示当前行的行尾，"2$"表示当前行的下一行的行尾。
3、h, j, k, l分别代表向左、下、上、右移动。如同许多vim命令一样，可以在这些键前加一个数字，表示移动的倍数。例如，"10j"表示向下移动10行；"10l"表示向右移动10列。
4、在vim中翻页，同样可以使用PageUp和PageDown，不过，像使用上下左右光标一样，你的手指会移出主键盘区。因此，我们通常使用CTRL-B和CTRL-F来进行翻页，它们的功能等同于PageUp和PageDown。CTRL-B和CTRL-F前也可以加上数字，来表示向上或向下翻多少页。
5、命令"gg"移动到文件的第一行，而命令"G"则移动到文件的最后一行。命令"G"前可以加上数字，在这里，数字的含义并不是倍数，而是你打算跳转的行号。例如，你想跳转到文件的第1234行，只需输入"1234G"。
你还可以按百分比来跳转，例如，你想跳到文件的正中间，输入"50%"；如果想跳到75%处，输入"75%"。注意，你必须先输入一个数字，然后输入"%"。如果直接输入"%"，那含义就完全不同了。":help N%"阅读更多细节。在文件中移动，你可能会迷失自己的位置，这时使用"CTRL-G"命令，查看一下自己位置。
6、"f"命令移动到光标右边的指定字符上，例如，"fx"，会把移动到光标右边的第一个'x'字符上。"F"命令则反方向查找，也就是移动到光标左边的指定字符上。
"t"命令和"f"命令的区别在于，它移动到光标右边的指定字符之前。例如，"tx"会移动到光标右边第一个'x'字符的前面。"T"命令是"t"命令的反向版本，它移动到光标左边的指定字符之后。
```
----------------------------------------------------------------------------
# bash
```sh
ctrl+a  行首
ctrl+e  行尾
ctrl+k  清除光标后至行尾的内容
ctrl+u  清除光标前至行首间的所有内容
ctrl+l  清屏，相当于clear
ctrl+z  把当前进程转到后台运行，使用 fg 命令恢复。比如top -d1 然后ctrl+z，到后台，然后fg,重新恢复
```
----------------------------------------------------------------------------
# 网络调优：
```sh
https://baijiahao.baidu.com/s?id=1677502435760515122&wfr=spider&for=pc
```
----------------------------------------------------------------------------
# mdadm 删除 软件 RAID
```sh
https://blog.csdn.net/u010953692/article/details/108858680
1、查看
	mdadm -D /dev/md0
	cat /proc/mdstat
	cat /etc/mdadm/mdadm.conf
	cat /etc/mdadm/mdadm.conf
2、umount /chain
3、	mdadm -S /dev/md0
	lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
	cat /etc/mdadm/mdadm.conf
	cat /dev/null > /etc/mdadm/mdadm.conf
4、删除元数据
# mdadm --zero-superblock /dev/sda
# mdadm --zero-superblock /dev/sdb
# mdadm --zero-superblock /dev/sdc
# mdadm --zero-superblock /dev/sdd
# lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
```
----------------------------------------------------------------------------
# vgextend
```sh
 3321  2022-07-18 11:06:29 root pvcreate -f /dev/sda
 3322  2022-07-18 11:08:41 root vgdisplay
 3323  2022-07-18 11:08:49 root vgs
 3324  2022-07-18 11:08:50 root pvs
 3325  2022-07-18 11:09:25 root disk -l
 3326  2022-07-18 11:09:27 root fdisk -l
 3327  2022-07-18 11:10:20 root pvs
 3328  2022-07-18 11:13:52 root dd if=/dev/zero of=/dev/sda bs=512K count=20
 3329  2022-07-18 11:14:02 root pvcreate -f /dev/sda
 3330  2022-07-18 11:14:19 root dd if=/dev/zero of=/dev/sdb bs=512K count=20
 3331  2022-07-18 11:14:25 root dd if=/dev/zero of=/dev/sdc bs=512K count=20
 3332  2022-07-18 11:14:30 root dd if=/dev/zero of=/dev/sdd bs=512K count=20
 3333  2022-07-18 11:14:36 root dd if=/dev/zero of=/dev/sde bs=512K count=20
 3334  2022-07-18 11:14:41 root pvcreate -f /dev/sdb
 3335  2022-07-18 11:14:43 root pvcreate -f /dev/sdc
 3336  2022-07-18 11:14:46 root pvcreate -f /dev/sdd
 3337  2022-07-18 11:14:55 root pvcreate /dev/sde
 3338  2022-07-18 11:14:57 root pvs
 3339  2022-07-18 11:15:19 root history |grep dd
 3340  2022-07-18 11:15:23 root history 
 3341  2022-07-18 11:16:27 root lvs
 3342  2022-07-18 11:16:34 root pvs
 3343  2022-07-18 11:17:09 root vgcreate vgtidb /dev/sda
 3344  2022-07-18 11:18:10 root vgextend vgtidb /dev/sdb 
 3345  2022-07-18 11:18:18 root pvdisplay
 3346  2022-07-18 11:18:39 root vgdisplay
 3347  2022-07-18 11:19:14 root vgextend vgtidb /dev/sdc 
 3348  2022-07-18 11:19:20 root vgextend vgtidb /dev/sdd
 3349  2022-07-18 11:19:25 root vgextend vgtidb /dev/sde
 3350  2022-07-18 11:19:32 root vgdisplay 
 3351  2022-07-18 11:23:26 root lvcreate -L 2.4T -n lvtikv vgtidb
 3352  2022-07-18 11:23:40 root lvdisplay
 3353  2022-07-18 11:24:28 root lvcreate -L 1.6T -n lvtiflash vgtidb
 3354  2022-07-18 11:24:33 root lsblk
 3355  2022-07-18 11:24:48 root df -h
 3356  2022-07-18 11:26:33 root cd /dev/mapper/
 3357  2022-07-18 11:26:35 root ll -rtc
 3358  2022-07-18 11:27:38 root cd 
 3359  2022-07-18 11:27:55 root mkdir /tidb
 3360  2022-07-18 11:28:11 root mkdir /tiflash
 3361  2022-07-18 11:29:38 root mkfs -t ext4 /dev/mapper/vgtidb-lvtikv 
 3362  2022-07-18 11:31:57 root mount -a /dev/mapper/vgtidb-lvtikv /tidb
 3363  2022-07-18 11:31:59 root df -h
 3364  2022-07-18 11:34:48 root lsblk -f
 3365  2022-07-18 11:35:45 root vim /etc/fstab 
 3366  2022-07-18 11:36:58 root cat /etc/fstab 
 3367  2022-07-18 11:41:52 root vim /etc/fstab 
 3368  2022-07-18 11:42:56 root mount -a
 3369  2022-07-18 11:43:32 root mkfs -t ext4 /dev/mapper/vgtidb-lvtiflash 
 3370  2022-07-18 11:44:30 root mount -a
 3371  2022-07-18 11:44:33 root df -h
 3372  2022-07-18 11:45:11 root umount
 3373  2022-07-18 11:45:19 root umount /tidb 
 3374  2022-07-18 11:45:22 root df -h
 3375  2022-07-18 11:45:28 root umount /tidb 
 3376  2022-07-18 11:45:30 root df -h
 3377  2022-07-18 11:45:36 root vim /etc/fstab 
 3378  2022-07-18 11:46:13 root mount -a
 3379  2022-07-18 11:46:16 root df -h
 3380  2022-07-18 11:46:26 root lsblk
 3381  2022-07-18 13:54:48 root cd /dev/mapper/
 3382  2022-07-18 13:54:48 root ls
 3383  2022-07-18 13:54:53 root ll -rtc
 3384  2022-07-18 13:54:59 root cd ..
 3385  2022-07-18 13:55:01 root ll -rtc
 3386  2022-07-18 13:55:03 root cd 
 3387  2022-07-18 13:55:08 root cd /dev/
 3388  2022-07-18 13:55:09 root ll -rtc
 3389  2022-07-18 13:55:45 root cd 
 3390  2022-07-18 13:55:47 root fdisk -l
 ```
----------------------------------------------------------------------------
# lsof
```sh
lsof -i tcp:31487
```
----------------------------------------------------------------------------
# cpu开启性能模式
```sh
cpupower frequency-info -o
```
----------------------------------------------------------------------------
# 查看透明页
```sh
cat /sys/kernel/mm/transparent_hugepage/
```
----------------------------------------------------------------------------
# lvs操作
```sh
ipvsadm –A –t <VIP>:<Port> -s <schedule: rr|wrr|lc|wlc|lblc|lblcr|dh|sh|sed|nq>
添加
ipvsadm -A -t 88.4.38.86:15004 -s sed
ipvsadm -a -r 88.4.38.76:15004 -t 88.4.38.86:15004 -w 10000 -i
ipvsadm -a -r 88.4.38.77:15004 -t 88.4.38.86:15004 -w 10000
ipvsadm -a -r 88.4.38.78:15004 -t 88.4.38.86:15004 -w 10000

ipvsadm -a -r 88.4.38.78:15003 -t 88.4.38.86:15003

删除
ipvsadm -d -r 88.4.38.76:15004 -t 88.4.38.86:15004
ipvsadm -d -r 88.4.38.77:15004 -t 88.4.38.86:15004
ipvsadm -d -r 88.4.38.78:15004 -t 88.4.38.86:15004
ipvsadm -D -t 88.4.38.86:15004

ipvsadm -A -t 88.4.38.86:15003 -s sed
ipvsadm -a -r 88.4.38.76:15003 -t 88.4.38.86:15003 -w 10000
ipvsadm -a -r 88.4.38.77:15003 -t 88.4.38.86:15003 -w 10000
ipvsadm -a -r 88.4.38.78:15003 -t 88.4.38.86:15003 -w 10000

ipvsadm -d -r 88.4.38.76:15003 -t 88.4.38.86:15003
ipvsadm -d -r 88.4.38.77:15003 -t 88.4.38.86:15003
ipvsadm -d -r 88.4.38.78:15003 -t 88.4.38.86:15003

ipvsadm -D -t 88.4.38.86:15002
```
----------------------------------------------------------------------------
# 硬件时间
```sh
利用系统时间设置硬件时间来
hwclock --systohc
```
----------------------------------------------------------------------------
# 拆掉raid
```sh
umount /data1
mdadm -S /dev/md0
mdadm --zero-superblock /dev/sda
mdadm --zero-superblock /dev/sdb
```
----------------------------------------------------------------------------
# 查看是否机械硬盘
```sh
#是否机械硬盘
egrep -Hi '' /sys/block/*/queue/rotational
lsblk -d -o name,rota
lsblk -a -o NAME,KNAME,MOUNTPOINT,ROTA,SIZE,ALIGNMENT,MIN-IO,OPT-IO,PHY-SEC,LOG-SEC,RA,RO,RM

fdisk -l
#"heads"（磁头），"track"（磁道）和"cylinders"（柱面）
smartctl --all /dev/sda
smartctl -A /dev/sda
```
----------------------------------------------------------------------------
# 查找对应线程所占的 cpu核
```sh
taskset -pc  PID

首先 可以通过  top  -H   -d  1  -p  PID 查看具体 进程的 cpu ，内存 等等 占据大小 比例
再 按下 1
可以查看到 cpu的占用比例，多少个核在使用 就可以看到多少个 %Cpu
```
----------------------------------------------------------------------------

