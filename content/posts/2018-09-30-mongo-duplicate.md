---
title: "MongoDB 数据重复的处理方法"
date: 2018-09-30T20:47:53+08:00
lastmod: 2018-09-30T20:47:53+08:00
keywords: [mongodb]
tags: [mongodb]
---

## 背景

在使用mongodb的过程中, 发现某张表因为一个bug引入了重复数据. 这个表的数据高达210多万条, 使用程序遍历查找重复数据, 并删除这些重复数据, 会显得不太合理. 下面是我解决这个问题的方法.

## 引起问题的原因

1. 确实有程序bug的影响
2. 表相关字段没有使用`unique`索引

## 解决方式

#### 一. 错误的解决方式

相当多的文章是用下面的解决方式:

```
db.test.ensureIndex({id: 1,name: 1}, {unique: true, dropDups: true})
```
这种方式由于BUG, 在`Mongodb 3.0.0`被废弃了.

#### 二. 正确的解决方法

##### 1. 查看重复数据

```
var count = 0;
db.bookshelves.aggregate(
    [
        { 
            "$group": { 
                "_id": { "bookshelfable_id": "$bookshelfable_id", "child_id": "$child_id" }, 
                "uniqueIds": { "$addToSet": "$_id" },
                "count": { "$sum": 1 } 
            }
        }, 
        { "$match": { "count": { "$gt": 1 } } },
    ],
    { allowDiskUse: true }
).map(function(record, index){
    count++
});
print(count);
```

这样就能看出来那些是重复数据, 并且重复数据有多少条.

##### 2. 删除重复数据

```
db.bookshelves.aggregate([
    { 
        "$group": {
        "_id": { "bookshelfable_id": "$bookshelfalble_id", "child_id": "$child_id" },
        "dups": { "$push": "$_id" },
        "count": { "$sum": 1 }
        }
    },
    { 
        "$match": { 
            "count": { "$gt": 1 } 
        }
    }
]).forEach(function(doc) {
    doc.dups.shift();
    db.events.remove({ "_id": {"$in": doc.dups }});
});
```

##### 3. 设置`unique` index

```
db.bookshelves.createIndex({"bookshelfable_id":1 , "child_id": 1},{unique:true})
```

设置完后, 插入重复数据就会抛出异常错误, 需要我们写的程序能够捕获这些异常.

```
E11000 duplicate key error collection: story.bookshelves index: bookshelfable_id_1_child_id_1 dup key: { : \"5acd7718dd43e0000b2ef5f9\", : 28.0 }
```

## 参考

1. https://blog.csdn.net/vacant2088/article/details/41491347
2. https://stackoverflow.com/questions/29072209/mongodb-duplicate-documents-even-after-adding-unique-key
3. https://stackoverflow.com/questions/13190370/how-to-remove-duplicates-based-on-a-key-in-mongodb