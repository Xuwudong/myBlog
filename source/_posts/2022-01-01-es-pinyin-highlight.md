---
title: ES使用拼音搜索并且highlight搜索结果
tags:
  - ES
date: 2022-01-01 23:08:57
---

### 实现
基于[此pinyin分词器](https://github.com/medcl/elasticsearch-analysis-pinyin) 实现搜索,自定义analyzer,使用type ngarm当做tokenizer,使用type pinyin当做filter,并且设置`"term_vector": "with_positions_offsets"`
### 🌰
#### 初始化
```bash
PUT es_pinyin_highlight
{
    "mappings" : {
      "dynamic" : "false",
      "properties" : {
        "text": {
          "type": "text",
          "fields": {
            "pinyin":{
              "type": "text",
              "analyzer": "pinyin_analyzer",
              "search_analyzer": "standard",
              "term_vector": "with_positions_offsets"
            }
          }
        }
      }
    },
    "settings" : {
      "index" : {
        "analysis": {
            "analyzer": {
                "pinyin_analyzer": {
                    "tokenizer": "my_ngram",
                    "filter": [
                        "pinyin_filter"
                    ]
                }
            },
            "tokenizer": {
                "my_ngram": {
                    "type": "ngram",
                    "min_gram": 2,
                    "max_gram": 3,
                    "token_chars": [
                        "letter",
                        "digit",
                        "punctuation",
                        "symbol"
                    ]
                }
            },
            "filter": {
                "pinyin_filter": {
                    "type": "pinyin",
                    "keep_full_pinyin": false,
                    "keep_joined_full_pinyin": true,
                    "keep_none_chinese_in_joined_full_pinyin": true,
                    "none_chinese_pinyin_tokenize": false,
                    "remove_duplicated_term": true
                }
            }
        }
      }
  }
}
```
#### 查询
```bash
PUT /es_pinyin_highlight/_create/1
{"text":"ES是世界上最好的搜索工具"}

POST /es_pinyin_highlight/_search
{
  "query": {
    "match": {
      "text.pinyin": "sousuo"
    }
  },
   "highlight": {
        "fields" : {
            "text" : {},
            "text.pinyin" : {}
        }
    }
}
```
##### 搜索结果
![img.png](/images/es/es_pinyin_highlight_res.png)
### 缺点
1. 基于ngarm当做tokenizer，需要控制`min_gram`、`max_gram`参数的大小，防止token过多，造成索引膨胀，写入性能降低
***
如果你有更好的方案，欢迎comment

### 参考
https://github.com/medcl/elasticsearch-analysis-pinyin/issues/185