---
layout: post
title:  "ElasticSearch笔记二：5.x版本变化"
subtitle: "ElasticSearch笔记系列"
date:   2017-10-15 15:22:00 +0800
categories: elasticsearch
header-img: img/posts/elastic/elasticsearch.jpg
tags:
- elasticsearch
- elastic
---


> 写在前面：去年写的有关Elastic的一些知识是基于2.x版本的，目前最新的版本是5.6（2017-10），一些重要的API与用法已经发生改变。这篇文章在之前系列的基础上，重点从API角度讲讲变化的部分。


## 一、映射的变化

### string类型变为为text/keyword

变化最大的是ES的基本类型string。目前string类型已标为废弃的，取而代之的变成了 text/keyword。text表示全文分析的string（即之前默认的string），keyword为不经分析的string(即not_analyzed的string)。

目前默认的字符串映射为

```json
{
  "type": "text",
  "fields": {
    "keyword": {
      "type": "keyword",
      "ignore_above": 256
    }
  }
}
```
即表明默认的字符串类型为分词的，可进行全文搜索等；其子关键字字段是未分析的，可进行精确查找、聚合及排序等。

> 如有个字段名为title为字符串类型。自动映射后，`title`可用于全文搜索，而`title.keyword`字段可进行聚合、排序等操作。


## 二、document API 变化

为演示方便，这里往ES添加一些数据。

> POST /cars/sale/_bulk
```json
{ "index": {}}
{ "price" : 10000, "color" : "red", "make" : "honda", "sold" : "2014-10-28" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 30000, "color" : "green", "make" : "ford", "sold" : "2014-05-18" }
{ "index": {}}
{ "price" : 15000, "color" : "blue", "make" : "toyota", "sold" : "2014-07-02" }
{ "index": {}}
{ "price" : 12000, "color" : "green", "make" : "toyota", "sold" : "2014-08-19" }
{ "index": {}}
{ "price" : 20000, "color" : "red", "make" : "honda", "sold" : "2014-11-05" }
{ "index": {}}
{ "price" : 80000, "color" : "red", "make" : "bmw", "sold" : "2014-01-01" }
{ "index": {}}
{ "price" : 25000, "color" : "blue", "make" : "ford", "sold" : "2014-02-12" }
```
现在我们有了关于汽车销售的有关数据。

###   _update_by_query 

5.x版本ES添加了`_update_by_query`API，可以根据查询到的结果进行更新。

目前我们有个需求是，所有福特（ford）汽车决定降价1000元。这正好可以使用`_update_by_query`完成。

> POST /cars/sale/_update_by_query
```json
{
	"query":{
		"term":{
			"make":"ford"
		}	
	},
	"script":{
		"inline":"ctx._source.price=ctx._source.price-1000",
		"lang":"painless" ①
	}
}
```
① `painless`为ES最新默认的脚本语言，相关资料可参考[painless脚本语言](https://www.elastic.co/guide/en/elasticsearch/painless/5.6/index.html)。

### _delete_by_query

`_delete_by_query`与上面提到的`_update_by_query`类似。它是根据查询删除某些文档。继续上面的示例。

删除所有宝马（bmw）车系。

> POST /cars/sale/_delete_by_query

```json
{
	"query":{
		"term":{
			"make":"bmw"
		}	
	}
}
```

### reindex

`_reindex` 功能为将文档从一个索引复制到另一个索引。利用它可以实现数据索引级别的无痛迁移，其中重要的是，在迁移时我们可以改变目标索引的某些字段类型。即平滑地升级我们的索引类型。

在我们的数据中，`color`和`make`都是text类型，意味着可用于全文检索，可在实际应用中，我们总是需要精确匹配他们，没必要分词，而ES的字段类型一旦确定又无法修改。

之前的做法是重建一个索引，然后利用`_bulk` 把数据批量导入新索引中。现在利用`_reindex`,可以实现一步导入。

首先需要重建一个索引，设置为需要的类型。

> PUT /cars_new

```json
{
	 "mappings": {
            "sale": {
                "properties": {
                    "color": {
                        "type": "keyword"
                    },
                    "make": {
                        "type": "keyword"
                    },
                    "price": {
                        "type": "integer"
                    },
                    "sold": {
                        "type": "date"
                    }
                }
            }
	 }
}
```
新索引`cars_new`的`color`和`make`为keyword类型,`price`修改为了`integer`类型（之前为`long`）。

使用`_reindex`迁移索引。

> POST　/_reindex
```json
{
	"source":{
		"index":"cars"
	},
	"dest":{
		"index":"cars_new"
	}
}
```
查看新索引`cars_new`确实创建成功，查看cars_new映射。

> GET /cars_new/_mappings/
```json
{
    "cars_new": {
        "mappings": {
            "sale": {
                "properties": {
                    "color": {
                        "type": "keyword"
                    },
                    "make": {
                        "type": "keyword"
                    },
                    "price": {
                        "type": "integer"
                    },
                    "sold": {
                        "type": "date"
                    }
                }
            }
        }
    }
}
```

### 关于过滤 filtered

目前过滤的API已经不支持`filtered`的语法了。实现过滤使用`constant_score`或在`bool`子句下`filter`实现。这两者都不会计算文档得分，使查询更高效。

如只获取绿色的汽车

> POST /cars_new/sale/_search

```json
{
	"query":{
		"constant_score":{
			"filter":{
				"term":{
					"color":"green"
				}
			}
		}
	}
}
```
或

```json
{
	"query":{
		"bool":{
			"filter":{
				"term":{
					"color":"green"
				}
			}
		}
	}
}
```
一般`bool` 下的过滤往往结合其他查询进行，若只有一个过滤，使用`constant_score`即可，它会将每个文档的评分都置为1。

## 三、其他变化

1.  5.x中取消了`search_type = count `语法，使用 `size:0`的方式来代替。
2. 添加了`profile`API，可以获取具体在查询时过滤，使可以有目的性的优化。
3. 其他变化可参考[5.0ES重大变化](https://www.elastic.co/guide/en/elasticsearch/reference/current/breaking-changes-5.0.html)


## 参考文章

1. [Elasticsearch之elasticsearch5.x 新特性](http://www.cnblogs.com/zlslch/p/6619089.html)
2. [Elasticsearch 5.4 中文文档](http://cwiki.apachecn.org/pages/viewpage.action?pageId=4260364)


