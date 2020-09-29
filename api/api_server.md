**应用在FinChat应用市场上架后，可通过服务端API提供的能力，实现业务需求。**

## 接口说明

### 格式

**协议：HTTPS**

**数据格式：非文件类型时，统一为JSON**

**编码：UTF-8**

**头部：非文件类型时，POST和PUT请求的Content-Type统一为application/json，app_access_token放在Authorization中，格式为"Bearer ${app_access_token}"**

### 术语
* app_id：应用的id，创建应用时生成。
* secret：应用的密钥，用于获取app_access_token。
* app_access_token：开放平台用户身份认证以外的接口的调用凭证。
* fcid：FinChat的用户id。
* room_id：聊天会话所在的房间id。

### 错误约定
错误发生时，会按照一定格式返回错误信息，优先级高于http状态码。错误为FC_UNKNOWN时，应根据http状态码判断错误。

#### 格式

| 参数    | 类型   | 说明                                                         |
| ------- | ------ | ------------------------------------------------------------ |
| errcode | string | 错误码，全局唯一的字符串，具有一定的可读性，对错误的简略描述 |
| error   | string | 错误描述，具有可读性，错误的详细描述                         |
| data    | Object | 错误发生时，可能返回的数据                                   |

#### 全局错误码 

| errcode                           | error                      |
| --------------------------------- | -------------------------- |
| FC_BAD_JSON                       | 请求参数错误               |
| FC_APP_NOT_FOUND                  | 应用不存在                 |
| FC_APP_NOT_PUBLISHED              | 应用未上架                 |
| FC_INVALID_SECRET                 | secret错误                 |
| FC_INVALID_ACCESS_TOKEN           | access_token错误           |
| FC_MESSAGE_FCID_LIMIT_EXCEEDED    | 发送消息的用户个数超出限制 |
| FC_MESSAGE_ROOM_ID_LIMIT_EXCEEDED | 发送消息的房间个数超出限制 |
| FC_ROOM_NOT_FOUND                 | 房间不存在                 |
| FC_ROOM_IS_NOT_FEDERATE           | 房间不支持跨域             |
| FC_FCID_NOT_FOUND                 | 用户不存在                 |
| FC_FCID_ALREADY_IN_ROOM           | 用户已经加入房间           |
| FC_FCID_NOT_IN_ROOM               | 用户未加入房间             |
| FC_NOT_FOUND                      | 资源不存在                 |
| FC_FORBIDDEN                      | 应用无权限                 |
| FC_INTERNAL_ERROR                 | 内部错误                   |
| FC_UNKNOWN                        | 未知错误                   |


## 授权

### 获取app_access_token
app_access_token为调用服务端API的唯一凭证，应用必须通过此接口获得app_access_token才能调用其他API。

**请求方式：POST**

**请求地址：/api/v1/finchat/open-api/auth/token**

**请求头部：**

| 参数         | 值               |
| ------------ | ---------------- |
| Content-Type | application/json |

**请求参数说明：**

| 参数   | 类型   | 是否必须 | 说明     | 默认值 |
| ------ | ------ | -------- | -------- | ------ |
| app_id | string | 是       | 应用id   | -      |
| secret | string | 是       | 应用密钥 | -      |

**请求示例：**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "secret": "90fa4652-5cd1-4280-ae20-e3d381f9b1fa"
}
```

**响应参数说明：**

| 参数             | 类型   | 说明                                           |
| ---------------- | ------ | ---------------------------------------------- |
| app_access_token | string | app_access_token，作为其他所有API调用凭证      |
| expires_in       | int64  | 过期时间，单位为秒，过期后需要重新调用接口获取 |
| token_type       | string | token类型，值为Bearer                          |

**响应示例：**
```
HTTP/1.1 200 OK
{
    "app_access_token": "18eac85d35a26f989317ad4f02e8bbbb"
    "expires_in": 3600,
    "token_type": "Bearer"
}
```


## 事件订阅
开发者在上架应用的过程中，可以根据需要，指定要订阅的事件，应用上架后，相应的事件发生时，FinChat开放平台会将事件推送至对应的地址中，应用需要响应200状态码。

### 回调接口

**响应和重试机制说明**
* 机器人收到事件推送时，必须在30秒内响应200状态码，其他情况都视为失败（包括网络不通）。
* 推送失败时，会执行重试推送，重试的时间间隔会逐渐增加，最多间隔10分钟。
* 以第1次推送失败的时间开始计算“失败累计时间”，推送成功时“失败累计时间”清零。
* 如果“失败累计时间”超过1小时，之前未推送成功的消息都会丢弃。
* 如果“失败累计时间”超过1小时，新产生的消息，推送失败时不会再重试，直接丢弃。

**注意事项**
* 推送失败原因可能有多种，机器人服务如有需要，可以选择使用event_id进行去重。
* 机器人服务为了不丢失消息，应该避免断线超过1小时。


**请求方式：PUT**

**请求地址：开发者填写的地址**

**请求头部：**

| key          | value            |
| ------------ | ---------------- |
| Content-Type | application/json |

**请求参数说明：**

| 参数   | 类型   | 说明            |
| ------ | ------ | --------------- |
| app_id | string | 应用id          |
| token  | string | 用于校验的token |
| event  | Object | 事件详情        |

**event**

| 参数         | 类型      | 是否必须 | 说明                   |
| ------------ | -------- | -------- | ---------------------- |
| event_id     | string   | 是       | 事件id                 |
| content      | Object   | 是       | 事件内容               |
| type         | string   | 是       | 事件类型               |
| timestamp    | int64    | 是       | 事件发生的事件，单位ms |
| sender       | string   | 是       | 事件发送者             |
| room_id      | string   | 否       | 事件相关的房间id       |
| room_type    | string   | 否       | 事件相关的房间类型     |
| receivers    | [string] | 否       | 收到消息的用户列表     |
| origin_event | string   | 否       | 原消息id               |
| webhook_id   | string   | 否       | webhook id             |
| fcid         | string   | 否       | 事件相关的用户id       |
| unsigned     | Object   | 否       | 扩展字段               |

**事件列表**
* 收到直聊房间消息或者群聊房间@应用机器人账号的消息（机器人需要在房间内）
* 获取所有房间的所有消息（机器人可以不在房间内，此权限为高级权限，仅限内部应用使用）
* 用户更新CUI卡片（机器人需要在房间内）
* webhook（机器人需要在房间内）
* 用户和应用机器人账号的直聊房间首次被创建 
* 用户申请使用应用（仅供默认的管理应用使用）
* 应用上下架流程状态改变（仅供默认的管理应用使用）
* 应用可见性修改（仅供默认的管理应用使用）
* 敏感词通知（仅供默认的管理应用使用）


**示例**

**直聊房间文本消息**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
            "body": "This is an example text message", // 消息体
            "msgtype": "m.text" // 文本消息
        },
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "room_id": "!jEsUZKDJdhlrceRyVU:example.org",
        "room_type": "direct_room", // 直聊房间
        "sender": "@example:example.org",
        "type": "m.room.message"
    }
}
```

**群聊房间@应用机器人账号消息**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
            "msgtype": "m.alert", // @应用机器人账号消息
            "body": "@FinChat小助手 test" // 消息体
        },
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "room_id": "!jEsUZKDJdhlrceRyVU:example.org",
        "room_type": "multi_room",
        "sender": "@example:example.org",
        "type": "m.room.message"
    }
}
```

**所有房间的所有消息**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
            "body": "This is an example text message", // 消息体
            "msgtype": "m.text" // 文本消息，还可能是其他消息类型
            // 其他字段，参考：https://matrix.org/docs/spec/client_server/r0.6.1#m-room-message-msgtypes
        },
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "room_id": "!jEsUZKDJdhlrceRyVU:example.org",
        "room_type": "direct_room", // 直聊房间
        "sender": "@example:example.org",
        "receivers": [              // 接受到消息的成员列表
            "@aa:example.org",
            "@bb:example.com"
        ]
        "type": "m.room.message"
    }
}
```

**用户和应用机器人账号直聊房间首次被创建**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
            "creator": "@finchat-help-bot:example.org", // 房间创建者账号id
        },
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "room_id": "!jEsUZKDJdhlrceRyVU:example.org",
        "room_type": "direct_room",
        "sender": "",
        "fcid": "@alice:example.org", // 加入直聊房间的用户id
        "type": "finstore.room.create"
    }
}
```

**用户操作CUI表单**

content.info的键值对，具体内容请参考[CUI消息格式](../backend/bot.html#_4-会话卡片)文档
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
            "action": "submit", // 目前仅支持submit
            "info": {
                "key1": "value1", // 更改元素的键值
                "key2": "value2",
                "key3": "value3"
            }
        },
        "event_id": "$143273582443PhrSn:example.org",
        "origin_event": "$153243582843Pxrdn:example.org", // 原CUI消息id
        "timestamp": 1432735824653,
        "room_id": "!jEsUZKDJdhlrceRyVU:example.org",
        "room_type": "direct_room",
        "sender": "@alice:example.org", // 提交表单的用户id
        "fcid": "@alice:example.org", // 表示直聊房间的用户id，群聊房间为空
        "type": "finstore.room.interactcui"
    }
}
```

