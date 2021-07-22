---
title: ES嵌套查询与退出嵌套
description: ES嵌套查询与退出嵌套
categories:
 - 后端
tags:
 - es
---

挺久没更新博客了，随手更新一篇吧，最新在使用ES时遇到的一个小问题，记录一下。

业务上有种场景需要对数据进行多条件的查询，数据结构大体如下：

```json
{
    "batch": 1111,
    "status": 100,
    "stages": [
        {
            "id": 123,
            "waybill": {
                "companyId": 123,
                "companyProduct": 321            
            }        
        }    
    ]
}
```

在es中索引信息大体如下：

```json
{
    "mappings": {
        "properties": {
            "batch": {
                "type": "long"            
            },
            "stages": {
                "type": "nested",
                "properties": {
                    "id": {
                        "type": "long"                    
                    },
                    "waybill": {
                        "properties": {
                            "companyId": {
                                "type": "integer"                            
                            },
                            "companyProduct": {
                                "type": "integer"                            
                            }                       
                        }                    
                    }                
                }            
            }         
        }    
    }
}
```

现在需要先按照 batch 字段做聚合，再根据stages下waybill的companyId和companyProduct做聚合，最后再根据status做聚合。

前面条件很容易能想到再aggs里这样写：

```json
{
    "aggs": {
        "batch": {
            "terms": {
                "field": "batch"            
            },
            "aggs": {
                "stages": {
                    "nested": {
                        "path": "stages"                    
                    },
                    "aggs": {
                        "companyId": {
                            "terms": {
                                "field": "stages.waybill.companyId"                            
                            },
                            "aggs": {
                                "companyProduct": {
                                    "terms": {
                                        "field": "stages.waybill.companyProduct"                                    
                                    }                                
                                }                            
                            }                        
                        }                    
                    }                
                }            
            }        
        }            
    }
}
```

而最后一个条件status字段又是在这个对象结构的根属性，如果继续在上面的json中companyProduct下添加子aggs，那么将查不到结果。

原因是在使用nested进行嵌套聚合时，后续的查询都在这个桶中，如果需要再对不属于当前嵌套的字段进行聚合时，需要先跳出这个桶，即退出嵌套：reverse_nested。

最终可以这样实现：

```json
{
    "aggs": {
        "batch": {
            "terms": {
                "field": "batch"            
            },
            "aggs": {
                "stages": {
                    "nested": {
                        "path": "stages"                    
                    },
                    "aggs": {
                        "companyId": {
                            "terms": {
                                "field": "stages.waybill.companyId"                            
                            },
                            "aggs": {
                                "companyProduct": {
                                    "terms": {
                                        "field": "stages.waybill.companyProduct"                                    
                                    },
                                    "aggs": {
                                        "rev": {
                                            "reverse_nested": {}, // 退出嵌套
                                            "aggs": {
                                                "status": {
                                                    "terms": {
                                                        "field": "status"                                                    
                                                    }                                                
                                                }                                            
                                            }                                        
                                        }                                    
                                    }                                
                                }                            
                            }                        
                        }                    
                    }                
                }            
            }        
        }            
    }
}
```


