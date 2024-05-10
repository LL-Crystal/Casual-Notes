# Elasticsearch常用命令

### 基本命令
关闭索引
``` 
curl -XPOST 'http://localhost:9200/_all/_close'
```
打开索引
```
curl -XPOST 'http://localhost:9200/_all/_open'
```
查看index与mapping
```
curl -XGET 'http://localhost:9200/_cat/indices?v'
curl -XGET 'http://localhost:9200/index_x/_mapping?pretty='
```
查看分片情况
```
curl -XGET 'http://localhost:9200/_cat/shards'
```
索引删除
```
curl -XDELETE 'http://localhost:9200/_all'
curl -XDELETE 'http://localhost:9200/index_x/?pretty'
```
创建别名
```
curl -XPOST 'http://localhost:9200/_aliases' -d '{"actions":[{"add":{"index":"index_x","alias":"message"}}]}'
```

### 快照与恢复
创建仓库
```
curl -XPUT 'http://localhost:9200/_snapshot/snapshot_backup/?pretty' -H 'Content-Type: application/json' -d '{"type":"hdfs","settings":{"uri":"hdfs://ip:9000/","path":"/elasticsearch/snapshot_x","user":"username"}}'
```
创建快照
```
curl -XPUT 'http://localhost:9200/_snapshot/snapshot_backup/snapshot_x'
```
恢复快照
```
curl -XPOST 'http://localhost:9200/_snapshot/snapshot_backup/snapshot_x/_restore?pretty'
```
查看快照恢复情况
```
curl -XGET 'http://localhost:9200/index_x/_recovery/'
```
删除快照
```
curl -XDELETE 'http://localhost:9200/_snapshot/snapshot_backup/snapshot_x?pretty' -H 'Content-Type: application/json' -d '{ "indices": "index_x", "ignore_unavailable": "true", "include_global_state": "false", "partial": "false" }'
```
