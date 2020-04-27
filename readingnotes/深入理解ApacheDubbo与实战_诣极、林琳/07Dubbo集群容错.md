1. Dubbo重试配置
 - provider端全局配置
```
<dubbo:provider delay="-1" timeout="6000"  retries="0"/>
```
 - consumer端全局配置
```
<dubbo:consumer retries="0" timeout="6000"/>
```