**webhook**

```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {}, // 相应网站推送的内容
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "webhook_id": "461cf042d9eedaa60d445f26dc747d5e",
        "room_type": "multi_room",
        "sender": "",
        "type": "finstore.app.webhook"
    }
}
```

**用户申请使用应用**

```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
			"info": [
				{
					"app_id": "461cf042d9eedaa60d445f26dc747d5e",
					"app_name": "测试机器人",
					"fcid": "@test001:example.com",
					"username": "测试账号001"
				}
			]
        },
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "sender": "@alice:example.org",
        "type": "finstore.app.application"
    }
}
```

**应用上下架流程状态改变**

```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
            "app_id": "461cf042d9eedaa60d445f26dc747d5e", // 应用id
            "app_name": "测试机器人",
            "from_status": "Publishing", // 之前的状态
            "operation": "ApprovePublishRequest", // 执行的操作
            "to_status": "PublishApproved", // 修改后的状态
            "developer_id": "@alice:example.org", // 应用开发者id
            "developer_name": "爱丽丝", // 应用开发者的用户名
            "operator_id": "@admin:example.org", // 操作者id
            "operator_name": "管理员", // 操作者用户名
            "operator_role": "administrator" // 操作者角色（developer/administrator）
        },
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "sender": "@admin:example.org",
        "type": "finstore.app.status"
    }
}
```

**应用状态列表**

| 值                | 说明   |
| ----------------- | ------ |
| InDevelopment     | 开发中 |
| Publishing        | 上架审核中 |
| PublishWithdrawed | 上架申请已撤回 |
| PublishApproved   | 上架申请已通过  |
| PublishRejected   | 上架申请已驳回 |
| Published         | 已上架 |
| Unpublishing      | 下架审核中 |
| UnpublishApproved | 下架申请已通过 |
| UnpublishRejected | 下架申请已驳回 |
| Unpublished       | 已下架 |

**应用操作列表**

| 值                            | 说明   |
| ----------------------------- | ------ |
| SubmitPublishRequest          | 提交上架申请 |
| WithdrawPublishRequest        | 撤回上架申请 |
| Publish                       | 上架 |
| ApprovePublishRequest         | 通过上架申请 |
| RejectPublishRequest          | 驳回上架申请 |
| SubmitUnpublishRequest        | 提交下架申请 |
| Unpublish                     | 下架 |
| ApproveUnpublishRequest       | 通过下架申请 |
| RejectUnpublishRequest        | 驳回下架申请 |


**应用可见性修改**

```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
            "app_id": "461cf042d9eedaa60d445f26dc747d5e", // 应用id
            "app_name": "测试机器人",
            "administrator_id": "@admin:example.org", // 管理员id
            "administrator_name": "管理员", // 管理员用户名
            "added_fcids": ["@alice:example.com"], // 新增可见的用户列表
            "removed_fcids": ["@alice:example.com"], // 不再可见的用户列表
        },
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "sender": "@admin:example.org",
        "type": "finstore.app.visibility"
    }
}
```

**敏感词通知**

```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
            "sender_id": "@aa:example.com", // 消息发送者fcid
            "sender_name": "张三", // 消息发送者姓名
            "room_id": "!jEsUZKDJdhlrceRyVU:example.org", // 房间id
            "room_name": "张三李四的房间", // 房间名称
            "timestamp": 52834096829, // 消息发送时间，单位ms
            "body": "xxxyyyzzz", // 消息文本内容 
            "sensitive_word": "xxx", // 敏感词
            "disposal": "禁止发送", // 触发处置
            "receivers": ["@bb.example.com"] // 接受通知的成员fcid列表
        },
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "sender": "@admin:example.org",
        "type": "observe.message.sensitivity"
    }
}
```


## 快捷搜索
开发者在上架应用的过程中，可以开启快捷搜索，允许用户直接搜索应用内的内容。应用上架后，用户发送搜索请求，FinChat开放平台会将事件推送至对应的地址中，应用返回搜索结果给用户。

### 回调接口

**应用收到搜索请求后，需要在5秒内返回结果，同时返回 http 200 状态码表示成功，否则视为失败**
**搜索请求为一次性请求，不管成功与失败，都不会重复推送**

**请求方式：POST**

**请求地址：开发者填写的地址**

**请求头部：**

| key          | value            |
| ------------ | ---------------- |
| Content-Type | application/json |

**请求参数说明：**

| 参数   | 类型   | 说明            |
| ------ | ------ | --------------- |
| app_id | string | 应用id          |
| token  | string | 用于校验的token |
| event  | Object | 事件详情        |

**event**

| 参数      | 类型   | 是否必须 | 说明                   |
| --------- | ------ | -------- | ---------------------- |
| event_id  | string | 是       | 事件id                 |
| content   | Object | 是       | 事件内容               |
| type      | string | 是       | 事件类型               |
| timestamp | int64  | 是       | 事件发生的事件，单位ms |
| sender    | string | 是       | 事件发送者             |
| unsigned  | Object | 否       | 扩展字段               |

**事件列表**
* 搜索请求

**示例**

**搜索请求**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "token": "2g7als3DgPW6Xp1xEpmcvgVhQG621bFY",
    "event": {
        "content": {
            "search_text": "my text",
            "page_no": 0,
            "page_size": 10
        },
        "event_id": "$143273582443PhrSn:example.org",
        "timestamp": 1432735824653,
        "sender": "@example:example.org",
        "type": "finstore.shortcut.search"
    }
}
```

**响应参数说明：**

| 参数    | 类型   | 说明                     |
| ------- | ------ | ------------------------ |
| layout  | string | 布局类型，目前仅支持list |
| content | Object | 响应内容                 |

**content**

| 参数     | 类型            | 是否必须 | 说明                             |
| -------- | --------------- | -------- | -------------------------------- |
| total    | int             | 是       | 搜索结果记录总数                 |
| list     | [search_result] | 是       | 搜索结果列表                     |
| opentype | string          | 是       | 列表打开方式, inapp or inbrowser |


**search_result**

| 参数        | 类型   | 是否必须 | 说明                                     |
| ----------- | ------ | -------- | ---------------------------------------- |
| type        | string | 是       | 类型，目前仅支持text                     |
| title       | string | 是       | 标题                                     |
| description | string | 是       | 内容                                     |
| url         | string | 否       | 点击列表时打开的 url, 无则不触发任何操作 |

**示例**

**搜索结果**
```
{
    "layout": "list",
    "content": {
        "total": 20,
        "list": [
            {
                "type": "text",
                "title": "保险",
                "description": "一种东西"
            }
        ]
    }
}
```


## 消息

### 发送消息

**支持的消息类型列表**
* 文本消息
* 文本卡片消息
* 图片消息
* 文件消息
* CUI消息

**请求方式：POST**

**请求地址：/api/v1/finchat/open-api/message/send**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数     | 类型     | 是否必须 | 说明                                                                                                     | 默认值 |
| -------- | -------- | -------- | -------------------------------------------------------------------------------------------------------- | ------ |
| app_id   | string   | 是       | 应用id                                                                                                   | -      |
| fcids    | [string] | 否       | 接收消息的用户id列表，将往应用与用户的直聊房间发送消息，直聊房间必须已存在；room_ids不为空时，忽略此参数 | -      |
| room_ids | [string] | 否       | 房间id列表，可以是直聊房间也可以是群聊房间；不为空时，忽略fcids                                          | -      |
| content  | Object   | 是       | 消息内容                                                                                                 | -      |

**请求示例：**

**发送文本**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "room_ids": [
        "!112321588723954859:example.org"
    ],
    "content": {
        "msgtype": "m.text", // 文本消息类型
        "body": "hello world" // 消息体
    }
}
```

**发送文件**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "room_ids": [
        "!112321588723954859:example.org"
    ],
    "content": {
        "msgtype": "m.file", // 文件消息类型
        "body": "test.pdf" // 文件名
        "info": {
            "mimetype": "", // MIME类型，文件为空字符串
            "size": 1234242 // 文件大小
        },
        "o_url": "mxc://example.org/66696e6f6765656b732e636c75625e6f4670efda61000110b517" // 文件路径，上传文件时返回
    }
}
```

**发送图片**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "room_ids": [
        "!112321588723954859:example.org"
    ],
    "content": {
        "msgtype": "m.image", // 图片消息类型
        "body": "test.png" // 图片名
        "info": {
            "mimetype": "image/png", // MIME类型
            "size": 1234242 // 文件大小
            "h": 1334, // 图片高度
            "w": 750 // 图片宽度
        },
        "o_url": "mxc://example.org/66696e6f6765656b732e636c75625e6f4670efda61000110b517" // 文件路径，上传文件时返回
    }
}
```

