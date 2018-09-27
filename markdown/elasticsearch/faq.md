# 问题及解决方案
---

## 一 maxClauseCount is set to 1024
    org.apache.lucene.search.BooleanQuery$TooManyClauses: maxClauseCount is set to 1024
            at org.apache.lucene.search.BooleanQuery.add(BooleanQuery.java:165)
            at org.apache.lucene.search.BooleanQuery.add(BooleanQuery.java:156)
            at org.apache.lucene.search.PrefixQuery.rewrite(PrefixQuery.java:54)
            at org.apache.lucene.search.BooleanQuery.rewrite(BooleanQuery.java:370)
            at org.apache.lucene.search.IndexSearcher.rewrite(IndexSearcher.java:151)
            at org.apache.lucene.search.Query.weight(Query.java:94)
一次业务查询中 es的in查询list大小超过5000，出现如上异常。解决方式是以业务逻辑限定list大小或者设置es集群的elasticsearch.yml添加如下配置
    
    ES5.0 之前版本
    index.query.bool.max_clause_count: 10240 
    ES5.0 及之后版本
    indices.query.bool.max_clause_count: 10240
    
## 二 string 没指定分词器导致查询失败
string在没有指定分词器情况下会使用默认分词器（standard Tokenizer），standard Tokenizer是于语法的分词器，适合欧洲语言的分词器，实现unicode算法，会将大写字母全部转换成小写，并存入倒排索引以供搜索。
    
    {
        "mappings": {
            "mumu": {
                "properties": {
                    "name": {
                        "type": "string"
                    }
                }
            }
        }
    }    

如果查询条件中包含中文使用ik-analyzer Tokenizer <br>
如果全是英文字符使用Whitespace Tokenizer(空格分词器) 或 在使用term确切查询把查询条件转换成小写字符串<br>

## 三 date数据差8小时
使用Java处理时间时，我们可能会经常发现时间不对，比如相差8个小时等等，其真实原因便是TimeZone。只有正确合理的运用TimeZone，才能保证系统时间无论何时都是准确的的。
    
    "date": {
        "format": "strict_date_optional_time||epoch_millis",
        "type": "date"
    } 
mapping 中设置时间format 使用带有时区的strict_date_optional_time 存储到es中的结果便是 2018-08-24T19:20:00.000+0800

## 四  Result window is too large
es 由于深翻页对性能有影响，可能造成集群不可用。所以查询结果集太大 会抛上诉异常

    Result window is too large, from + size must be less than or equal to: [10000] but was [10100]. 
    See the scroll api for a more efficient way to request large data sets. 
    This limit can be set by changing the [index.max_result_window] index level parameter.
可以通过查询添加参数 max_result_window

    curl -XPUT http://es-ip:9200/_settings -d '{ "index" : { "max_result_window" : 100000000}}
或修改 config/elasticsearch.yml 

    max_result_window: 100000
但是出现深翻页问题 最好以业务方式处理不要调大Result window size


