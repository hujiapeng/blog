1. 脚本就是命令的组合
2. 一行中组合多个命令执行可以用英文分号分隔命令，顺序执行
```
[root@ittutu ~]# cd /var;ls
adm  cache  crash  db  empty  games  gopher  kerberos  lib  local  lock  log  mail  nis  opt  preserve  run  spool  tmp  yp
```
3. 做成脚本文件，如果在一行多个命令，用英文分号分隔；也可以每个命令写一行，脚本文件起始行要写上使用哪种Shell编译器(注意：如果是可执行文件，给用户可执行权限即可；如果是脚本文件，要有可读可执行权限)
```
[root@ittutu ~]# vi testShell.sh
[root@ittutu ~]# cat testShell.sh
#!/bin/bash
# 注释 
cd /var
ls
pwd
du -sh
du -sh *
[root@ittutu ~]# chmod u+x testShell.sh 
[root@ittutu ~]# ls -l testShell.sh 
-rwxr--r-- 1 root root 31 Sep  3 18:08 testShell.sh
[root@ittutu ~]# bash testShell.sh 
```
如果运行脚本前指定了使用哪种Shell，如bash testShell.sh，则起始行的#表示注释