# agent-github-fieldguide

中文 · [English](README.en.md)

两个 Agent Skill，让 agent 自己把东西弄上 GitHub。**github-onboard** 管凭证，**github-operate** 管读写、开 PR、发版。都是纯文本，放进客户端就生效。

## 下载

- [github-onboard-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.1.0/github-onboard-SKILL.md) — 17431 B
- [github-operate-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.1.0/github-operate-SKILL.md) — 32020 B
- [agent-github-fieldguide-v1.1.0.zip](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.1.0/agent-github-fieldguide-v1.1.0.zip) — 24635 B，两个 skill 连文件夹打包

## 装

解压得到两个文件夹，连文件夹放进客户端的 skills 目录：

```
你的 skills 目录/github-onboard/SKILL.md
你的 skills 目录/github-operate/SKILL.md
```

文件夹名要和文件第二行的 name 一样，不一样客户端不认。

skills 目录在哪，各家客户端不同。设置里搜 skills 或「技能」，搜不到就按客户端文档新建一个。

只下 SKILL.md 也行，自己建个同名文件夹放进去。客户端不支持 skill 的，把 SKILL.md 全文粘进系统提示，照样能用。

要克隆的话：

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

那个目录换成你客户端的实际路径。

## github-onboard

手上没有能用的凭证时用它。终点是拿这个凭证读通一个真实仓库；拿到 token 不算完。

- 没账号：去 https://github.com/signup 注册，被拒的那一栏按页面提示改。
- 有账号：去 https://github.com/login 登录。登没登上，看 https://github.com/settings/profile 能不能打开。
- 能不能走 OAuth，看客户端有没有自带注册好的 GitHub App。VS Code 1.101 起、Visual Studio、JetBrains、Eclipse、Cursor 都自带；通用 MCP 连接器没有，只能建 token。原因在 GitHub 那边：它没有动态客户端注册，客户端没法登记自己。
- 走不了 OAuth 就建 fine-grained token。勾哪个权限由任务定，有六件事任何权限都盖不住，只能换 classic token。
- 四条通道：MCP 走 OAuth、MCP 在请求头里带 token、纯 REST、要 clone 或 push 时走 git。
- 最后一步是拿凭证真读一次仓库。名下没有仓库，就读仓库列表，返回空数组也算通。

## github-operate

凭证通了、要开始改东西时用它。

- 读仓库列表、目录树、文件内容。文件超过 1 MB 要换 raw 接口；列表默认一次 30 条，要全量得翻页。
- 改单个文件前先读当前的 sha 值。文件不存在就不带这个字段，带空值返回 422。
- 一次改多个文件走 Git Data API 合成一个提交：请求带 base_tree，移动 ref 之前再读一次 head。
- 分支、PR、Issue、workflow、Release 都管。三种合并方式各留下什么，动手前会讲。Release 附件是独立对象，移动 tag 不会带着它走。
- 删除之前先给一份编号清单，每项写清会带走什么，再说明清单之外还有什么。拿到明确确认才动手，一次一项。删仓库、删分支、删文件、删本地目录各算各的。
- 仓库设置也管：description、homepage、topics、Discussions 开关。改 topics 得走 PUT，PATCH 返回 200 但一个字段都不改。

## 两个一起用

github-onboard 在第一次读通仓库时结束，github-operate 从那儿接着做，token 直接复用。

会话里第一个请求要带仓库名，后面的可以省。

OAuth 拿到的凭证只活在那一个客户端里。之后要在 shell 里调 REST 或跑 git，得另外建 token。

## 连不上时对照这四种回答

装的时候用不到，连不上再回来。GitHub 的 MCP 端点按请求带了什么决定回什么，四种里有三种看着像 token 问题，只有一种是。

### 401 bad request: missing required Authorization header

服务端没收到 Authorization 头。多半是客户端在走 OAuth：GitHub 没有动态客户端注册，客户端注册不了自己，拿不到 token，发出去的请求里就没有这个头。换 token 没用，这次请求里本来没有 token。把 PAT 写进头里：

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

token 到了，被拒了。过期、被撤销，或者这个端点不收这种类型。先看到期时间和权限，再决定要不要重签。

### 400 bad request: Authorization header is badly formatted

头在，格式不对。Bearer 后面是空的，或者 scheme 不是 Bearer。token 本身可能没问题，问题在带它的那个头。

### 200 带 mcp-session-id

通了。响应头有 mcp-session-id，body 里有 serverInfo。客户端只报成功、拿不出这两样，就是没真连上。

## 反馈

Issues 开着。哪一步和你看到的不一样，说清是第几步，把实际返回的原文贴上来。原文能看出是四种原因里的哪一种。

用法问题去 Discussions 的 Q&A，那里能标已解决答案。不保证回复。

## 许可

Apache-2.0，见 [LICENSE](LICENSE) 和 [NOTICE](NOTICE)。与 GitHub, Inc. 无隶属关系。
