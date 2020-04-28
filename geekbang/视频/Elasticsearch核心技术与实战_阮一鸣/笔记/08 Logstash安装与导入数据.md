1. 下载并解压Logstash
```
[hjp@es1 ~]$ wget https://artifacts.elastic.co/downloads/logstash/logstash-7.6.0.tar.gz
[hjp@es1 ~]$ tar -zxvf logstash-7.6.0.tar.gz
```
2. 下载测试数据movies.csv
```
[hjp@es1 ~]$ wget https://github.com/hujiapeng/geektime-ELK/blob/master/part-1/2.4-Logstash%E5%AE%89%E8%A3%85%E4%B8%8E%E5%AF%BC%E5%85%A5%E6%95%B0%E6%8D%AE/movielens/ml-latest-small/movies.csv
```
3. 配置logstash配置文件logstash.conf，注意将file下path指向测试数据
```
input {
  file {
    path => "/home/hjp/movies.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}
filter {
  csv {
    separator => ","
    columns => ["id","content","genre"]
  }
  mutate {
    split => { "genre" => "|" }
    remove_field => ["path", "host","@timestamp","message"]
  }
  mutate {
    split => ["content", "("]
    add_field => { "title" => "%{[content][0]}"}
    add_field => { "year" => "%{[content][1]}"}
  }
  mutate {
    convert => {
      "year" => "integer"
    }
    strip => ["title"]
    remove_field => ["path", "host","@timestamp","message","content"]
  }
}
output {
   elasticsearch {
     hosts => "http://localhost:9200"
     index => "movies"
     document_id => "%{id}"
   }
  stdout {}
}
```
4. 启动logstash，注意要用-f指定logstash的配置文件；本机需要Java环境，可以使用```yum install java```进行安装
```
[hjp@es1 ~]$ logstash-7.6.0/bin/logstash -f logstash.conf
```