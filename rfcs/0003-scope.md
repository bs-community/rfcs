- 仓库：blessing-skin-server
- 日期：2021-04-15
- 作者：[Asnxthaony](https://github.com/Asnxthaony)
- RFC PR: [#0003](https://github.com/bs-community/rfcs/pull/7)
- 实现 PR: [blessing-skin-server#287](https://github.com/bs-community/blessing-skin-server/pull/287)

# 令牌授权范围

## 概要

允许用户控制第三方应用访问指定信息。

## 初衷

目前，OAuth 应用申请到的 Access Token 可访问 Blessing Skin API 中的所有内容。一旦站点管理员通过 OAuth 登录第三方站点，第三方站点可通过站点管理员的 Access Token 对站点进行破坏性操作（包括但不限于：更改用户密码，更改用户权限，删除用户）。

通过限制 Access Token 的授权访问，可限制 OAuth 应用只可以访问用户的个人资料。

## 详细设计

### 授权范围(Scope)

| 作用域                      | 描述                       |
| --------------------------- | -------------------------- |
| (no scope)                  | 获取用户的基本信息         |
| User.Read                   | 获取用户的基本信息         |
| Notification.Read           | 读取通知、将通知标记为已读 |
| Notification.ReadWrite      | 发送通知                   |
| Player.Read                 | 获取用户的角色列表         |
| Player.ReadWrite            | 操作用户拥有的角色         |
| Closet.Read                 | 获取用户的衣柜             |
| Closet.ReadWrtie            | 操作用户的衣柜             |
| UsersManagement.Read        | 获取站点所有用户的信息     |
| UsersManagement.ReadWrite   | 操作站点所有的用户         |
| PlayersManagement.Read      | 获取站点所有的角色         |
| PlayersManagement.ReadWrite | 操作站点所有的角色         |
| ClosetManagement.Read       | 获取站点所有用户的衣柜     |
| ClosetManagement.ReadWrite  | 操作站点所有用户的衣柜     |
| ReportsManagement.Read      | 获取站点举报列表           |
| ReportsManagement.ReadWrite | 处理站点举报               |

## 缺点

现有的 Access Token 将无法访问 API 中的所有内容，需用户重新授权 OAuth 应用。

## 备选方案

暂无。

## FAQ

暂无。

## 未能解决的问题

暂无。

## 相关内容

暂无。

## 其它

暂无。