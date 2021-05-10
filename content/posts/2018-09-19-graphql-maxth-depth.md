---
title: "Graphql最大复杂度和最大深度设置"
date: 2018-09-19T16:44:46+08:00
lastmod: 2018-09-19T16:44:46+08:00
keywords: [graphql]
categories: [graphql]
---

# 设置最大深度和最大负责度的好处

因为把查询的权利交给了客户端, 客户端可以进行非常复杂的查询. 因为客户端可能进行恶意的查询或者进行非常大的查询, 因此服务端要拒绝这样的查询.

总共有三种方式可以进行:

1. 设置响应过期时间
2. 设置查询最大深度
3. 设置查询最大复杂度

设置响应过期时间需要对服务端代码和性能要求比较高, 可以先不管, 当服务器代码和性能上去再设置.

# 如何判断最大深度

#### 1. 简单的查询,depth = 0

```
query shallow1 {
  thing1
}
```

#### 2. 内联片段（Inline Fragments）不能增加depth, depth = 0

```
query shallow2 {
  thing1
  ... on Query {
    thing2
  }
}
```

#### 3. Named Fragments同样不能增加depth, depth = 0

```
query shallow3 {
  ...queryFragment
}

fragment queryFragment on Query {
  thing1
}
```

#### 4. depth = 1的查询

```
query deep1_1 {
  viewer {
    name
  }
}

query deep1_2 { // Inline Fragments
  viewer {
    ... on User {
      name
    }
  }
}
```

#### 5. depth = 2的查询

```
query deep2 {
  viewer {
    albums {
      title
    }
  }
}
```

#### 6. depth = 3的查询

```
query deep3 {
  viewer {
    albums {
      ...musicInfo
      songs{
        ...musicInfo
      }
    }
  }
}

fragment musicInfo on Music {
  id
  title
  artists
}
```

## IntrospectionQuery

这里需要注意, 客户端在使用服务端graphql的时候, 需要去问GraphQL Schema它支持哪些查询, GraphQL 通过内省系统让我们可以做到这点. 通过查询 __schema 字段来向 GraphQL 询问哪些类型是可用的

这个`IntrospectionQuery`的最大深度是11. 在使用[laravel-graphql](https://github.com/Folkloreatelier/laravel-graphql)的时候, 要注意这个问题. 

```
query IntrospectionQuery {
    __schema {
        queryType {
            name
        }
        mutationType {
            name
        }
        subscriptionType {
            name
        }
        types {
            ...FullType
        }
        directives {
            name
            description
            locations
            args {
            ...InputValue
            }
        }
    }
}

fragment FullType on __Type {
    kind
    name
    description
    fields(includeDeprecated: true) {
        name
        description
        args {
            ...InputValue
        }
        type {
            ...TypeRef
        }
        isDeprecated
        deprecationReason
    }
    inputFields {
        ...InputValue
    }
    interfaces {
        ...TypeRef
    }
    enumValues(includeDeprecated: true) {
        name
        description
        isDeprecated
        deprecationReason
    }
    possibleTypes {
        ...TypeRef
    }
}

fragment InputValue on __InputValue {
    name
    description
    type {
        ...TypeRef
    }
    defaultValue
}

fragment TypeRef on __Type {
    kind
    name
    ofType {
        kind
        name
        ofType {
            kind
            name
            ofType {
                kind
                name
                ofType {
                    kind
                    name
                    ofType {
                        kind
                        name
                        ofType {
                            kind
                            name
                            ofType {
                                kind
                                name
                            }
                        }
                    }
                }
            }
        }
    }
}
```

# 设置最大复杂度

```
query Test {
  droid(id: "1000") {
    id
    serialNumber
  }

  pets(limit: 20) {
    name
    age
  }
}
```

如何计算这个请求的最大复杂度: 每个字段都是1, 最大复杂度是每个字段的和, 参数不管有多少个, 都是1. 所以这个复杂度是6


# 参考资料

1. [内省](https://graphql.cn/learn/introspection/)
2. [Security](https://www.howtographql.com/advanced/4-security/)
3. [Query Complexity Analysis](https://sangria-graphql.org/learn/#query-complexity-analysis)