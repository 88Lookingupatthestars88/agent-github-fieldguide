# agent-github-fieldguide

两个给 agent 用的 GitHub skill。装一个够用就装一个，两个都要就都装。

## 下载

三个文件，要哪个下哪个：

- [github-onboard-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-onboard-SKILL.md) — 14 KB
- [github-operate-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-operate-SKILL.md) — 18 KB
- [两个一起，zip](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/agent-github-fieldguide-v1.0.0.zip) — 19 KB

单个文件就是某个 skill 的 `SKILL.md`。放进同名文件夹再改个名：

```
<skills 目录>/github-onboard/SKILL.md
<skills 目录>/github-operate/SKILL.md
```

文件夹名必须和文件开头那行 `name:` 一致。客户端在 `onboard/` 这种文件夹里找到 `SKILL.md` 会直接拒掉。

zip 里已经是这个结构，直接解压到 skills 目录就行，不用改名。

也可以在仓库里点开文件，用右上角的下载图标，拿到的是一样的东西。

## github-onboard

给「agent 还没法进 GitHub」这种情况用。管到 agent 读到一个你的仓库为止，然后就停。

- 没账号：指到注册页，字段被拒的时候把报错读回来解释。不背密码规则，那些一直在变。
- 有账号：登录，然后去打开个人设置页。判断标准是那个页面能开，不是登录框消失。
- 判断你的客户端能不能走完 OAuth。桌面 IDE 自带注册好的 GitHub App，通用 MCP 连接器大多没有，重试也不会多出来一个。
- 走不了 OAuth 就建 fine-grained token：眼前这个任务该勾哪个权限，以及哪五件事 fine-grained token 无论怎么勾都做不到。
- 三条通道选一条：MCP 走 OAuth、MCP 在 header 里带 token、纯 REST。
- 最后一定去读一个真实仓库。在那之前不算完成。

## github-operate

给「已经能读仓库，现在想让它改点东西」这种情况用。

- 读：仓库列表、目录树、文件内容。超过 1MB 的走 raw，JSON 形式到那里就断。
- 改一个文件：先读当前 `sha`。更新时不带返回 409，编一个返回 422。
- 一次改多个文件：走 Git Data API 合成一个提交，期间分支被别人推过就直接拒绝。
- 分支、PR、Issue、workflow、Release。选合并方式之前，先告诉你这种方式会留下什么。
- 删除：列出每一样会被删的东西，说明清单之外还有什么，等一句明确的同意，然后一次删一项。删仓库、删分支、删文件、删本地目录，是四次确认，因为这四种错误的量级不一样。

## 两个一起用

`github-onboard` 在第一次成功读到仓库时收手，`github-operate` 接下一个请求。

装完第一次用，agent 会先问你要哪个仓库。之后就直接照你说的做。

只要一个就装一个，两个之间不互相依赖。

## 凭证这部分为什么值得看

GitHub 的 MCP 端点会根据请求里带了什么，给出四种不同的回答。三种看着都像 token 出了问题，其实只有一种是。

### 401 bad request: missing required Authorization header

服务端没收到 Authorization 头。

多数情况是客户端在走 OAuth。GitHub 没有实现动态客户端注册，客户端没地方注册自己，拿不到 token，于是发出去的请求里没有这个头。这种时候换新 token 也没用，因为这次请求里根本没有 token。改成把 PAT 写在头里：

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

token 到了服务端，被拒了。过期、被撤销，或者这个端点不接受这种类型。

去看到期时间和权限，然后重新签发。

### 400 bad request: Authorization header is badly formatted

头在，值是坏的。`Bearer` 后面是空的，或者 scheme 不是 Bearer。

这种 token 本身可能是好的，坏的是外面那层头。

### 200 带 mcp-session-id

通了。响应头里有 `mcp-session-id`，body 里有 `serverInfo`。只报成功、这两样拿不出来的客户端，报的是它自己的状态。

## 从源码装

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

`~/.claude/skills/` 换成你客户端实际的 skills 目录。不支持 skill 的客户端，把 SKILL.md 粘进系统提示也能用。

## 反馈

Issues 开着。哪一步和你看到的不一样，说清是第几步，把实际返回的内容贴上来。原文能看出是四种原因里的哪一种。

不保证回复。

## 许可

Apache-2.0，见 [LICENSE](LICENSE) 和 [NOTICE](NOTICE)。与 GitHub, Inc. 无关联，未获其背书。
