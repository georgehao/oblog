---
title: mongodb联表查询
author: haohongfan
type: post
date: 2017-06-17T14:36:31+00:00
categories:
  - mongodb
tags:
  - aggregation
  - mongodb
  - php

---
#### picture_books表
![picture_book](http://images.haohongfan.com/bookshelf.png)

#### bookshelfs表
![bookshelf](http://images.haohongfan.com/bookshelf1.png)

现在下面的需求:
1. bookshelf表的绘本列表展示(分页)
2. picture_book的**status**为1的需要为一组, 且按照时间排序
3. picture_book的**status**不为1需要分为一组, 且按照时间排序

**Note:** 
数据库: mongodb
关系: bookshelf通过**bookshelfabld_id**和picture_book的**id**关联

### 先分析一下这里面的难点:
#### mongodb的表之间无关系的, 无法关联查询
参照这两个表来说, 要分组的话, 需要知道bookshelfable_id对应的picture_book的status, 然后根据这个值进行分组. 但是因为是NOSQL,无法关联查询. 一般能想到的就是遍历**bookshelf**, 通过bookshelfable_id查询picture_book. 
这样就会造成: 本来在mysql中**一次**能查询搞定的事情, 在mongodb中, 需要查询**N次**才能完成. 这个效率不可谓不低. 

#### 如何解决这个问题?
其实说难也难, 说简单也简单. 这就需要我们学会去查看[官方文档](https://docs.mongodb.com/v3.0/). 

能解决这个问题的优雅方法只有: [Aggregation](https://docs.mongodb.com/v3.0/core/aggregation-introduction/)了

### 解决问题
解决这个问题的过程中, 我遇到下面几个难题, 都已经基本解决.

##### 1. 如何联表查询
其实这个挺简单的, 只要使`$match`,`$lookup`就可以了.
```
db.bookshelves.aggregate([ 
	{ 
		$match: {
			"child_id":223 
		}
	}, 
	{
		$lookup: {
			from: "picture_books",
			localField: "bookshelfable_id",
			foreignField: "_id",
			as: "picture_books"
		}
	}
])
```
**是不是很简单?**, 如果要是这么想你就错了.
```
localField: "bookshelfable_id",
foreignField: "_id",
```
`bookshelfable_id`是string类型, 而`_id`是ObjectId类型. 所以直接这样比较是没有办法直接进行匹配. `所以这样写是错误的`
其实这个问题我没有解决, 从stackoverflow看的用```ObjectId()```是无法起作用的. 有解决的同学请给我留言.

**想了一个折中的办法:** 
就是在```picture_books```表后再增加一个字段```book_id```,也就是```_id```的```String```格式.

picture_books表就变成下面的样子:
![11](http://images.haohongfan.com/picture_book_book_id-1.png)
于是, 查询的语句也变成下面的样子:
```
db.bookshelves.aggregate([ 
	{ 
		$match: {
			"child_id":223 
		}
	}, 
	{
		$lookup: {
			from: "picture_books",
			localField: "bookshelfable_id",
			foreignField: "book_id",
			as: "picture_books"
		}
	}
])
```
其返回数据:
![22](http://images.haohongfan.com/书架返回1.png)

##### 2. 如何过滤想要的字段
过滤想要的字段只有使用```$project```就可以了. 但是这里有个坑(因为当时没有注意到, 导致调试了很久)
```
db.bookshelves.aggregate([ 
	{ 
		$match: {
			"child_id":223 
		}
	}, 
	{
		$lookup: {
			from: "picture_books",
			localField: "bookshelfable_id",
			foreignField: "book_id",
			as: "picture_books"
		}
	}, 
	{
		$project: {
			picture_books: {
				name: 1,
				status: 1
			}, 
			cmp: {
				$eq: ["$picture_books.status", 1]
			}
		}
	}
])
```
过滤主要是利用[$project](https://docs.mongodb.com/manual/reference/operator/aggregation/project/). 使用`$project`很简单. 要显示的字段`置1`即可. 看我上面贴出的代码有什么问题 ? ?. 

上面代码的执行结果是: 
```
{ "_id" : ObjectId("594107284bcd1c0ac87bde82"), "picture_books" : [ { "name" : "14只老鼠吃早餐", "status" : 1 } ], "cmp" : false }
{ "_id" : ObjectId("5941073b4bcd1c0ad666a3c2"), "picture_books" : [ { "name" : "14只老鼠过冬天", "status" : 1 } ], "cmp" : false }
.....
```
不管怎么执行, `cmp`始终是`false`. 要是这么写, 要是这么写基本上就入坑了. 

![](http://images.haohongfan.com/书架返回1.png)

注意看图片上标红色`下划线`的地方, 返回的`picture_books`是个数组. 这就是这个问题造成的关键点. 对`$eq: ["$picture_books.status", 1]`操作的时候, 只能是对具体对象进行比较, 不能对整个数组操作. 

找到问题了, 贴出正确的代码:
```
db.bookshelves.aggregate([
	{
		$match: {"child_id":257}
	},
	{
		$lookup: {
			from: "picture_books",
			localField: "bookshelfable_id", 
			foreignField: "book_id", 
			as: "picture_books"
		}
	},
	{
		$unwind: "$picture_books"
	},
	{
		$project: {
			picture_books: {
				name: 1,
				status: 1
			}, 
			cmp: {
				$eq: ["$picture_books.status", 1]
			}
		}
	}])
```
上面代码的返回值:
```
{ "_id" : ObjectId("5949e6474bcd1c4bc71d5083"), "picture_books" : { "name" : "托马斯不要泪汪汪", "status" : 1 }, "cmp" : true }
{ "_id" : ObjectId("5949e6594bcd1c4c2201c063"), "picture_books" : { "name" : "托马斯不怕黑洞洞", "status" : 1 }, "cmp" : true }
Type "it" for more
{ "_id" : ObjectId("5949e6914bcd1c4c4d6c7eb3"), "picture_books" : { "name" : "拇指姑娘", "status" : 0 }, "cmp" : false }
```
##### 3. 如何分组
看到这个问题, 理论上应该使用`$group`. 但是[$group](https://docs.mongodb.com/v3.0/reference/operator/aggregation/group/#pipe._S_group)其实不合适.  这里我们直接使用`$sort`代替分组. 

> 现在下面的需求:
> 1. bookshelf表的绘本列表展示(分页)
> 2. picture_book的**status**为1的需要为一组, 且按照时间排序
> 3. picture_book的**status**不为1需要分为一组, 且按照时间排序

根据这个需求, 这里的思路是:
1. 先按照`cmp`倒序排序, 把`status=1`和 `status != 1`的分为两组
2. 然后再分别利用`order_column`排序

具体`$sort`如何使用, 请查看[$sort](https://docs.mongodb.com/manual/reference/operator/aggregation/sort/)

##### 4. 如何排序
上面已经解释了, 直接贴代码
```
db.bookshelves.aggregate([
	{
		$match: {"child_id":257}
	},
	{
		$lookup: {
			from: "picture_books",
			localField: "bookshelfable_id", 
			foreignField: "book_id", 
			as: "picture_books"
		}
	},
	{
		$unwind: "$picture_books"
	},
	{
		$project: {
			order_column:1,
			picture_books: {
				name: 1,
				status: 1
			}, 
			cmp: {
				$eq: ["$picture_books.status", 1]
			}
		}
	},
	{
		$sort: {
			cmp: -1,
			order_column: -1 
		}
	}])
```
结果:
```
{ "_id" : ObjectId("594206184bcd1c0dae330b59"), "order_column" : 50, "picture_books" : { "alg_book_id" : 23, "name" : "杰琪的自行车旅行", "status" : 1 }, "cmp" : true }
{ "_id" : ObjectId("594210da4bcd1c134e4a6823"), "order_column" : 52, "picture_books" : { "alg_book_id" : 169, "name" : "我去找回太阳", "status" : 1 }, "cmp" : true }
{ "_id" : ObjectId("594210e44bcd1c134e4a6825"), "order_column" : 53, "picture_books" : { "alg_book_id" : 167, "name" : "我想有颗星星", "status" : 1 }, "cmp" : true }
```

__NOTE:重点看order_column的值__

这里其实还有一个坑, 如果上面的代码变成这样, 结果会怎么样呢?
```
db.bookshelves.aggregate([
	{
		$match: {"child_id":257}
	},
	{
		$lookup: {
			from: "picture_books",
			localField: "bookshelfable_id", 
			foreignField: "book_id", 
			as: "picture_books"
		}
	},
	{
		$unwind: "$picture_books"
	},
	{
		$project: {
			picture_books: {
				name: 1,
				status: 1
			}, 
			cmp: {
				$eq: ["$picture_books.status", 1]
			}
		}
	},
	{
		$sort: {
			cmp: -1,
			order_column: -1 
		}
	}])
```

 和上面正确的代码相比, 这段代码缺少第18行: `order_column:1,`. 其实这样是无法按照`order_column`进行排序. 所以要使用`$sort`的话, 一定需要在`project`中设置出来. 

##### 5. 如何分页
关于如何分页就使用`$limit`, `$skip`, 就可以了, 很简单. 

但是, 需要注意的是:
>When a $sort immediately precedes a $limit in the pipeline, the $sort operation only maintains the top n results as it progresses, where n is the specified limit, and MongoDB only needs to store n items in memory. This optimization still applies when allowDiskUse is true and the n items exceed the aggregation memory limit.


想了很久, 结合实际的操作, 这句话的意思是: sort排序是对整体排序, 但是排序结果, mongodb在内存中只会保持`$limit`指定的n个数.

### 局限性
OK, 经过上面的啰嗦, 下面要说点最重要的事情.

mongodb的官方文档说
> $lookup: 
> Performs a left outer join to an unsharded collection in the same database to filter in documents from the “joined” collection for processing. The $lookup stage does an equality match between a field from the input documents with a field from the documents of the “joined” collection.

简单点说就是: `$lookup`只能在同一个数据库中, 且这个`collection`不能有分片. 如果你的表设计不在一个库中, 且设置了分片的话, 那上面的全都是废话. 
