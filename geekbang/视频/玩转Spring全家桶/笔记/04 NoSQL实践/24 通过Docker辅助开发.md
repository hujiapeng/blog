1. 停止所有容器，-a表示所有,-q表示静默模式，只显示容器编号
 ```docker stop $(docker ps -a -q) ```
2. 删除所有容器
 ```docker rm $(docker ps -a -q) ```
3. 删除所有镜像
 ```docker rmi $(docker images -q) ```