**发送文本卡片**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "room_ids": [
        "!112321588723954859:example.org"
    ],
    "content": {
        "msgtype": "m.textcard", // 文本卡片消息类型
        "body": "" // 文本卡片为空字符串
        "info": {
            "title": "测试卡片", // 卡片标题
            "url": "https://example.org/home // 跳转链接
            "description": "测试卡片的内容", // 卡片详情
            "btntext": "发送" // 卡片按钮内容
            "opentype": "inapp", // inapp 或 inbrowser 打开方式
        },
        "o_url": "mxc://example.org/66696e6f6765656b732e636c75625e6f4670efda61000110b517" // 文件路径，上传文件时返回
    }
}
```

**发送CUI**

content.info字段具体内容请参考[CUI消息格式](../backend/bot.html#_4-会话卡片)章节

```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "room_id": "!112321588723954859:example.org",
    "content": {
        "body": "[交互卡片]",
        "info": {},
        "msgtype": "m.cui",
        "version": "0.1.0",
    }
}
```


**响应参数说明：**

| 参数           | 类型             | 说明             |
| -------------- | ---------------- | ---------------- |
| room_id_result | [room_id_result] | 房间发送结果列表 |
| fcid_result    | [fcid_result]    | 用户发送结果列表 |

**room_id_result**

| 参数     | 类型   | 说明                           |
| -------- | ------ | ------------------------------ |
| room_id  | string | 房间id                         |
| errcode  | string | 错误码，成功时为空             |
| error    | string | 错误详情，成功时为空           |
| event_id | string | 发送成功时的消息id，失败时为空 |

**fcid_result**

| 参数     | 类型   | 说明                           |
| -------- | ------ | ------------------------------ |
| fcid     | string | 用户id                         |
| errcode  | string | 错误码，成功时为空             |
| error    | string | 错误详情，成功时为空           |
| event_id | string | 发送成功时的消息id，失败时为空 |

**响应示例：**
```
HTTP/1.1 200 OK
{
    "room_id_result": [
        {
            "room_id": "!132221587624558592:example.org",
            "errcode": "FC_INTERNAL_ERROR",
            "error": "内部错误"
        }
    ]
}
```

**响应示例：**
```
HTTP/1.1 200 OK
{
    "fcid_result": [
        {
            "fcid": "@test:example.org",
            "event_id": "$143273582443PhrSn:example.org"
        }
    ]
}
```

### 发送webhook消息

**支持的消息类型列表**
* 文本消息
* 文本卡片消息
* 图片消息
* 文件消息
* CUI消息

**请求方式：POST**

**请求地址：/api/v1/finchat/open-api/message/webhook/send**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数       | 类型   | 是否必须 | 说明            | 默认值 |
| ---------- | ------ | -------- | --------------- | ------ |
| app_id     | string | 是       | 应用id          | -      |
| webhook_id | string | 是       | webhook机器人id | -      |
| content    | Object | 是       | 消息内容        | -      |

**请求示例：**

**发送文本**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "webhook_id": "5df29mec9ccfd002ccd28076",
    "content": {
        "msgtype": "m.text", // 文本消息类型
        "body": "hello world" // 消息体
    }
}
```

**发送文件**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "webhook_id": "5df29mec9ccfd002ccd28076",
    "content": {
        "msgtype": "m.file", // 文件消息类型
        "body": "test.pdf" // 文件名
        "info": {
            "mimetype": "", // MIME类型，文件为空字符串
            "size": 1234242 // 文件大小
        },
        "o_url": "mxc://example.org/66696e6f6765656b732e636c75625e6f4670efda61000110b517" // 文件路径，上传文件时返回
    }
}
```

**发送图片**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "webhook_id": "5df29mec9ccfd002ccd28076",
    "content": {
        "msgtype": "m.image", // 图片消息类型
        "body": "test.png" // 图片名
        "info": {
            "mimetype": "image/png", // MIME类型
            "size": 1234242 // 文件大小
            "h": 1334, // 图片高度
            "w": 750 // 图片宽度
        },
        "o_url": "mxc://example.org/66696e6f6765656b732e636c75625e6f4670efda61000110b517" // 文件路径，上传文件时返回
    }
}
```

**发送文本卡片**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "webhook_id": "5df29mec9ccfd002ccd28076",
    "content": {
        "msgtype": "m.textcard", // 文本卡片消息类型
        "body": "" // 文本卡片为空字符串
        "info": {
            "title": "测试卡片", // 卡片标题
            "url": "https://example.org/home // 跳转链接
            "description": "测试卡片的内容", // 卡片详情
            "btntext": "发送" // 卡片按钮内容
        },
        "o_url": "mxc://example.org/66696e6f6765656b732e636c75625e6f4670efda61000110b517" // 文件路径，上传文件时返回
    }
}
```

**发送CUI**

content.info字段具体内容请参考[CUI消息格式](../backend/bot.html#_4-会话卡片)章节

```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "webhook_id": "5df29mec9ccfd002ccd28076",
    "content": {
        "body": "[交互卡片]",
        "info": {},
        "msgtype": "m.cui",
        "version": "0.1.0",
    }
}
```


**响应参数说明：**

| 参数     | 类型   | 说明   |
| -------- | ------ | ------ |
| event_id | string | 消息id |

**响应示例：**
```
HTTP/1.1 200 OK
{
    "event_id": "$143273582443PhrSn:example.org"
}
```


### 以用户名义发送消息

**以用户名义发送的消息，需要获取用户授权，授权流程还在完善中，目前先开放接口调用。**

**支持的消息类型列表**
* 文本消息
* 文本卡片消息
* 图片消息
* 文件消息

**请求方式：POST**

**请求地址：/api/v1/finchat/privilege-api/message/send-as-user**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数     | 类型     | 是否必须 | 说明                                                                                                                               | 默认值 |
| -------- | -------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------ |
| app_id   | string   | 是       | 应用id                                                                                                                             | -      |
| sender   | string   | 是       | 以此用户id名义发送消息，此用户需要和应用在同一个域                                                                                 | -      |
| topic    | string   | 否       | 指定fcids时，可设置对应的直聊房间topic，对应的房间列表都设置同一个topic，不需要修改房间topic时，无须指定此字段                     | -      |
| fcids    | [string] | 否       | 接收消息的用户id列表，将替sender往sender与列表中用户之间的直聊房间发送消息，房间不存在时，会自动创建；room_ids不为空时，忽略此参数 | -      |
| room_ids | [string] | 否       | 房间id列表，可以是直聊房间也可以是群聊房间；不为空时，忽略fcids                                                                    | -      |
| content  | Object   | 是       | 消息内容                                                                                                                           | -      |

**请求示例：**

**发送文本**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "sender": "@user001:example.org",
    "room_ids": [
        "!112321588723954859:example.org"
    ],
    "content": {
        "msgtype": "m.text", // 文本消息类型
        "body": "hello world" // 消息体
    }
}
```

**发送文件**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "sender": "@user001:example.org",
    "room_ids": [
        "!112321588723954859:example.org"
    ],
    "content": {
        "msgtype": "m.file", // 文件消息类型
        "body": "test.pdf" // 文件名
        "info": {
            "mimetype": "", // MIME类型，文件为空字符串
            "size": 1234242 // 文件大小
        },
        "o_url": "mxc://example.org/66696e6f6765656b732e636c75625e6f4670efda61000110b517" // 文件路径，上传文件时返回
    }
}
```

**发送图片**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "sender": "@user001:example.org",
    "room_ids": [
        "!112321588723954859:example.org"
    ],
    "content": {
        "msgtype": "m.image", // 图片消息类型
        "body": "test.png" // 图片名
        "info": {
            "mimetype": "image/png", // MIME类型
            "size": 1234242 // 文件大小
            "h": 1334, // 图片高度
            "w": 750 // 图片宽度
        },
        "o_url": "mxc://example.org/66696e6f6765656b732e636c75625e6f4670efda61000110b517" // 文件路径，上传文件时返回
    }
}
```

