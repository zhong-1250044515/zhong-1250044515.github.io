---
layout: post
title: "RESTful API 设计规范"
date: 2018-06-06
excerpt: "在项目接口开发过程中，要求团队遵循的 RESTful API 设计规范。"
tags: [技术文档]
comments: false
---
## API 地址

在 ``url`` 中指定 API 的版本是个很好地做法。如果 API 变化比较大，可以把 API 设计为子域名，比如``https://api.github.com``；也可以简单地把版本放在路径中，比如``https://example.com/api/``。



## 返回格式

JSON



## 以资源为中心的 URL 设计

在 RESTful 架构中，每个网址代表一种资源（resource），所以网址中不能有动词，只能有名词，而且所用的名词往往与数据库的表格名对应。一般来说，数据库中的表都是同种记录的”集合”（collection），所以API中的名词也应该使用复数。

举例来说，有一个API提供动物园（zoo）的信息，还包括各种动物和雇员的信息，则它的路径应该设计成下面这样。
```
https://api.example.com/v1/zoos
https://api.example.com/v1/animals
https://api.example.com/v1/employees
```
我们可以看到几个特性：
 - 资源分为单个文档和集合，尽量使用复数来表示资源，单个资源通过添加 id 或者 name 等来表示
 - 一个资源可以有多个不同的 URL
 - 资源可以嵌套，通过类似目录路径的方式来表示，以体现它们之间的关系
> 根据RFC3986定义，URL是大小写敏感的。所以为了避免歧义，尽量使用小写字母。



## HTTP动词

对于资源的具体操作类型，由 HTTP 动词表示，项目用到以下四种。

- GET（SELECT）：从服务器取出资源（一项或多项）。
- POST（CREATE）：在服务器新建一个资源。
- PUT（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源）。
- DELETE（DELETE）：从服务器删除资源。

比如
```
GET /zoos：列出所有动物园
POST /zoos：新建一个动物园
GET /zoos/ID：获取某个指定动物园的信息
PUT /zoos/ID：更新某个指定动物园的信息（提供该动物园的全部信息）
PATCH /zoos/ID：更新某个指定动物园的信息（提供该动物园的部分信息）
DELETE /zoos/ID：删除某个动物园
GET /zoos/ID/animals：列出某个指定动物园的所有动物
DELETE /zoos/ID/animals/ID：删除某个指定动物园的指定动物
```



## 不符合 CURD 的情况

在实际资源操作中，总会有一些不符合 CRUD（Create-Read-Update-Delete） 的情况，一般有几种处理方法。

**1. 使用 POST**

为需要的动作增加一个 endpoint，使用 POST 来执行动作，比如 ``POST /login`` 登录。

**2. 增加控制参数**

添加动作相关的参数，通过修改参数来控制动作。比如一个博客网站，会有把写好的文章“发布”的功能，可以用上面的 ``POST /articles/:article_id/publish`` 方法，也可以在文章中增加 ``published:boolean`` 字段，发布的时候就是更新该字段 ``PUT /articles/:article_id?published=true``

**3. 把动作转换成资源**

把动作转换成可以执行 ``CRUD`` 操作的资源。
比如“喜欢”一个 ``gist``，就增加一个 ``/gists/:id/star`` 子资源，然后对其进行操作：“喜欢”使用``PUT /gists/:id/star``，“取消喜欢”使用 ``DELETE /gists/:id/star``。
另外一个例子是 ``Fork``，这也是一个动作，但是在 gist 下面增加 ``forks``资源，就能把动作变成 ``CRUD`` 兼容的：``POST /gists/:id/forks`` 可以执行用户 fork 的动作。



## 过滤信息

- ?limit=10：指定返回记录的数量
- ?offset=10：指定返回记录的开始位置。
- ?page=2&per_page=100：指定第几页，以及每页的记录数。
- ?sortby=name&order=asc：指定返回结果按照哪个属性排序，以及排序顺序。
- ?animal_type_id=1：指定筛选条件

参数的设计允许存在冗余，即允许API路径和URL参数偶尔有重复。比如，``GET /zoo/ID/animals`` 与 ``GET /animals?zoo_id=ID`` 的含义是相同的。



## 状态码规范

使用 HTTP 状态规范，在 response 中自定义 code 状态码的方法不推荐。

| HTTP 状态码 | 说明                    |
| -------- | --------------------- |
| 200      | GET 请求成功              |
| 201      | POST/PUT/PATCH 修改数据成功 |
| 204      | DELETE 删除数据成功         |
| --       | --                    |
| 400      | 请求格式错误，如“只支持json格式”   |
| 404      | 资源不存在                 |
| 422      | 请求信息校验错误              |
| 429      | 请求过于频繁                |
| --       | --                    |
| 401      | 访问令牌没有提供，或者无效         |
| 403      | 访问令牌有效，但没有接口权限        |
| --       | --                    |
| 500      | 服务器内部抛出错误（代码错误）       |



## 错误处理

如果状态码是``4xx``，在 response body 中通过 ``message`` 给出明确的信息。如 ``HTTP 400``，返回：
```
{"message":"Problems parsing JSON"}
```
如果状态码是`` HTTP 422``（表单验证错误）, 除了 ``message``外，还需通过``errors``给出所有字段的错误信息，能够方便调用方快速排错，如
```
{
    "message": "The given data was invalid.",
    "errors": {
        "title": [
            "请填写显示名"
        ],
        "name": [
            "请填写权限标识"
        ],
        "pid": [
            "请选择父级权限"
        ],
        "type": [
            "请选择权限类型"
        ]
    }
}
```

## 成功返回结果
针对不同操作，服务器向用户返回的结果应该符合以下规范。

- GET /collection：返回资源对象的列表（数组）
- GET /collection/resource：返回单个资源对象
- POST /collection：返回新生成的资源对象
- PUT /collection/resource：返回完整的资源对象
- DELETE /collection/resource：返回一个空文档



## Hypermedia API

Hypermedia API的设计被称为HATEOAS。

RESTful API 最好做到 Hypermedia，即返回结果中提供链接，连向其他 API 方法，使得用户不查文档，也知道下一步应该做什么。

比如，当用户向 api.example.com 的根目录发出请求，会得到这样一个文档。
```
{
    "link": {
        "rel": "collection https://www.example.com/zoos",
        "href": "https://api.example.com/zoos",
        "title": "List of zoos",
        "type": "application/vnd.yourformat+json"
    }
}
```
上面代码表示，文档中有一个 link 属性，用户读取这个属性就知道下一步该调用什么 API 了。rel 表示这个 API 与当前网址的关系（collection 关系，并给出该 collection 的网址），href 表示 API 的路径，title 表示 API 的标题，type 表示返回类型。



## 接口请求限制

如果对访问的次数不加控制，很可能会造成 API 被滥用，甚至被 DDos 攻击。根据使用者不同的身份对其进行限流，可以防止这些情况，减少服务器的压力。状态码返回``HTTP 429``



## 接口文档

前后端分离后，接口文档是前后端技术沟通的重要依据，好的接口文档能提高团队的开发效率，接口文档必须日常维护，保持最新版本。
