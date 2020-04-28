1. Elasticsearch基于Java语言开发，之前需要JDK开发环境，从7.0开始内置了Java环境。
2. 下载并解压es
```
[root@localhost ~]# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.0-linux-x86_64.tar.gz
[root@localhost ~]# tar -zxvf elasticsearch-7.6.0-linux-x86_64.tar.gz 
```
3. es目录介绍：
 - bin：脚本文件，包括启动elasticsearch，安装插件。运行统计数据等；
 - config：配置文件elasticsearch.yml，集群配置文件，user、role based相关配置；
 - JDK：Java运行环境；
 - data：配置文件path.data，数据文件；
 - lib：Java类库；
 - logs：配置文件path.log，日志文件；
 - modules：包含所有ES模块；
 - plugins：包含所有已安装插件。

4. JVM配置
 - 修改JVM-config/jvm.options：7.1下载的默认设置是1GB
 - 配置建议：
    - Xmx和Xms设置成一样；
    - Xmx不要超过机器内存的50%
    - 不要超过30GB

5. 启动ES，执行bin下的elascticsearch文件。注意，如果使用root用户启动会报错```org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root```。
 - 修改```/etc/hostname```为es1;同时修改```/etc/hosts```
```
127.0.0.1   localhost es1
::1         localhost es1
```
 - 添加用户及设置密码
```
[root@es1 ~]# useradd hjp
[root@es1 ~]# passwd hjp
```
 - 切换到hjp用户，执行bin下的elascticsearch，如果想后台运行，命令后加上&符号。此时只能本地访问，如果想可以外网访问，继续下面操作步骤
```
[root@es1 ~]# su hjp
[hjp@es1 root]$ cd
[hjp@es1 ~]$ elasticsearch-7.6.0/bin/elasticsearch
```
 - 允许外网访问
    - 修改conf/elasticsearch.yml文件，添加```network.host: 0.0.0.0```
    - 启动ES后，如果报错如下，需要用root用户继续下面操作。错误日志可以在ES日志文件内看```/home/hjp/elasticsearch-7.6.0/logs/elasticsearch.log```
        ```
ERROR: [4] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
[2]: max number of threads [3884] for user [hjp] is too low, increase to at least [4096]
[3]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[4]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```
 - 处理错误```max file descriptors```，修改/etc/security/limits.conf文件，添加如下
    ```
    * soft nofile 65535
    * hard nofile 65535
```
 - 处理错误```max number of threads```，修改/etc/security/limits.conf文件，添加如下
    ```
    * soft nproc  4096
    * hard nproc  4096
```
 - 处理错误```max virtual memory areas vm.max_map_count```，修改/etc/sysctl.conf文件，添加如下
    ```
    vm.max_map_count=262144
```
 - 处理错误```the default discovery settings are unsuitable for production use```，修改文件ES配置文件config下elasticsearch.yml，添加如下
```
cluster.initial_master_nodes: ["node-1"]
```
 - reboot now重启服务器，切换到hjp用户下，启动ES，会有提示信息如下
 ```
······
[2020-02-21T13:57:22,309][INFO ][o.e.h.AbstractHttpServerTransport] [es1] publish_address {192.168.2.155:9200}, bound_addresses {[::]:9200}
[2020-02-21T13:57:22,311][INFO ][o.e.n.Node               ] [es1] started
······
```
 - 访问```http://192.168.2.155:9200/```可查看到相关信息，如下
 ```
    {
      "name" : "es1",
      "cluster_name" : "elasticsearch",
      "cluster_uuid" : "5VdL3nMBSAq2n4j2kKhn-g",
      "version" : {
        "number" : "7.6.0",
        "build_flavor" : "default",
        "build_type" : "tar",
        "build_hash" : "7f634e9f44834fbc12724506cc1da681b0c3b1e3",
        "build_date" : "2020-02-06T00:09:00.449973Z",
        "build_snapshot" : false,
        "lucene_version" : "8.4.0",
        "minimum_wire_compatibility_version" : "6.8.0",
        "minimum_index_compatibility_version" : "6.0.0-beta1"
      },
      "tagline" : "You Know, for Search"
    }
```
6. 为ES安装插件
 - 执行如下命令查看ES现有插件
```
[hjp@es1 ~]$ elasticsearch-7.6.0/bin/elasticsearch-plugin list
```
 - 安装国际化分词插件analysis-icu，然后再通过查看ES现有插件命令就可以看到安装的插件了。也可以将插件下载到本地，通过file协议安装，如```elasticsearch-plugin install file://插件路径```
```
[hjp@es1 ~]$ elasticsearch-7.6.0/bin/elasticsearch-plugin install analysis-icu
-> Installing analysis-icu
-> Downloading analysis-icu from elastic
[=================================================] 100%   
-> Installed analysis-icu
[hjp@es1 ~]$ elasticsearch-7.6.0/bin/elasticsearch-plugin list
analysis-icu
```
 - 重启ES后，通过访问```http://192.168.2.155:9200/_cat/plugins```也可查出已有插件
```
es1 analysis-icu 7.6.0
```
 - 本机启动多个ES实例，搭建本地集群。我服务器配置内存3GB，ES配置config下jvm.options内-Xms和-Xmx内存为1g；如果太小服务启动失败。下面启动两个实例node0和node1，要修改elasticsearch.yml来指定集群实例```cluster.initial_master_nodes: ["node0"]```；还有就是，由于多个实例在本机，不要在elasticsearch.yml文件中配置端口
 ```
[hjp@es1 ~]$ elasticsearch-7.6.0/bin/elasticsearch -E node.name=node0 -E cluster.name=geektime -E path.data=node0_data -d
[hjp@es1 ~]$ elasticsearch-7.6.0/bin/elasticsearch -E node.name=node1 -E cluster.name=geektime -E path.data=node1_data -d
```
 - 通过查看日志文件，查看各个实例端口
```
···
[2020-02-21T22:16:00,847][INFO ][o.e.h.AbstractHttpServerTransport] [node0] publish_address {192.168.2.155:9200},
···
[2020-02-21T22:17:26,360][INFO ][o.e.h.AbstractHttpServerTransport] [node1] publish_address {192.168.2.155:9201},
···
```
 - 访问```http://192.168.2.155:9200/_cat/nodes```查看各实例
 ```
192.168.2.155 16 97 2 0.11 0.12 0.15 dilm * node0
192.168.2.155  7 97 1 0.11 0.12 0.15 dilm - node1
```