**发送文本卡片**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "sender": "@user001:example.org",
    "room_ids": [
        "!112321588723954859:example.org"
    ],
    "content": {
        "msgtype": "m.textcard", // 文本卡片消息类型
        "body": "" // 文本卡片为空字符串
        "info": {
            "title": "测试卡片", // 卡片标题
            "url": "https://example.org/home // 跳转链接
            "description": "测试卡片的内容", // 卡片详情
            "btntext": "发送" // 卡片按钮内容
        },
        "o_url": "mxc://example.org/66696e6f6765656b732e636c75625e6f4670efda61000110b517" // 文件路径，上传文件时返回
    }
}
```


**响应参数说明：**

| 参数           | 类型             | 说明             |
| -------------- | ---------------- | ---------------- |
| room_id_result | [room_id_result] | 房间发送结果列表 |
| fcid_result    | [fcid_result]    | 用户发送结果列表 |

**room_id_result**

| 参数     | 类型   | 说明                           |
| -------- | ------ | ------------------------------ |
| room_id  | string | 房间id                         |
| errcode  | string | 错误码，成功时为空             |
| error    | string | 错误详情，成功时为空           |
| event_id | string | 发送成功时的消息id，失败时为空 |

**fcid_result**

| 参数     | 类型   | 说明                           |
| -------- | ------ | ------------------------------ |
| fcid     | string | 用户id                         |
| errcode  | string | 错误码，成功时为空             |
| error    | string | 错误详情，成功时为空           |
| event_id | string | 发送成功时的消息id，失败时为空 |

**响应示例：**
```
HTTP/1.1 200 OK
{
    "room_id_result": [
        {
            "room_id": "!132221587624558592:example.org",
            "errcode": "FC_INTERNAL_ERROR",
            "error": "内部错误"
        }
    ]
}
```

**响应示例：**
```
HTTP/1.1 200 OK
{
    "fcid_result": [
        {
            "fcid": "@test:example.org",
            "event_id": "$143273582443PhrSn:example.org"
        }
    ]
}
```

### 主动获取回调消息

主动获取存储在服务器的回调消息.
需要将机器人的回调地址设为 **/api/v1/finchat/privilege-api/forwarder/event**

**请求方式：GET**

**请求地址：/api/v1/finchat/privilege-api/forwarder/:app_id/events?since=${since}**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数      | 类型   | 是否必须 | 说明                                       | 默认值 |
| --------- | ------ | -------- | ------------------------------------------ | ------ |
| app_id    | string | 是       | 应用id                                     | -      |
| since     | string | 否       | 已获取到的最新的消息, 为空则表示从未获取过 | -      |
| page_size | int    | 否       | 此次获取的消息条数                         | 20     |

**请求示例：**

**响应参数说明：**

| 参数   | 类型     | 是否必须 | 说明                                                    | 默认值 |
| ------ | -------- | -------- | ------------------------------------------------------- | ------ |
| since  | string   | 是       | 已获取到的最新的消息, 为空则表示从未获取过              | -      |
| events | [Object] | 是       | 消息, 按event_id排序, 具体内容参见[回调接口](#回调接口) | 20     |

**响应示例：**

```
HTTP/1.1 200 OK
{
    "events": [
        {
            "app_id": "5cf29aec6ccfd004ccd28075",
            "content": {
                "action": "submit",
                "info": {
                    "key1": "value1",
                    "key2": "value2",
                    "key3": "value3"
                }
            },
            "event_id": "$123:example.org",
            "type": "finstore.room.interactcui",
            "sender": "@alice:example.org",
            "timestamp": 1432735824653,
            "room_id": "!jEsUZKDJdhlrceRyVU:example.org",
            "room_type": "direct_room",
            "fcid": "@alice:example.org",
            "origin_event": "$153243582843Pxrdn:example.org"
        }
    ],
    "since": "$123:example.org"
}
```

### 更新CUI

**请求方式：PUT**

**请求地址：/api/v1/finchat/open-api/message/update-cui**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数         | 类型     | 是否必须 | 说明                                     | 默认值 |
| ------------ | -------- | -------- | ---------------------------------------- | ------ |
| app_id       | string   | 是       | 应用id                                   | -      |
| fcids        | [string] | 否       | 用户id列表，room_ids不为空时，忽略此参数 | -      |
| room_ids     | [string] | 否       | 房间id列表，不为空时，忽略fcids          | -      |
| origin_event | string   | 是       | 原CUI消息id                              | -      |
| content      | Object   | 是       | 消息内容                                 | -      |

**请求示例：**

content.info字段具体内容请参考[CUI消息格式](../backend/bot.html#_4-会话卡片)章节

```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "room_ids": ["!112321588723954859:example.org"],
    "content": {
        "body": "[交互卡片]",
        "info": {},           
        "msgtype": "m.cui",
        "version": "0.1.0",
    },
    "origin_event": "$22222222:dev.finogeeks.club",
}
```

**响应参数说明：**

| 参数           | 类型             | 说明             |
| -------------- | ---------------- | ---------------- |
| room_id_result | [room_id_result] | 房间发送结果列表 |
| fcid_result    | [fcid_result]    | 用户发送结果列表 |

**room_id_result**

| 参数     | 类型   | 说明                           |
| -------- | ------ | ------------------------------ |
| room_id  | string | 房间id                         |
| errcode  | string | 错误码，成功时为空             |
| error    | string | 错误详情，成功时为空           |
| event_id | string | 发送成功时的消息id，失败时为空 |

**fcid_result**

| 参数     | 类型   | 说明                           |
| -------- | ------ | ------------------------------ |
| fcid     | string | 用户id                         |
| errcode  | string | 错误码，成功时为空             |
| error    | string | 错误详情，成功时为空           |
| event_id | string | 发送成功时的消息id，失败时为空 |

**响应示例：**
```
HTTP/1.1 200 OK
{
    "room_id_result": [
        {
            "room_id": "!132221587624558592:example.org",
            "errcode": "FC_INTERNAL_ERROR",
            "error": "内部错误"
        }
    ]
}
```

**响应示例：**
```
HTTP/1.1 200 OK
{
    "fcid_result": [
        {
            "fcid": "@test:example.org",
            "event_id": "$143273582443PhrSn:example.org"
        }
    ]
}
```


### 上传文件

**请求方式：POST**

**请求地址：/api/v1/finchat/open-api/media/upload?app_id=xxx**

**请求头部：**

| key           | value                    |
| ------------- | ------------------------ |
| Content-Type  | 相应的文件类型           |
| Authorization | "Bearer ${access_token}" |

**请求参数说明：**

| 参数   | 类型   | 是否必须 | 说明   | 默认值 |
| ------ | ------ | -------- | ------ | ------ |
| app_id | string | 是       | 应用id | -      |

**请求体：**
以二进制形式上传

**请求示例：**
```
<bytes>
```

**响应参数说明：**

| 参数        | 类型   | 说明      |
| ----------- | ------ | --------- |
| content_uri | string | 文件的uri |

**响应示例：**
```
{
    "content_uri": "mxc://example.com/AQwafuaFswefuhsfAFAgsw" 
}
```


## 群聊房间

### 创建群聊房间

**请求方式：POST**

**请求地址：/api/v1/finchat/open-api/rooms**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**
| 参数     | 类型     | 是否必须 | 说明                                         | 默认值           |
| -------- | -------- | -------- | -------------------------------------------- | ---------------- |
| app_id   | string   | 是       | 应用id                                       | -                |
| type     | string   | 是       | 房间类型，目前仅支持multi_chat，表示群聊房间 | -                |
| name     | string   | 否       | 房间名称                                     | -                |
| topic    | string   | 否       | 房间主题（群通告）                           | -                |
| history_visibility    | string   | 否       | 历史可见性，shared表示开启，joined表示关闭                           | shared                |
| federate | bool     | 否       | 是否支持跨域                                 | false            |
| owner    | string   | 否       | 群主fcid                                     | 默认机器人为群主 |
| fcids    | [string] | 否       | 加入房间的用户列表                           | -                |

**请求示例：**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "type": "multi_chat",
    "name": "name001",
    "federate": true,
    "owner": "@test001:example.com",
    "fcids": ["@test001:example.com"]
}
```

**响应参数说明：**
| 参数    | 类型   | 说明   |
| ------- | ------ | ------ |
| room_id | string | 房间id |


**响应示例：**
```
HTTP/1.1 201 OK
{
    "room_id": "!112321588723954859:example.org"
}
```

### 修改群聊房间

**请求方式：PUT**

**请求地址：/api/v1/finchat/open-api/rooms/:room_id**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**
| 参数                           | 类型     | 是否必须 | 说明               | 默认值 |
| ------------------------------ | -------- | -------- | ------------------ | ------ |
| app_id                         | string   | 是       | 应用id             | -      |
| room_id                        | string   | 是       | 房间id             | -      |
| name                           | string   | 否       | 房间名称           | -      |
| topic                          | string   | 否       | 房间主题（群通告） | -      |
| history_visibility    | string   | 否       | 历史可见性，shared表示开启，joined表示关闭                           | -                |
| owner                          | string   | 否       | 群主fcid           | -      |
| message_forbidden_except_owner | bool     | 否       | 开启仅群主可发言   | -      |
| invite_forbidden_except_owner  | bool     | 否       | 开启仅群主可加人   | -      |
| fcids_to_invite                | [string] | 否       | 添加成员列表       | -      |
| fcids_to_kick                  | [string] | 否       | 删除成员列表       | -      |

* 参数优先级为：name > topic > history_visibility > owner > message_forbidden_except_owner,invite_forbidden_except_owner > fcids_to_invite,fcids_to_kick
* 指定了优先级在前的参数时，优先级在后的参数将忽略
* 部分操作需要相应的权限，无权限时返回错误

