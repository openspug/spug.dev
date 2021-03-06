---
id: practice
title: 实践指南
---

### 安全性实践指南
写在前面，没有绝对的安全，如果环境允许的话尽量把 `Spug` 部署在内网且禁止公网访问，如果必须通过公网访问则尽可能通过防火墙等手段限制指定 IP 访问。

`Spug` 在登录安全方面提供了连续3次错误密码自动禁用账户的功能，尽可能的避免了被暴力破解登录的可能性。在 `Token` 安全方便提供了基于访问来源 IP 的
安全策略，当同一个 `Token` 访问来源 IP 发生变化时 `Token` 将自动失效，强制重新登录。
> `Token` 安全策略中的来源 IP 依赖 Nginx 转发请求时携带的 HTTP 头 `X-Real-IP` 和 `X-Forwarded-For`（v2.3.12+），
> 所以请务必确保 `X-Real-IP` 为客户端的真实 IP 而不是上级的代理 IP，如果你使用的推荐的 Docker 部署方式，则这些工作默认已经做好了。
> 如果要在 Docker 容器暴露的 80 端口上再增加反向代理层，请正确配置 `X-Forwarded-For`，例如 `Nginx` 增加 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`。
> 
> 在无法验证访问者的真实 IP 的情况下，会在登录时弹出一个警告信息，如果 Spug 部署在内网且仅在内网使用（通过内网地址访问 Spug），则可以在 v2.3.14 版本后通过
> 系统管理 / 系统设置 / 安全设置 来关闭这一特性。


### 发布过程中不同主机执行不同操作
假设一个应用需要发布到不同的三台主机上，但发布需要在主机上执行一些比如更新数据库等一次性且不能重复执行的操作，这时候可以考虑使用 `Spug` 的全局变量
`SPUG_HOST_ID` 或 `SPUG_HOST_NAME` 来指定只在某个主机上执行这些操作，例如：
```shell script
if [ "$SPUG_HOST_NAME" = "192.168.10.100" ];then
    echo "exec sql"
fi
```

### 默认PATH环境变量导致命令找不到问题
默认的 `PATH` 变量可能并不完整，这是因为 `Spug` 执行命令时并不是一个登录交互 `SHELL`，如果出现一些命令找不到等报错情况可以使用类似如下写法来解
决这个问题（注意以下写法并不会跨越不同的任务钩子，例如执行发布前执行的操作并不会影响发布执行后）：
```shell script
# 添加jdk至PATH变量
PATH=$PATH:/usr/local/jdk1.8.0_231/bin
java -jar xxx.jar
```

### 项目数据的持久存储
一些项目会在运行过程中生产需要持久存储的数据，例如用户上传的图片或需要留存的日志文件等。当使用常规发布时每个版本都是全新的目录，默认情况下这些文件会
在新版本中不可见。这种情况可以使用考虑把这些动态生成且需要持久存储的数据放在发布目录之外，通过软链接的形式链接至项目内，这样实际文件存储在一个公共的地
方每次更新时只需要额外做一个软链接的映射就可以解决问题了。

### 忽略执行中的报错
默认 `Spug` 执行任何一个命令如果返回的退出状态码不为 `0` 则终止执行（应用发布也是一样的），这是为了最大限度的保证执行的安全性，
防止未引起注意错误导致更严重的后果。你也可以在确保知道可能引起的后果的前提下，通过在命令的开头写上 `set +e` 来禁用这一特性。
但 `Spug` 仍会对最终整个任务的执行结果有个状态码的判断，如果是非 `0` 会返回异常状态或终止发布。
