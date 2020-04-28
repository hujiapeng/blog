1.  Linux启动过程中也会通过一些Shell脚本执行命令
  * BIOS：BIOS系统在主板上，是一个输入输出系统，可以引导用户选择系统启动方式，一个是硬盘，另一个是光盘。(启动时候按F2，选择不同引导介质)
  * MBR：硬盘引导，引导启动grub软件
  * BootLoader(grub)：引导选择内核版本
  * kernel：内核启动并初始化，加载硬件
  * systemd：此刻Shell脚本开始工作，如果是CentOS7那么运行的第一个程序就是systemd，如果是CentOS6那第一个程序就是init
  * 系统初始化：第一个程序就是一号进程，来加载运行一些驱动程序
  * shell：执行一些shell脚本
2. grub2相关操作
```
[root@ittutu ~]# cd /boot/grub2
[root@ittutu grub2]# ls
device.map  fonts  grub.cfg  grubenv  i386-pc  locale
[root@ittutu grub2]# grub2-editenv list
saved_entry=CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)
[root@ittutu grub2]# uname -r
3.10.0-693.2.2.el7.x86_64
```
3. 如果是CentOS6，引导程序是init
```
[root@ittutu ~]# which init
/usr/sbin/init
```
4. 验证CentOS7一号进程是systemd
```
[root@ittutu ~]# top -p 1
top - 16:48:17 up 475 days,  5:10,  1 user,  load average: 0.13, 0.09, 0.12
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.7 us,  1.7 sy,  0.0 ni, 95.2 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  1883492 total,    78920 free,   870836 used,   933736 buff/cache
KiB Swap:        0 total,        0 free,        0 used.   828348 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND                                                                      
    1 root      20   0   51692   3328   2048 S  0.3  0.2  20:45.87 systemd                                                                      
```
5. file命令查看文件类型
```
[root@ittutu ~]# file dev
dev: directory
[root@ittutu ~]# file test.sh
test.sh: POSIX shell script, ASCII text executable
```