**请求示例：**
```
{
    "app_id": "5cf29aec6ccfd004ccd28075",
    "name": "name002"
}
```

**响应参数说明：**
| 参数                   | 类型     | 说明               |
| ---------------------- | -------- | ------------------ |
| failed_fcids_to_invite | [string] | 添加失败的成员列表 |
| failed_fcids_to_kick   | [string] | 删除失败的成员列表 |

* 仅指定了fcids_to_invite或者fcids_to_kick参数，并且有部分失败时，返回失败的用户列表


**响应示例：**
```
HTTP/1.1 201 OK
{}
```

### 获取群聊房间详情

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/rooms/:room_id?app_id=xxx**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**
| 参数    | 类型   | 是否必须 | 说明   | 默认值 |
| ------- | ------ | -------- | ------ | ------ |
| app_id  | string | 是       | 应用id | -      |
| room_id | string | 是       | 房间id | -      |


**响应参数说明：**
| 参数     | 类型     | 说明               |
| -------- | -------- | ------------------ |
| name     | string   | 房间名称           |
| type     | string   | 房间类型           |
| topic    | string   | 房间主题（群通告） |
| federate | bool     | 是否支持跨域       |
| owner    | string   | 群主               |
| fcids    | [string] | 成员列表           |

**响应示例：**
```
HTTP/1.1 201 OK
{
    "name": "room001",
    "type": "multi_chat",
    "topic": "test",
    "federate": true,
    "owner": "@test001:example.org",
    "fcids": ["@test001:example.org"]
}
```

### 获取应用的群聊列表

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/rooms?app_id=xxx**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**
| 参数   | 类型   | 是否必须 | 说明   | 默认值 |
| ------ | ------ | -------- | ------ | ------ |
| app_id | string | 是       | 应用id | -      |


**响应参数说明：**
房间id列表

**响应示例：**
```
HTTP/1.1 201 OK
[
    "!112321588723954859:example.org",
    "!212426598723954819:example.org"
]
```


## 通讯录信息

### 根据权限名称获取用户列表

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/contact/get-users-by-authority?authority_name={authority_name}&app_id=${app_id}**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数           | 类型   | 是否必须 | 说明     | 默认值 |
| -------------- | ------ | -------- | -------- | ------ |
| authority_name | string | 是       | 权限名称 | -      |
| app_id         | string | 是       | 应用id   | -      |


**响应参数说明：**
返回fcid列表


**响应示例：**
```
HTTP/1.1 200 OK
[
    "@test001:dev.finogeeks.club",
    "@test002:dev.finogeeks.club"
]
```


### 根据机构类型获取机构列表

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/contact/get-organizations-by-type?app_id=${app_id}&fcid=${fcid}&organization_type=${organization_type}**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数                   | 类型   | 是否必须 | 说明               | 默认值 |
| ---------------------- | ------ | -------- | ------------------ | ------ |
| app_id                 | string | 是       | 应用id             | -      |
| fcid                 | string | 是       | 用户id             | -      |
| organization_type          | int | 是       | 机构类型             | -      |


**响应参数说明：**

| 参数              | 类型     | 说明         |
| ----------------- | -------- | ------------ |
| organization_id     | string   | 机构id       |
| organization_name   | string   | 机构名称     |
| organization_type   | int   | 机构类型     |
| dispatch_supervision_organizations   | [string]   | 所属派出机构id列表     |


**响应示例：**
```
HTTP/1.1 200 OK
[
    {
        "organization_id": "00f3",
        "organization_name": "74564564",
        "organization_type": 3,
        "dispatch_supervision_organizations": [
            "00f0"
        ]
    }
]
```


### 获取机构详情

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/contact/organizations/:organization_id?app_id=${app_id}**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数                   | 类型   | 是否必须 | 说明               | 默认值 |
| ---------------------- | ------ | -------- | ------------------ | ------ |
| organization_id          | string | 是       | 机构id             | -      |
| app_id                 | string | 是       | 应用id             | -      |


**响应参数说明：**

| 参数              | 类型     | 说明         |
| ----------------- | -------- | ------------ |
| organization_id     | string   | 机构id       |
| organization_name   | string   | 机构名称     |
| organization_type   | int   | 机构类型     |
| dispatch_supervision_organizations   | [string]   | 所属派出机构id列表     |


**响应示例：**
```
HTTP/1.1 200 OK
{
    "organization_id": "55c22c00-09e9-11ea-ab7d-a15ba0c1027b",
    "organization_name": "九九九",
    "organization_type": 0,
    "dispatch_supervision_organizations": ["0a072670-ca61-11e9-af7f-877a0c9dfbd0"]
}
```

### 获取部门详情

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/contact/departments/:department_id?app_id=${app_id}&include_sub_departments=true**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数                   | 类型   | 是否必须 | 说明               | 默认值 |
| ---------------------- | ------ | -------- | ------------------ | ------ |
| department_id          | string | 是       | 部门id             | -      |
| app_id                 | string | 是       | 应用id             | -      |
| include_sub_deparments | bool   | 否       | 是否返回子部门成员 | false  |


**响应参数说明：**

| 参数              | 类型     | 说明         |
| ----------------- | -------- | ------------ |
| department_id     | string   | 部门id       |
| department_name   | string   | 部门名称     |
| parent_department | string   | 上级部门id   |
| sub_departments   | [string] | 子部门id列表 |
| fcids             | [string] | 成员列表     |


**响应示例：**
```
HTTP/1.1 200 OK
{
    "department_id": "55c22c00-09e9-11ea-ab7d-a15ba0c1027b",
    "department_name": "九九九",
    "parent_department": "0a072670-ca61-11e9-af7f-877a0c9dfbd0",
    "sub_departments": [
        "55c38b90-09e9-11ea-ab7d-a15ba0c1027b",
        "55c49d00-09e9-11ea-ab7d-a15ba0c1027b",
        "030b9770-09ea-11ea-ab7d-a15ba0c1027b",
        "7f5dff90-517d-11ea-8022-2f1b1e84209f"
    ],
    "fcids": [
        "@tai01:dev.finogeeks.club",
        "@qqq1110:dev.finogeeks.club",
        "@test0007:dev.finogeeks.club",
        "@test001:dev.finogeeks.club",
        "@test005:dev.finogeeks.club",
        "@czr02:dev.finogeeks.club"
    ]
}
```


## 用户信息

### 获取用户详情

**接口后续需要用户授权，目前暂时开放使用**

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/users/:fcid?app_id=${app_id}**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数   | 类型   | 是否必须 | 说明   | 默认值 |
| ------ | ------ | -------- | ------ | ------ |
| app_id | string | 是       | 应用id | -      |
| fcid   | string | 是       | 用户id | -      |


**响应参数说明：**

| 参数        | 类型     | 说明       |
| ----------- | -------- | ---------- |
| fcid        | string   | 用户id     |
| account     | string   | 账号       |
| name        | string   | 用户名     |
| gender      | string   | 性别       |
| mobile      | string   | 手机号     |
| landline    | string   | 座机       |
| position    | string   | 职级       |
| email       | string   | 邮箱       |
| job_number  | string   | 工号       |
| roles       | [string] | 角色id列表 |
| departments | [string] | 部门id列表 |
| organization_id | string | 机构id |


**响应示例：**
```
HTTP/1.1 200 OK
{
    "fcid": "@linian002:dev.finogeeks.club",
    "account": "linian002",
    "name": "linian002",
    "gender": "male",
    "mobile": "12346278569",
    "landline": "020-7789898",
    "position": "产品总监",
    "email": "222@qq.com",
    "job_number": "001",
    "roles": [
        "b6539010-ca68-11e9-af7f-877a0c9dfbd0",
        "bc8c4670-ca68-11e9-af7f-877a0c9dfbd0"
    ]
    "departments": [
        "d7646430-53b8-11ea-8719-19c1a365f2ff"
    ],
    "organization_id": "00001"
}
```

### 获取用户好友列表

**接口后续需要用户授权，目前暂时开放使用**

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/users/:fcid/friendships?app_id=${app_id}**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数   | 类型   | 是否必须 | 说明   | 默认值 |
| ------ | ------ | -------- | ------ | ------ |
| app_id | string | 是       | 应用id | -      |
| fcid   | string | 是       | 用户id | -      |


**响应参数说明：**

| 参数    | 类型   | 说明               |
| ------- | ------ | ------------------ |
| to_fcid | string | 好友id             |
| room_id | string | 与好友的直聊房间id |
| is_bot  | bool   | 好友是否机器人     |


**响应示例：**
```
HTTP/1.1 200 OK
[
    {
        "to_fcid": "@vsadf-bot:dev.finogeeks.club",
        "room_id": "!224069434157826048:dev.finogeeks.club",
        "is_bot": true
    },
    {
        "to_fcid": "@linian002:dev.finogeeks.club",
        "room_id": "!230427174354550784:dev.finogeeks.club",
        "is_bot": false
    }
]
```


