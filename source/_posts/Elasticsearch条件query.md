---
title: Elasticsearch条件query
date: 2019-06-25 17:10:30
tags: [Elasticsearch]
categories:
- 数据库
- Elasticsearch
---

## 按照时间升序降序query

```python
def query_byid(self, id, size, reverse_order=False):
        if reverse_order == True:
            _order = 'desc'
        else:
            _order = 'asc'
        dsl = {
            "query": {
                "bool":{
                    "must": [
                        {"match":{"id":id}},
                    ],
                }
            },
            "sort": [
                { "service.data.time" : {"order" : _order}},
                "_score"
            ],
            "_source": [
                "service.data",
                "deviceId"
            ],
            "size":size,
        }
        raw_data = self.query(dsl)
        raw_list = raw_data["hits"]["hits"]
        return raw_list
```
