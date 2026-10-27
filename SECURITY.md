# 安全性

这个仓库没有依赖，也没有构建步骤。但两份 SKILL.md 里有命令块，agent 会照着敲。所以这里错一处，仓库就真的被改了。

## 什么算安全问题

两种。

**一、照着做会做错。** 命令、参数、地址、权限名、分支名、目标仓库，有一处不对，agent 就打到别的地方去了。常见的是这几种：

- token 被写进文件、命令行参数、URL、commit、issue 或 shell history
- 要的权限比这件事本身需要的宽
- 该停下来问一句的地方直接跑了：删除、改可见性、覆盖分支、改保护规则

**二、写的和实际不符。** 地址、字段名、权限名、状态码、数字跟 GitHub 返回的对不上；说这一步能成，做不成；链接已经不通。

## 报之前

真 token 不要写进任何渠道：私密报告、issue、讨论，都一样。

要说明是格式问题，给个前缀就够，后面打码。secret scanning 会扫公开内容，但等它报出来，东西已经贴出去了。

同样别贴：完整的命令回显、带账号的 URL、截图里的响应头。

## 私密报

https://github.com/88Lookingupatthestars88/agent-github-fieldguide/security/advisories/new

这个入口开着。下面三种走私密：

- 凭证可能已经漏出去
- 命令可能删掉东西
- 错到照做会造成不可逆的改动

公开报等于把做法摊到会用它的人面前。

## 不用走私密的

错别字、Markdown 渲染、链接失效、翻译别扭、章节顺序。开 issue，或者在讨论里说。

## 报的时候写什么

哪个文件、哪一节或哪一行、现在写的是什么、照着做会怎样、你实际看到什么。报错贴原文，不要截图。

---

# Security

Nothing in this repository has dependencies or a build step. But the two SKILL.md files carry command blocks, and an agent types them out. A wrong line here changes a repository.

## What counts as a security problem

Two kinds.

**One: following it does the wrong thing.** A command, a flag, an address, a permission name, a branch, a target repository — one of them wrong and the agent lands somewhere else. The usual shapes:

- a token written into a file, a command-line argument, a URL, a commit, an issue or shell history
- a permission wider than the task needs
- a step that runs where it should have stopped to ask: delete, visibility change, branch overwrite, protection rules

**Two: the text does not match the facts.** An address, a field name, a permission name, a status code or a number that disagrees with what GitHub returns; a step described as working that does not; a link that no longer resolves.

## Before you report

A real token does not go into any channel: private report, issue, discussion, all the same.

To point at a format problem, a prefix with the rest masked is enough. Secret scanning watches public content, but by the time it fires, the thing is already out.

Also leave out full command output, URLs with an account in them, and screenshots of response headers.

## Private report

https://github.com/88Lookingupatthestars88/agent-github-fieldguide/security/advisories/new

That form is open. Send it there when:

- a credential may already be exposed
- a command may delete something
- the text is wrong enough that following it does damage you cannot undo

A public report lays the method out for whoever would use it.

## What does not need one

Typos, Markdown rendering, dead links, awkward translation, section order. Open an issue, or say it in the discussions.

## What to write

The file, the section or the line, what it says now, what happens when someone follows it, and what you saw instead. Paste the error text rather than a screenshot.