## 应用信息

### 获取应用加入的直聊房间列表

**接口后续需要用户授权，目前暂时开放使用**

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/apps/:app_id/friendships**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数   | 类型   | 是否必须 | 说明   | 默认值 |
| ------ | ------ | -------- | ------ | ------ |
| app_id | string | 是       | 应用id | -      |


**响应参数说明：**

| 参数    | 类型   | 说明                 |
| ------- | ------ | -------------------- |
| to_fcid | string | 加入直聊房间的用户id |
| room_id | string | 直聊房间id           |


**响应示例：**
```
HTTP/1.1 200 OK
[
    {
        "to_fcid": "@vsadf-bot:dev.finogeeks.club",
        "room_id": "!224069434157826048:dev.finogeeks.club"
    },
    {
        "to_fcid": "@linian002:dev.finogeeks.club",
        "room_id": "!230427174354550784:dev.finogeeks.club"
    }
]
```

### 获取企业成员对应用的可见范围

**请求方式：GET**

**请求地址：/api/v1/finchat/open-api/apps/:app_id/visibility**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |
| Authorization | "Bearer ${app_access_token}" |

**请求参数说明：**

| 参数   | 类型   | 是否必须 | 说明   | 默认值 |
| ------ | ------ | -------- | ------ | ------ |
| app_id | string | 是       | 应用id | -      |


**响应参数说明：**

| 参数        | 类型     | 说明                                |
| ----------- | -------- | ----------------------------------- |
| unlimited   | bool     | 是否全员可见                        |
| departments | [string] | 部门id列表，unlimited为true时不返回 |
| fcids       | [string] | 成员id列表，unlimited为true时不返回 |


**响应示例：**
```
{
    "unlimited": false,
    "fcids": [
        "@rcstest1:dev.finogeeks.club"
    ],
    "departments": [
        "55c22c00-09e9-11ea-ab7d-a15ba0c1027b"
    ]
}
```

## 身份认证

### 概述

* FinChat支持对接实现了OIDC协议的IAM系统，以IAM的账户登录。当前支持的有FinChat IAM以及行业通讯录IAM。
* 用户以IAM方式登录FinChat后，网页应用可以通过OIDC协议，获取当前用户在IAM中的信息，无需用户再次登录，从而实现单点登录和身份认证。

### 术语
| 名称             | 说明                                                                            |
| --------------- | ----------------------------------------------------------------------------   |
| IAM             | Identity and Access Management，身份识别与访问管理，即统一认证授权服务                |
| SSO             | Single Sign On，单点登录，为IAM的一个功能                                          |
| OIDC            | OpenID Connect，用于实现SSO的一种协议                                             |
| HOST            | FinChat服务器地址，客户端登录和调用FinChat相关后端接口的地址                           |
| IAM_HOST        | IAM服务器地址，独立于HOST，请求IAM相关OIDC接口的地址                                  |
| app_id          | 应用id，应用市场上架时生成，对应OIDC的client_id                                      |
| secret          | secret，应用市场上架时生成，对应OIDC的secret                                        |
| redirect_uri    | 重定向uri，应用市场上架时指定，应用的回调地址，用于获取一次性code                        |
| code            | 一次性code，用于获取access_token                                                  |
| access_token    | 从IAM获取的access_token，有效时间较短，用于获取当前用户身份信息，区别于app_access_token  |
| refresh_token   | access_token过期时用于重新获取access_token                                         |


### 流程

**OIDC官方流程图简介**
![](../material/server_api/oidc.png)

