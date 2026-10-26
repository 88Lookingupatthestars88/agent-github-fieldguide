# agent-github-fieldguide

中文 · [English](README.en.md)

这两个 skill 给在 GitHub 上干活的 agent 用。`github-onboard` 负责注册、token 和第一次读通仓库，`github-operate` 负责写文件、开分支、发 PR、发版。两个互不依赖。

一个 skill 就是一个文件夹加一个 `SKILL.md`，客户端按文件夹名加载。

## 下载

- [github-onboard-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-onboard-SKILL.md) — 17468 B
- [github-operate-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-operate-SKILL.md) — 24212 B
- [agent-github-fieldguide-v1.0.0.zip](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/agent-github-fieldguide-v1.0.0.zip) — 21932 B，两个 skill 连文件夹一起打包

每个文件都是一个 skill 的 SKILL.md，放进和它同名的文件夹里：

```
你的 skills 目录/github-onboard/SKILL.md
你的 skills 目录/github-operate/SKILL.md
```

文件夹名必须和文件开头那行的 name 一致，否则客户端直接拒收。zip 解开就是这两层目录，复制进去即可。

skills 目录在哪，各家客户端不一样。设置里搜 skills 或「技能」，找不到就翻客户端文档。

## github-onboard

agent 手上还没有可用凭证时用这个 skill，它会一直做到凭证真正读通一个仓库为止。

- 没有账号：先去 https://github.com/signup 注册，哪一栏被拒就按页面上的提示改。
- 已经有账号：去 https://github.com/login 登录。登录成不成，看 https://github.com/settings/profile 能不能打开。
- 客户端能不能走完 OAuth，取决于它有没有自带注册好的 GitHub App。VS Code 1.101 及以后、Visual Studio、JetBrains、Eclipse、Cursor 都自带；通用的 MCP 连接器没有，也就走不了 OAuth，只能改用 token。根因在 GitHub 这边：它没有动态客户端注册，客户端无处登记自己。
- 走不了 OAuth 就建 fine-grained token。这个 skill 会说明这个任务该勾哪个权限，以及哪六件事没有任何权限能覆盖、只能换 token 类型。
- 它支持四条通道：MCP 走 OAuth、MCP 在 header 里带 token、纯 REST、需要 clone 和 push 时走 git。
- 最后一步是拿凭证真读一次。有仓库就读那个仓库；一个仓库都没有，就读仓库列表，返回空数组同样算通过。

## github-operate

agent 已经能读仓库、需要改东西时用这个 skill。

- 它能读仓库列表、目录树和文件内容。内容超过 1 MB 要改用 raw；列表默认只返回 30 条，要拿全得加 `--paginate`。
- 改一个文件时，它先读当前的 `sha`。文件不存在就不带这个字段，带空值会返回 422。
- 一次改多个文件时，它走 Git Data API 合成一个提交；请求里带 `base_tree`，移动 ref 之前再读一次 head。
- 分支、PR、Issue、workflow、Release 也都管。三种合并方式各会留下什么，动手前会讲清楚。Release 附件是独立对象，移动 tag 不会带着它走。
- 删除之前，它先给一份编号清单，每项写清会被带走什么，再说明清单之外还有什么。拿到明确确认才动手，一次删一项。删仓库、删分支、删文件、删本地目录各自单独确认。

## 两个一起用

`github-onboard` 在第一次读通仓库时结束，`github-operate` 从那里接着做，可以复用同一个 token。

会话里第一个请求要带仓库名，后面的可以省略。

OAuth 拿到的凭证只在那一个客户端里有效。之后要在 shell 里调 REST 或跑 git，还得另外建一个 token。

## 这个端点会给出四种回答

这一节装的时候用不到，连不上时回来对照。GitHub 的 MCP 端点按请求里带了什么决定回什么，四种里有三种看着像 token 的问题，只有 `token authentication failed` 这一种真要换 token。

### 401 bad request: missing required Authorization header

服务端没收到 Authorization 头。

多半是客户端在走 OAuth。GitHub 没有实现动态客户端注册，客户端注册不了自己，拿不到 token，发出去的请求里就没有这个头。换 token 不影响结果，这次请求里本来没有 token。把 PAT 写进头里：

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

token 到了，被拒了。过期、被撤销，或者这个端点不接受这种类型。

先看到期时间和权限，再决定要不要重签。

### 400 bad request: Authorization header is badly formatted

头在，格式不对：`Bearer` 后面是空的，或者 scheme 不是 `Bearer`。token 可能没问题，问题在带它的那个头。

### 200 带 mcp-session-id

通了。响应头有 `mcp-session-id`，body 有 `serverInfo`。客户端只报成功、却拿不出这两样，就是没真连上。

## 从源码装

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

`~/.claude/skills/` 换成你客户端实际的 skills 目录。没有 git 就用上面的下载链接。客户端不支持 skill 的，把 SKILL.md 粘进系统提示也能用。

## 反馈

Issues 开着。哪一步和你看到的不一样，说清是第几步，把实际返回的内容贴上来，原文能看出是四种原因里的哪一种。不保证回复。

## 许可

Apache-2.0，见 [LICENSE](LICENSE) 和 [NOTICE](NOTICE)。与 GitHub, Inc. 无隶属关系。
