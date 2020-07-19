- 仓库：blessing-skin-plugins
- 日期: 2020-07-17
- 作者：[Steven Qiu](https://github.com/tnqzh123)
- RFC PR：[#0002](https://github.com/bs-community/rfcs/pull/2)
- 实现 PR：暂无

# 材质详情中的材质描述

## 概要

增加「材质描述」功能，允许用户在上传材质时填写材质的描述及其它信息，并在材质详情页中展示。

## 初衷

对于材质的发布来说，材质描述可以是很重要的一块。材质发布者可能会提供光影渲染图、材质制作的幕后故事和相关资料等信息，而且部分材质（比如 [这个](https://www.mcbbs.net/thread-1073023-1-1.html)）要求在转载时明确说明版权归属和授权方式等信息，但当前 BS 并没有位置可以填写这些信息。

以上的问题可以通过增加「材质描述」功能解决。

## 详细设计

可以直接整合进 BS 核心，也可以使用插件实现。我倾向于使用插件（因为这样可以更灵活），故该部分仅阐述使用插件实现的方案。关于整合进 BS 核心的方案，请看「备选方案」部分。

### 数据库

新增一个 textures_desc 表用于存储相关信息。表中有三个字段 `id`、 `tid` 和 `desc`，数据类型分别为 `unsigned int`、 `unsigned int` 和 `varchar`，其中 `id` 字段作为主键，`tid` 和 `desc` 字段用于记录材质的 TID 和材质描述文本。

同时，在 options 表中增加一条 `textures_desc_limit` 记录，用于存储下方提到的「字数限制」的值。

可以仅在用户有填写材质描述时才在数据表中创建相关记录。

### 用户填写材质描述

在「上传材质」页面中，左侧「选择文件」和「内容政策」的中间增加一个文本框（`textarea`），供用户填写材质描述。

材质成功发布后，应该允许用户修改材质描述。可以在 `card-header` 标题中加一个「编辑」符号（`fa-edit`，就像管理员访问用户中心仪表盘时的「站点公告」部分那样），用户点击后 `card-body` 中原先的内容变为 `textarea` 来让用户修改；若该材质不存在描述，则在 `card-body` 中展示一个「添加描述」按钮。（可以参考 [这个](https://codepen.io/mochaa/pen/BajqMvQ)）

### 材质描述的字数限制

材质描述应存在字数限制，但这个限制应该允许管理员在插件配置中自行设定，且应该在用户填写材质描述时提示（在文本框下方增加一个 `callout-info` 或 `alert-info`）。前后端都应该对字数限制进行检查，若字数超限，则弹出 modal 或 toast 来提示用户。

### 材质描述的展示

可以在材质详情页中的材质预览的下方、评论区（如果有）的上方增加一个 `card-secondary` 来展示。当材质描述不存在时，除非当前用户为该材质的上传者或管理员，否则不应该展示材质描述。

应该支持解析 Markdown。可以使用 [thephpleague/commonmark](https://github.com/thephpleague/commonmark) 开启 GFM 扩展来解析 GitHub Flavored Markdown。

## 缺点

暂无。

## 备选方案

除了通过插件实现外，还可以考虑直接整合进 BS 核心。

整合进核心的方案与通过插件实现的方案的差异主要体现在数据库上。如果整合进核心，则可以直接在 textures 表中增加一个 `desc` 字段，数据类型为 `varchar`，用于存储材质描述文本，而不需要单独开数据表。其它部分与用插件实现的方案相同。

## FAQ

暂无。

## 未能解决的问题

暂无。

## 相关内容

暂无。

## 其它

感谢 Pig Fang（[@g-plane](https://github.com/g-plane)）和 [@mochaaP](https://github.com/mochaaP) 提出的建议。