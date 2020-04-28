1. 文档和索引偏向于开发人员；节点和分片偏向于运维人员
2. 文档
 - Elasticsearch是面向文档的，文档是所有可搜索数据的最小单位
 - 文档会被序列化成JSON格式，保存在Elasticsearch中。JSON对象由字段组成，每个字段都有对应的字段类型(字符串/数值/布尔/日期/二进制/范围类型)
 - 每个文档都有一个UniqueID。可自己指定也可通过Elasticsearch自动生成
 - 文档元数据
    - _index：文档所属的索引名
    - _type：文档所属的类型名
    - _id：文档唯一ID
    - _source：文档的原始JSON数据
    - _version：文档的版本号
    - _score：相关性分值
3. 索引
 - Index：索引是文档的容器，是一类文档的集合。Index体现的是逻辑空间的概念，每个索引都有自己的Mapping定义，用于定义包含的文档的字段名和字段类型；Shard体现了物理空间的概念，索引中的数据分散在Shard上
 - 索引的Mapping与Settings。Mapping定义文档字段的类型；Settings定义不同的数据分布，如需要多少分片
4. Type：7.0之前可以设置多个Type；6.0开始Type被Deprecated,7.0开始，一个索引只能创建一个Type，那就是"_doc"
5. ES中的Index相当于关系型数据库中的Table，ES中Document相当于关系型数据库中的Row，ES中的Field相当于关系型数据库中的Column，ES中的Mapping相当于关系型数据库中的Schema，ES中的DSL相当于关系型数据库中的SQL。
6. ES对外服务使用REST API，可以很容易的被各种语言调用 
7. 在Kibana系统中进入管理(Management)菜单下，点击索引管理(Index Management)，可以看到通过logstash导入的movies测试数据。点击Name列的movies，可以看到Settings和Mapping
8. 在Kibana开发工具(dev tools)中调用ES的REST API。注意调用中如果路径开始没有反斜杆(/)，Kibana会自动加上，如```GET _cat/plugins```相当于```GET /_cat/plugins```
```
//查看movies索引相关信息
GET movies
//查看movies索引的文档总数
GET /movies/_count
//根据索引名称进行通配符查询(indices为index复数形式之一)
GET /_cat/indices/movie*?v&s=index
//按照文档个数排序
GET /_cat/indices?v&s=docs.count:desc
```