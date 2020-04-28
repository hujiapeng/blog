1. 下载docker-compose
```
[root@ittutu ~]# curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
2. 添加对docker-compose的可执行权限
```
[root@ittutu ~]# chmod +x /usr/local/bin/docker-compose
```
3. 将docker-compose添加为直接可执行命令
```
[root@ittutu ~]# ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
4. 查看docker-compose版本
```
[root@ittutu ~]# docker-compose --version 
```
5. 创建docker-compose.yml文件，并配置zookeeper镜像，搭建zookeeper集群环境。注意zookeeper版本，测试中，如果用最新的集群搭建失败。
```
version: '3'
services: 
        zoo1: 
            image: zookeeper:3.4
            restart: always
            container_name: zoo1_a
            ports: 
               - "2181:2181"
            environment: 
                ZOO_MY_ID: 1
                ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

        zoo2: 
            image: zookeeper:3.4
            restart: always
            container_name: zoo2_b
            ports: 
                - "2182:2181"
            environment: 
                ZOO_MY_ID: 2
                ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

        zoo3: 
            image: zookeeper:3.4
            restart: always
            container_name: zoo3_c
            ports: 
                - "2183:2181"
            environment: 
                ZOO_MY_ID: 3
                ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
```
6. 启动docker-compose，集群即可搭建完成。
```
[root@ittutu ~]# docker-compose up -d
```
7. 如果停掉docker-compose，执行如下命令，注意docker-compose启动的容器也会被删除
```
[root@ittutu ~]# docker-compose down
```
8. 查看docker-compose下运行的容器
```
[root@ittutu ~]# docker-compose ps
 Name               Command               State                     Ports                   
 --------------------------------------------------------------------------------------------
zoo1_a   /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2181->2181/tcp, 2888/tcp, 3888/tcp
zoo2_b   /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2182->2181/tcp, 2888/tcp, 3888/tcp
zoo3_c   /docker-entrypoint.sh zkSe ...   Up      0.0.0.0:2183->2181/tcp, 2888/tcp, 3888/tcp
```
9. docker inspect 容器名称或容器ID，可以查看容器详细信息。如果想查看某个具体数据，可以使用类似如下命令查看容器IP
```
[root@ittutu ~]# docker inspect -f '{{.NetworkSettings.Networks.root_default.IPAddress}}' zoo1_a
172.19.0.2
```
10. 进入其中一个容器，使用zkCli.sh创建一个节点，然后进入另一个容器查看
```
[root@ittutu ~]# docker exec -it zoo1_a bash
root@6faa0026e4dd:/zookeeper-3.4.14# zkCli.sh
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper]
[zk: localhost:2181(CONNECTED) 2] create /zk_test zk_data
Created /zk_test
[zk: localhost:2181(CONNECTED) 3] ls /
[zookeeper, zk_test]
```
```
[root@ittutu ~]# docker exec -it zoo2_b bash
root@34a901a91006:/zookeeper-3.4.14# zkCli.sh 
[zk: localhost:2181(CONNECTED) 0] ls /
[zookeeper, zk_test]
```