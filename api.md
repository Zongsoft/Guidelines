## 前言

根据 [REST 成熟度模型](https://martinfowler.com/articles/richardsonMaturityModel.html "Richardson Maturity Model")，本规范基于 `Level 2` 等级。

![Glory of REST](https://martinfowler.com/articles/images/richardsonMaturityModel/overview.png "Glory of REST")


## 设计概要

RESTful API 是对资源的表述，通过 HTTP 协议相关要素表达对资源访问的语义化表征。我们设计一个 RESTful 的 API 通常通过如下步骤：


### 1. 资源定义

识别出业务领域中的资源，它是对所提供服务进行分解、组合后的一个被命名的抽象概念。资源通常是以名词定义的，应避免使用动词来表示资源。


### 2. 标识资源

资源定义好后，需要采用 URI 来标识资源，一个资源可能有多种 URI 标识，但是一个 URI 标识只可能对应一个资源。

> 譬如 `任务` 和 `我的任务`
> - 任务 `/tasks`
>
> - 我的任务 _(假设我的用户编号是 `100`)_
>   - `/tasks/mine`
>   - `/users/100/tasks`


#### 命名规则

对于多单词的资源名建议采用蛇形风格(短横线分隔符)。

关于资源名是单数还是复数，则应该符合该资源所蕴含的数量语义，在实际技术实现中如果不能确保每个资源的准确含义，那应该尽量统一单复数命名（即要么全采用复数或单数形式，切不要单、复数形式混搭）。


### 3. 方法映射

- 使用 HTTP 方法来映射资源的操作(CRUD)
- 使用 HTTP 头来承载必要的请求/响应的元数据
- 使用 HTTP 状态码来表示服务的响应状态

Http Method | Safe | Idempotent | Description
:----------:|:----:|:----------:|:-----------
GET         | ✔ | ✔ | 获取资源
PUT         | ✘ | ✔ | 完整更新（_如果不存在则新增？_）
POST        | ✘ | ✘ | 创建资源（以及未符合其他方法语义的操作）
PATCH       | ✘ | ✘ | 部分更新
DELETE      | ✘ | ✔ | 删除资源

> - 安全性(**S**afety)：操作不会对资源产生副作用，不会修改资源。
> - 幂等性(**I**dempotent)：执行一次和重复执行多次，结果是一样的。


在某些情况下不是所有操作都能恰如其分的映射到 HTTP 方法，我们应将操作目标抽象成一种子资源，譬如禁用某种资源的操作，就可以将被禁用的资源作为其子资源看待。

- 禁用调度器：
    - `[PUT] /schedulers/disabled/{id}`
- 启用调度器
    - `[DELETE] /schedulers/disabled/{id}`
- 获取被禁用的调度器集
    - `[GET] /schedulers/disabled`

这个世界总是充满意外的惊喜，上面的列子也可以采用 `URI/动词` 的组合方式：
- 禁用调度器
    - `[POST] /schedulers/{id}/disable`
- 启用调度器
    - `[POST] /schedulers/{id}/enable`


## 版本标识

因为业务调整导致 API 的参数和结果变化，造成兼容性被破坏，这时需要进行 API 版本标识，版本标识大致有如下三种方法。

1. 将 API 版本信息在 URL 的路径中：
```
[GET] api.zongsoft.com/v1/users/100
```

2. 采用自定义的 HTTP 头：
```
[GET] api.zongsoft.com/users/100
api-version: v1
```

3. 采用 HTTP 的 Accept 头：
```
[GET] api.zongsoft.com/users/100
Accept: application/vnd.v1+json
```

> 对于新系统，建议采用 `方法二`，所有请求默认加上版本标识以便后续向下兼容。


## 状态码

RESTful 服务采用 HTTP 状态码指定方法的执行结果。

状态码 | 描述说明
------|----------
200 OK | 执行成功。
201 Created | 资源创建成功，应该返回 **L**ocation 响应头来提供新创建资源的URL地址。
202 Accepted | 服务端已经接受了请求，但是并未处理完成，适用于一些异步操作，譬如批量导出文件等。
204 No Content | 执行成功，但是不会在响应内容无数据。
400 Bad Request | 客户端请求错误，客户端应该根据响应内容中的错误描述来修改请求，然后才能再次发送。
401 Unauthorized | 客户端未提供授权信息。
403 Forbidden | 客户端无权访问 _（客户端已经提供了授权信息，但是权限不够）_。
404 Not Found | 客户端请求的资源不存在。
405 Method Not Allowed | 客户端使用了不被允许的方法。比如某个操作只允许 POST 但是客户端采用了 PUT。
406 Not Acceptable | 客户端发送的 **A**ccept 不被支持。<br/>比如客户端发送了 `Accept:application/xml`，但是服务器只支持 `application/json`。
409 Conflict | 客户端提交的数据过于陈旧，和服务端的存在冲突，需要客户端重新获取最新的资源再发起请求。
415 Unsupported Media Type | 客户端发送的 **C**ontent-**T**ype 不被支持。<br/>比如客户端发送了`Content-Type:application/xml`，但是服务器只支持 `application/json`。
429 Too Many Requests | 客户端在指定的时间内发送了太多次数的请求。
500 Internal Server Error | 服务器遇见了未知的内部错误。
501 Not Implemented | 服务器还未实现此项功能。
503 Service Unavailable | 服务器繁忙，无法处理客户端的请求。


## 错误响应

当服务执行失败，仅凭 HTTP 状态码并不足以表达失败的原因，因此当 HTTP 状态码值不为 `2xx` 段时必须定义一套表达失败信息的结构。

```json
[
    {
        "error":"ArgumentNull",
        "field":"name",
        "message":"指定的某某名称不能为空。"
    },
    {
        "error":"ArgumentOutOfRange",
        "field":"creation",
        "message":"指定的创建时间超出范围。"
    }
]
```

> 注：如果错误未限定具体字段，则其中 `field` 属性可以缺失。


## 公共参数

公共参数指通过 URL 中查询字符串指定的部分。

- 分页参数：`page`，语法：`{pageIndex}[/{pageSize}]`
    > - `pageIndex` 和 `pageSize` 的值均为正整数，其中 `pageSize` 可忽略 _（即为系统默认值）_。
    > - 禁用分页即 `page=0`

- 排序参数：`sort`，语法：`{field}|(asc|desc)[,...]`

- 集合参数，使用小括号标注，元素间采用逗号(`,`)分隔。

- 区间参数，使用小括号标注，起止值采用波浪线(`~`)分隔，起止值可以缺少其中一个，缺失项采用星号(`*`)占位，值类型可以是数字或日期格式。


示例：
```
GET /users?
    page=2/10&
    sort=creation|desc,age&
    status=(1,3)&
    grade=(1~5)&
    creation=(2019-1-1~*)
```


## 公共头

- `x-json-casing`
    > 指定返回 JSON 键的命名规范，支持的值有：`camel` 和 `pascal`
- `x-json-behaviors`
    > 指定返回的 JSON 的选项，支持的值有：`ignores:null` 和 `ignores:default`
- `x-json-datetime`
    > 指定返回的 JSON 的日期时间格式，譬如：`yyyy-MM-dd HH:mm:ss`
- `x-data-schema`
    > 指定当前操作的数据模式，有关数据模式的详细定义请参考 [Zongsoft.Data](https://github.com/Zongsoft/Zongsoft.Data) 项目文档。


## 响应内容

响应内容应尽量简洁直白，避免无谓的嵌套结构。对于支持 `x-data-schema` 公共头的查询操作的内容结构如下：

```json
/* 单项内容 */
{
    "$":
    {
        ...
    },
    "$$":
    {
        "$children":"1/1"
    }
}

/* 集合内容 */
{
    "$":[
        {...},
        {...}
    ],
    "$$":
    {
        "$":"1/100(1989)",
        "$children":"1/1"
    }
}
```