1. Shell是命令解释器，用于解释用户对操作系统的操作；CentOS7默认使用的Shell是bash
2. 当我们写入脚本命令后，命令解释器会解析并执行操作系统内核，从而实现用户想要的操作结果
3. 查看系统中的Shell
```
[root@ittutu ~]# cat /etc/shells
/bin/sh
/bin/bash
/sbin/nologin
/usr/bin/sh
/usr/bin/bash
/usr/sbin/nologin
```