**OIDC协议详情参考：[OpenID Connect官方文档](https://openid.net/connect/)**


**FinChat身份认证主要步骤**

1. 应用跳转至IAM授权页面，引导用户授权
2. 默认授权，不会显示授权页面，用户无需点击
3. IAM重定向回应用页面，带上一次性code
4. 应用使用一次性code调用/token接口获取access_token
5. 应用使用access_token调用/userinfo接口获取用户信息


### 接口详情

### 获取IAM_HOST

**请求方式：GET**

**请求地址：{HOST}/api/v1/finchat-control/iam-info**

**请求头部：**

| key           | value                        |
| ------------- | ---------------------------- |
| Content-Type  | application/json             |

**响应参数说明：**

| 参数         | 类型       | 说明                                |
| ----------- | ---------- | ---------------------------------- |
| flows       | [flow]     |  结果列表                           |

**flow**

| 参数         | 类型     | 说明                               |
| ----------- | -------- | --------------------------------- |
| type        | string   | 登录类型，为iam时支持IAM登录          |
| url         | string   | IAM_HOST的值                       |

**响应示例：**
```
{
    "flows": [
        {
            "type": "iam",
            "url": "https://dev-finchat-iam.finogeeks.club/auth"
        }
    ]
}
```


### 获取一次性授权code
网页应用的页面应该跳转到此页面，用户授权后（授权页面隐藏不显示），会重定向至应用指定的重定向地址，并且在URL中返回一次性code

**请求方式：GET**

**请求地址：{IAM_HOST}/auth/realms/master/protocol/openid-connect/auth?client_id=xxx&redirect_uri=xxx&response_type=code&scope=openid profile**

**请求参数说明：**

| 参数           | 类型      | 说明                                                         | 是否必须 |
| -----------   | -------- | ----------------------------------------------------------   | ------ |
| client_id     | string   | 应用id                                                        | 是     |
| redirect_uri  | string   | 应用回调地址                                                   | 是      |
| response_type | string   | 必须为code                                                    | 是      |
| scope         | string   | 以空格分隔，必须包含openid，指定profile时可获取用户信息             | 是      |
| state         | string   | 可选字段，重定向时会原样返回，用于检测信息中途是否被修改               | 否      |

**请求示例：**
```
https://dev-finchat-iam.finogeeks.club/auth/realms/master/protocol/openid-connect/auth?client_id=test-scope&state=xyz&redirect_uri=https://www.baidu.com&response_type=code&scope=openid profile
```

**响应参数说明：**
| 参数           | 类型     | 说明                                     |
| ------------- | -------- | --------------------------------------- |
| code          | string   | 一次性code，只能使用1次，并且必须在1分钟内使用 | 
| state         | string   | 请求的参数，原样返回                        | 
| session_state | string   | session状态字段，无需使用                   | 

**响应示例：**
```
https://www.baidu.com/?state=xyz&session_state=b4d27b8d-63cd-4187-b839-e165411d273c&code=0da8c2c4-5d39-4379-8246-3f7b4ab8b931.b4d27b8d-63cd-4187-b839-e165411d273c.test-scope
```

### 获取access_token
因为要使用到secret，而secret应该存储在后端，此接口一般是后端调用

**请求方式：POST**

**请求地址：{IAM_HOST}/auth/realms/master/protocol/openid-connect/token**

**请求头部：**

| key           | value                                       |
| ------------- | ------------------------------------------- |
| Authorization | Basic base64.encode(client_id:secret) |
| Content-Type  | application/x-www-form-urlencoded           |
需要对client_id和secret进行base64编码


**请求参数说明：**
| 参数           | 类型      | 说明                                                         | 是否必须 |
| -----------   | -------- | ----------------------------------------------------------   | ------- |
| client_id     | string   | 应用id                                                        | 是      |
| redirect_uri  | string   | 应用回调地址                                                   | 是       |
| code          | string   | 一次性code                                                    | 是       |
| grant_type    | string   | 必须为authorization_code                                      | 是       |

**请求示例：**
```
POST /auth/realms/master/protocol/openid-connect/token HTTP/1.1
Host: https://finchat-dev.finogeeks.club
Authorization: Basic dGVzdC1zY29wZTo0MzViMjFhNC00ZTJiLTQ5MDctOTA1Ny0xZDg2ZmJhNDI4Y2Q=
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=0da8c2c4-5d39-4379-8246-3f7b4ab8b931.b4d27b8d-63cd-4187-b839-e165411d273c.test-scope&redirect_uri=https://www.baidu.com&client_id=test-scope
```

**响应参数说明：**
| 参数                      | 类型     | 说明                                               |
| ------------------------ | -------- | ------------------------------------------------- |
| access_token             | string   | access_token，用于调用userinfo接口                  | 
| expires_in               | int      | access_token过期时间，单位为秒                       | 
| refresh_token            | string   | 用于在access_token过期时获取access_token             | 
| refresh_rexpires_in      | int      | refresh_token过期时间，单位为秒                               | 
| token_type               | string   | token类型，为bearer                                 | 
| id_token                 | string   | 可直接解析获取部分用户信息，也可以调用userinfo获取更多信息 | 
| not_before_policy        | int      | 另外设置的过期时间字段，无需使用                         | 
| session_state            | string   | session状态字段，无需使用                             | 
| scope                    | string   | 应用拥有的权限，无需使用                               | 


**响应示例：**
```
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJxV0FLZGhsbm9KaDBtc2U4aG1FYVZ4OEpHcXhOc3ZMYnpIQnpUMTJpc1QwIn0.eyJqdGkiOiI4N2FiMTZlZS0yY2UxLTQyMWYtYjE5Zi05NGYxMGYyYTVlZTUiLCJleHAiOjE1OTY2MTMxNjQsIm5iZiI6MCwiaWF0IjoxNTk2NjEyODY0LCJpc3MiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJzdWIiOiIwMDQzMTZkNC02ZDllLTRjNmUtOWY1ZC0xZmZjYWI0ODYwMTIiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJ0ZXN0LXNjb3BlIiwiYXV0aF90aW1lIjoxNTk2NjEyNjI5LCJzZXNzaW9uX3N0YXRlIjoiYjRkMjdiOGQtNjNjZC00MTg3LWI4MzktZTE2NTQxMWQyNzNjIiwiYWNyIjoiMCIsInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgb2ZmbGluZV9hY2Nlc3MiLCJvcmdhbml6YXRpb25faWQiOiIwMDAxMCIsImZjaWQiOiJAbGluaWFuMjAzOmRldi5maW5vZ2Vla3MuY2x1YiIsIm5hbWUiOiLmnY7lv7UyMDMifQ.XXLa-tWkZADeZir3-9I8EZSjygSkbdpOOmouNjjiC6xXLgIOMXlvWHZJW4eS4abKoILqQc9fbShTmTEPFPfidqPo1MoI2ANMFUm4fbcFnNTdNPxr-kNly7E1plrZsiJoksxQjQ7CMVuQf-dA1OuWw1NWrd3uBKWgrFeTJ4hFoOggj_DkIJYYIJdio3MhfyFfD6Gm0XwLCb1W5qHYOk1YScgnyXl_Oq1XNzdvtaJ6DsMeK_saqA7BkyerQVJ2uBhn9Xw7FE7xlDcxVNDCzjy-lAP_HCwlNm-sgUhC97L9FhB1HCx81caNiLzzBYwq1Jt9QWNtkwZ4WkFFtOKZMt-JSw",
    "expires_in": 300,
    "refresh_expires_in": 52342343,
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICIxZjRhODA0ZS1iNDg1LTQzNDQtOGY4Zi1hZWJiZmVjNDdjMTgifQ.eyJqdGkiOiJlNjZjY2YzOS1mMzVhLTQxYzItOGNhYy1mOWNiYTI3MWNjN2IiLCJleHAiOjAsIm5iZiI6MCwiaWF0IjoxNTk2NjEyODY0LCJpc3MiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJhdWQiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJzdWIiOiIwMDQzMTZkNC02ZDllLTRjNmUtOWY1ZC0xZmZjYWI0ODYwMTIiLCJ0eXAiOiJPZmZsaW5lIiwiYXpwIjoidGVzdC1zY29wZSIsImF1dGhfdGltZSI6MCwic2Vzc2lvbl9zdGF0ZSI6ImI0ZDI3YjhkLTYzY2QtNDE4Ny1iODM5LWUxNjU0MTFkMjczYyIsInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgb2ZmbGluZV9hY2Nlc3MifQ.fkhIJVLr21tHhXuwwJOotgrqo7X2lrlWPSG70NWowtM",
    "token_type": "bearer",
    "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJxV0FLZGhsbm9KaDBtc2U4aG1FYVZ4OEpHcXhOc3ZMYnpIQnpUMTJpc1QwIn0.eyJqdGkiOiJjNWVlYWY2Mi03YjA5LTQ0OTgtOTU4Yi02ZGRiMzc1N2I0NDEiLCJleHAiOjE1OTY2MTMxNjQsIm5iZiI6MCwiaWF0IjoxNTk2NjEyODY0LCJpc3MiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJhdWQiOiJ0ZXN0LXNjb3BlIiwic3ViIjoiMDA0MzE2ZDQtNmQ5ZS00YzZlLTlmNWQtMWZmY2FiNDg2MDEyIiwidHlwIjoiSUQiLCJhenAiOiJ0ZXN0LXNjb3BlIiwiYXV0aF90aW1lIjoxNTk2NjEyNjI5LCJzZXNzaW9uX3N0YXRlIjoiYjRkMjdiOGQtNjNjZC00MTg3LWI4MzktZTE2NTQxMWQyNzNjIiwiYWNyIjoiMCIsIm9yZ2FuaXphdGlvbl9pZCI6IjAwMDEwIiwiZmNpZCI6IkBsaW5pYW4yMDM6ZGV2LmZpbm9nZWVrcy5jbHViIiwibmFtZSI6IuadjuW_tTIwMyJ9.SJPXQWtGc2Z4lJCQr4b5TTRVg6wULnvuWtJrORGnLJjK6SiYYWbJ2lSc83wdp_lFV5pyjRxiaPCAU7CLTIEUISbXuV528jV5Meud8LAIBe1S83YP0yrMTuYmSErBuZ_mtCT3dRuWeGgDDHSJf7rdnJIx80v4J60AuNRu4ze_ki3aSK5v-tta489EpoHEG-LYv8I3-pOwVLzwISJxmP-OFRCnMLzxlDUuyw2nApvIXX_yCgIt58GEggGJV5Lu-zFYxKKLtbuqBc53avaJK-fLtxB_ZI3Xwt_PuOgm5BLgurYhbXNlvWsXOkG5w3nAIvsdMquo0G5eB5mzvs0_ENzCPw",
    "not-before-policy": 1596531955,
    "session_state": "b4d27b8d-63cd-4187-b839-e165411d273c",
    "scope": "openid profile"
}
```

### 获取用户信息

**请求方式：GET**

**请求地址：{IAM_HOST}/auth/realms/master/protocol/openid-connect/userinfo**

**请求头部：**

| key           | value                                       |
| ------------- | ------------------------------------------- |
| Authorization | Bearer ${access_token}                      |

**请求示例：**
```
GET /auth/realms/master/protocol/openid-connect/userinfo HTTP/1.1
Host: https://finchat-dev.finogeeks.club
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJxV0FLZGhsbm9KaDBtc2U4aG1FYVZ4OEpHcXhOc3ZMYnpIQnpUMTJpc1QwIn0.eyJqdGkiOiI4N2FiMTZlZS0yY2UxLTQyMWYtYjE5Zi05NGYxMGYyYTVlZTUiLCJleHAiOjE1OTY2MTMxNjQsIm5iZiI6MCwiaWF0IjoxNTk2NjEyODY0LCJpc3MiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJzdWIiOiIwMDQzMTZkNC02ZDllLTRjNmUtOWY1ZC0xZmZjYWI0ODYwMTIiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJ0ZXN0LXNjb3BlIiwiYXV0aF90aW1lIjoxNTk2NjEyNjI5LCJzZXNzaW9uX3N0YXRlIjoiYjRkMjdiOGQtNjNjZC00MTg3LWI4MzktZTE2NTQxMWQyNzNjIiwiYWNyIjoiMCIsInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgb2ZmbGluZV9hY2Nlc3MiLCJvcmdhbml6YXRpb25faWQiOiIwMDAxMCIsImZjaWQiOiJAbGluaWFuMjAzOmRldi5maW5vZ2Vla3MuY2x1YiIsIm5hbWUiOiLmnY7lv7UyMDMifQ.XXLa-tWkZADeZir3-9I8EZSjygSkbdpOOmouNjjiC6xXLgIOMXlvWHZJW4eS4abKoILqQc9fbShTmTEPFPfidqPo1MoI2ANMFUm4fbcFnNTdNPxr-kNly7E1plrZsiJoksxQjQ7CMVuQf-dA1OuWw1NWrd3uBKWgrFeTJ4hFoOggj_DkIJYYIJdio3MhfyFfD6Gm0XwLCb1W5qHYOk1YScgnyXl_Oq1XNzdvtaJ6DsMeK_saqA7BkyerQVJ2uBhn9Xw7FE7xlDcxVNDCzjy-lAP_HCwlNm-sgUhC97L9FhB1HCx81caNiLzzBYwq1Jt9QWNtkwZ4WkFFtOKZMt-JSw
```

**响应参数说明：**
| 参数             | 类型     | 说明                                               |
| --------------- | -------- | ------------------------------------------------- |
| sub             | string   | 用户在IAM中的id，无需使用                            | 
| name            | string   | 用户姓名                                           | 
| fcid            | string   | 用户在FinChat中的id                                 | 
| organization_id | string   | 用户所在机构id                                      | 
| phone_number    | string   | 手机号                                             | 
| email           | string   | 邮箱                                               | 


**响应示例：**
```
{
    "sub": "004316d4-6d9e-4c6e-9f5d-1ffcab486012",
    "organization_id": "00010",
    "fcid": "@test203:dev.finogeeks.club",
    "name": "测试203",
    "phone_number": "13345678901",
    "email": "33@qq.com"
}
```


### 刷新access_token

**请求方式：POST**

**请求地址：{IAM_HOST}/auth/realms/master/protocol/openid-connect/token**

**请求头部：**

| key           | value                                       |
| ------------- | ------------------------------------------- |
| Authorization | Basic base64.encode(client_id:secret)       |
| Content-Type  | application/x-www-form-urlencoded           |
需要对client_id和secret进行base64编码


**请求参数说明：**
| 参数           | 类型      | 说明                                                         | 是否必须 |
| -----------   | -------- | ----------------------------------------------------------   | ------- |
| grant_type    | string   | 必须为refresh_token                                           | 是      |
| refresh_token | string   | refresh_token的值                                             | 是      |

**请求示例：**
```
POST /auth/realms/master/protocol/openid-connect/token HTTP/1.1
Host: https://finchat-dev.finogeeks.club
Authorization: Basic dGVzdC1zY29wZTo0MzViMjFhNC00ZTJiLTQ5MDctOTA1Ny0xZDg2ZmJhNDI4Y2Q=
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICIxZjRhODA0ZS1iNDg1LTQzNDQtOGY4Zi1hZWJiZmVjNDdjMTgifQ.eyJqdGkiOiJlNjZjY2YzOS1mMzVhLTQxYzItOGNhYy1mOWNiYTI3MWNjN2IiLCJleHAiOjAsIm5iZiI6MCwiaWF0IjoxNTk2NjEyODY0LCJpc3MiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJhdWQiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJzdWIiOiIwMDQzMTZkNC02ZDllLTRjNmUtOWY1ZC0xZmZjYWI0ODYwMTIiLCJ0eXAiOiJPZmZsaW5lIiwiYXpwIjoidGVzdC1zY29wZSIsImF1dGhfdGltZSI6MCwic2Vzc2lvbl9zdGF0ZSI6ImI0ZDI3YjhkLTYzY2QtNDE4Ny1iODM5LWUxNjU0MTFkMjczYyIsInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgb2ZmbGluZV9hY2Nlc3MifQ.fkhIJVLr21tHhXuwwJOotgrqo7X2lrlWPSG70NWowtM
```

**响应参数说明：**
| 参数                      | 类型     | 说明                                               |
| ------------------------ | -------- | ------------------------------------------------- |
| access_token             | string   | access_token，用于调用userinfo接口                  | 
| expires_in               | int      | access_token过期时间，单位为秒                       | 
| refresh_token            | string   | 用于在access_token过期时获取access_token             | 
| refresh_rexpires_in      | int      | refresh_token过期时间，单位为秒                      | 
| token_type               | string   | token类型，为bearer                                 | 
| id_token                 | string   | 可直接解析获取部分用户信息，也可以调用userinfo获取更多信息 | 
| not_before_policy        | int      | 另外设置的过期时间字段，无需使用                        | 
| session_state            | string   | session状态字段，无需使用                             | 
| scope                    | string   | 应用拥有的权限，无需使用                               | 


**响应示例：**
```
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJxV0FLZGhsbm9KaDBtc2U4aG1FYVZ4OEpHcXhOc3ZMYnpIQnpUMTJpc1QwIn0.eyJqdGkiOiI4N2FiMTZlZS0yY2UxLTQyMWYtYjE5Zi05NGYxMGYyYTVlZTUiLCJleHAiOjE1OTY2MTMxNjQsIm5iZiI6MCwiaWF0IjoxNTk2NjEyODY0LCJpc3MiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJzdWIiOiIwMDQzMTZkNC02ZDllLTRjNmUtOWY1ZC0xZmZjYWI0ODYwMTIiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJ0ZXN0LXNjb3BlIiwiYXV0aF90aW1lIjoxNTk2NjEyNjI5LCJzZXNzaW9uX3N0YXRlIjoiYjRkMjdiOGQtNjNjZC00MTg3LWI4MzktZTE2NTQxMWQyNzNjIiwiYWNyIjoiMCIsInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgb2ZmbGluZV9hY2Nlc3MiLCJvcmdhbml6YXRpb25faWQiOiIwMDAxMCIsImZjaWQiOiJAbGluaWFuMjAzOmRldi5maW5vZ2Vla3MuY2x1YiIsIm5hbWUiOiLmnY7lv7UyMDMifQ.XXLa-tWkZADeZir3-9I8EZSjygSkbdpOOmouNjjiC6xXLgIOMXlvWHZJW4eS4abKoILqQc9fbShTmTEPFPfidqPo1MoI2ANMFUm4fbcFnNTdNPxr-kNly7E1plrZsiJoksxQjQ7CMVuQf-dA1OuWw1NWrd3uBKWgrFeTJ4hFoOggj_DkIJYYIJdio3MhfyFfD6Gm0XwLCb1W5qHYOk1YScgnyXl_Oq1XNzdvtaJ6DsMeK_saqA7BkyerQVJ2uBhn9Xw7FE7xlDcxVNDCzjy-lAP_HCwlNm-sgUhC97L9FhB1HCx81caNiLzzBYwq1Jt9QWNtkwZ4WkFFtOKZMt-JSw",
    "expires_in": 300,
    "refresh_expires_in": 52342343,
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICIxZjRhODA0ZS1iNDg1LTQzNDQtOGY4Zi1hZWJiZmVjNDdjMTgifQ.eyJqdGkiOiJlNjZjY2YzOS1mMzVhLTQxYzItOGNhYy1mOWNiYTI3MWNjN2IiLCJleHAiOjAsIm5iZiI6MCwiaWF0IjoxNTk2NjEyODY0LCJpc3MiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJhdWQiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJzdWIiOiIwMDQzMTZkNC02ZDllLTRjNmUtOWY1ZC0xZmZjYWI0ODYwMTIiLCJ0eXAiOiJPZmZsaW5lIiwiYXpwIjoidGVzdC1zY29wZSIsImF1dGhfdGltZSI6MCwic2Vzc2lvbl9zdGF0ZSI6ImI0ZDI3YjhkLTYzY2QtNDE4Ny1iODM5LWUxNjU0MTFkMjczYyIsInNjb3BlIjoib3BlbmlkIHByb2ZpbGUgb2ZmbGluZV9hY2Nlc3MifQ.fkhIJVLr21tHhXuwwJOotgrqo7X2lrlWPSG70NWowtM",
    "token_type": "bearer",
    "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJxV0FLZGhsbm9KaDBtc2U4aG1FYVZ4OEpHcXhOc3ZMYnpIQnpUMTJpc1QwIn0.eyJqdGkiOiJjNWVlYWY2Mi03YjA5LTQ0OTgtOTU4Yi02ZGRiMzc1N2I0NDEiLCJleHAiOjE1OTY2MTMxNjQsIm5iZiI6MCwiaWF0IjoxNTk2NjEyODY0LCJpc3MiOiJodHRwczovL2Rldi1maW5jaGF0LWlhbS5maW5vZ2Vla3MuY2x1Yi9hdXRoL3JlYWxtcy9tYXN0ZXIiLCJhdWQiOiJ0ZXN0LXNjb3BlIiwic3ViIjoiMDA0MzE2ZDQtNmQ5ZS00YzZlLTlmNWQtMWZmY2FiNDg2MDEyIiwidHlwIjoiSUQiLCJhenAiOiJ0ZXN0LXNjb3BlIiwiYXV0aF90aW1lIjoxNTk2NjEyNjI5LCJzZXNzaW9uX3N0YXRlIjoiYjRkMjdiOGQtNjNjZC00MTg3LWI4MzktZTE2NTQxMWQyNzNjIiwiYWNyIjoiMCIsIm9yZ2FuaXphdGlvbl9pZCI6IjAwMDEwIiwiZmNpZCI6IkBsaW5pYW4yMDM6ZGV2LmZpbm9nZWVrcy5jbHViIiwibmFtZSI6IuadjuW_tTIwMyJ9.SJPXQWtGc2Z4lJCQr4b5TTRVg6wULnvuWtJrORGnLJjK6SiYYWbJ2lSc83wdp_lFV5pyjRxiaPCAU7CLTIEUISbXuV528jV5Meud8LAIBe1S83YP0yrMTuYmSErBuZ_mtCT3dRuWeGgDDHSJf7rdnJIx80v4J60AuNRu4ze_ki3aSK5v-tta489EpoHEG-LYv8I3-pOwVLzwISJxmP-OFRCnMLzxlDUuyw2nApvIXX_yCgIt58GEggGJV5Lu-zFYxKKLtbuqBc53avaJK-fLtxB_ZI3Xwt_PuOgm5BLgurYhbXNlvWsXOkG5w3nAIvsdMquo0G5eB5mzvs0_ENzCPw",
    "not-before-policy": 1596531955,
    "session_state": "b4d27b8d-63cd-4187-b839-e165411d273c",
    "scope": "openid profile"
}
```