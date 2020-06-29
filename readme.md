# Blessing Skin RFCs

对于大部分的改动，例如 bug 修复，可以直接向源代码仓库 push 或发起 Pull Request，
但对于较大的改动或需要进行讨论的设计应该/可以通过 RFC 流程进行。

此 RFC 仓库不限于 Blessing Skin Server 本身，
即 Blessing Skin organization 内的任意仓库都可以使用，
但 Blessing Skin Server 以外的仓库在发起 RFC 时务必在草案文本顶部注明仓库。

## 什么时候需要进行 RFC 流程

当需要对 Blessing Skin 较大的改动时，如：

- 增加一个比较显著的新特性
- 进行影响较大的改动
- 破坏性变动

实际中可以灵活判断。另外有些虽然涉及基础部分但影响不大的时候，不必进行 RFC 流程。

## 如何提交 RFC

1. fork 这个仓库（organization 内的成员开新分支即可）；
2. 将 `0000-template.md` 文件复制到 `rfcs` 目录内，并进行适当改名：以递增的方式根据最后一个 RFC 修改新 RFC 的 ID；
并将 `template` 改为合适的词句（力求简短）；
3. 填写 RFC 模板，RFC 中部分小节可能不需要填写或没有内容可以写，填「无。」即可，但不要删掉；
4. 提交 Pull Request，任何人都可以在 Pull Request 中讨论；
5. 被 accepted 的 RFC 其 PR 将会被合并。

## 实现 RFC

任何人都可以实现 RFC，而并非一定是 RFC 的作者。

## 感谢

我们参考了 Rust 的 RFC 和 ESLint 的 RFC。
