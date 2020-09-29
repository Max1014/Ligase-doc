# URL Scheme唤起FinChat本地应用

FinChat定义了相关协议：finchat://，通过该协议，我们可以在网页中直接唤起finchat本地应用。

使用效果如同下载文件时候自动打开迅雷。

## 使用条件

需要安装finchat才能使用；如果使用绿色版免安装的客户端，需要至少启动过一次。

注：注册协议为桌面端应用常规操作，但如果安装了360之类的程序，客户端在注册此协议时可能会被拦截，这会导致协议唤起失效。

## 注意事项

如果之前没有登录，则唤起是默认先打开登录页面，且传入的参数不会生效。

在部分支持多开的特殊版本中，现在仅能通过协议唤起第一个打开的客户端。

## 参数说明

### 1.加入房间
唤起应用并直接打开相应的房间。
```
finchat://rooms/join/{roomId}
```
Example：
<finchat://rooms/join/!b72c3a01-a685-409e-ae25-d3c0735a38e4:finogeeks.club>

| name    | type   | Required | desc |
| :------ | :----- | :------- | :--------------------------------- |
| roomId | String | true     | 想要加入的房间ID号码，必须满足matrix房间id格式    |


### 2.创建房间
唤起并打开创建房间的弹窗。
```
finchat://rooms/create?type={type}&name={name}&topic={topic}&preset={preset}&isFederation={isFederation}
```
Example：
<finchat://rooms/create?type=channel&name=房间名称&topic=房间主题&preset=public_chat&isFederation=inner>

| name    | type   | Required | desc |
| :------ | :----- | :------- | :--------------------------------- |
| type | String | true     | 创建的房间类型 ，目前有两种种 normal 和 channel    |
| name | String | false     | 创建channel时，如果填入name字段，则将其填入频道名称    |
| topic | String | false     | 创建channel时，如果填入topic字段，则将其填入频道简介    |
| preset | String | false     | 默认选择私密或公开，字段为 private_chat 或 public_chat    |
| isFederation | String | false     | 是否支持跨域，若本来就不支持跨域，传了不生效， 值为 inner 或者 external    |

### 3.直接对用户发起聊天

已经添加好友存在直聊房间的，到达直聊房间
未添加好友的，自动添加好友并到达直聊房间
对不存在的id，则提示改用户不存在
```
finchat://chat?userId={fc_id}
```
Example：
<finchat://chat?userId=@arluber:finogeeks.club>

| name    | type   | Required | desc |
| :------ | :----- | :------- | :--------------------------------- |
| userId | String | true     | 对方的userid，必须满足matrix用户id格式    |

