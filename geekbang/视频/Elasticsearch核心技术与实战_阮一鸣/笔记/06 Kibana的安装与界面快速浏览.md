1. 下载并解压Kibana
```
[hjp@es1 ~]$ wget https://artifacts.elastic.co/downloads/kibana/kibana-7.6.0-linux-x86_64.tar.gz
[hjp@es1 ~]$ tar -zxvf kibana-7.6.0-linux-x86_64.tar.gz
```
2. 修改kibana配置文件config下kibana.yml，设置elasticsearch.hosts指定ES实例，默认是```http://localhost:9200```，如果想外网访问还要加配置```server.host: "0.0.0.0"```。注意冒号后的空格。
```
elasticsearch.hosts: ["http://localhost:9200"]
server.host: "0.0.0.0"
```
3. 运行bin/kibana，启动有点慢
```
[hjp@es1 ~]$ kibana-7.6.0-linux-x86_64/bin/kibana
```
4. 访问``` http://192.168.2.155:5601  ```就是Kibana首页；Kibana还有DevTools(```http://192.168.2.155:5601/app/kibana#/dev_tools/console```)可以访问ES接口，如```GET /```返回访问ES首页数据；如访问```GET /_cat/plugins```返回ES插件信息。
5. 相关学习在官网教程```https://www.elastic.co/guide/cn/kibana/current/index.html```