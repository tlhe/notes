__1.创建修改索引__
```
curl -XPUT 'localhost:9200/blogs/_settings?pretty' -H 'Content-Type: application/json' -d'
'
 {
    "number_of_replicas" : 2
 }
```
__2.插入数据__
```
curl -XPUT 'localhost:9200/website/blog/123?pretty' -H 'Content-Type: application/json' -d'
 {
   "title": "My first blog entry",
   "text":  "Just trying this out...",
   "date":  "2014/01/01"
 }
 '
```

__3.根据id查数据__
```
curl -i -XGET http://localhost:9200/website/blog/123?pretty
```

__3-1 查询部分字段__
```
curl -XGET 'localhost:9200/website/blog/123?_source=title,text&pretty'
```

__3-2 只获取source，不要元字段__
```
curl -XGET 'localhost:9200/website/blog/123/_source?pretty'
```

__4.判断文档是否存在__
__4-1 如果只想检查一个文档是否存在 --根本不想关心内容--那么用 HEAD 方法来代替 GET 方法__
```
curl -i -XHEAD http://localhost:9200/website/blog/123
```

__5.更新文档（根据id）【①从旧文档构建 JSON②更改该 JSON③删除旧文档④索引一个新文档】__
```
curl -XPUT 'localhost:9200/website/blog/123?pretty' -H 'Content-Type: application/json' -d'
{
  "title": "My first blog entry",
  "text":  "I am starting to get the hang of this...",
  "date":  "2014/01/02"
}
'
```

__5-1.局部更新文档__
```
curl -XPOST 'localhost:9200/website/blog/1/_update?pretty' -H 'Content-Type: application/json' -d'
{
   "script" : "ctx._source.views+=1"
}
'
```

```
curl -XPOST 'localhost:9200/website/blog/1/_update?pretty' -H 'Content-Type: application/json' -d'
{
    "script" : {
        "source": "ctx._source.tags.add(params.tag)",
        "lang": "painless",
        "params" : {
            "tag" : "blue"
        }
    }
}
'
```

__6.删除文档__
```
curl -XDELETE 'localhost:9200/website/blog/123?pretty'
```

__7.获取多个文档（顺序与请求中的顺序相同）__
```
curl -XGET 'localhost:9200/_mget?pretty' -H 'Content-Type: application/json' -d'
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    1
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         "_source": "views"
      }
   ]
}
'
```

__8.父子文档__
```
{
  "order": 1,
  "template": "company",
  "settings": {
    "index": {
      "number_of_shards": "3"
    }
  },
  "mappings": {
    "employee": {
      "_parent": {
        "type": "branch"
      }
    },
    "branch": {}
  },
  "aliases": {}
}
```

```
curl -XPOST 'localhost:9200/company/branch/_bulk?pretty' -H 'Content-Type: application/json' -d'
{ "index": { "_id": "london" }}
{ "name": "London Westminster", "city": "London", "country": "UK" }
{ "index": { "_id": "liverpool" }}
{ "name": "Liverpool Central", "city": "Liverpool", "country": "UK" }
{ "index": { "_id": "paris" }}
{ "name": "Champs élysées", "city": "Paris", "country": "France" }'
```

```
curl -XPUT 'localhost:9200/company/employee/1?parent=london&pretty' -H 'Content-Type: application/json' -d'
{
  "name":  "Alice Smith",
  "dob":   "1970-10-24",
  "hobby": "hiking"
}'
```

```
{
  "query": {
    "has_child": {
      "type": "employee",
      "query": {
        "match": {
          "hobby": "hiking"
        }
      }
    }
  }
}
```