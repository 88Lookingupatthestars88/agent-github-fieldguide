# agent-github-fieldguide

中文 · [English](README.en.md)

两个 skill，给在 GitHub 上干活的 agent 用。`github-onboard` 管注册、token、第一次读通仓库；`github-operate` 管写入、分支、PR、发版。两个互不依赖。

## 下载

- [github-onboard-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-onboard-SKILL.md) — 14788 B
- [github-operate-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-operate-SKILL.md) — 20246 B
- [agent-github-fieldguide-v1.0.0.zip](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/agent-github-fieldguide-v1.0.0.zip) — 19974 B，两个 skill 连文件夹一起打包

单个文件就是一个 skill 的 `SKILL.md`。存成 `<skills 目录>/github-onboard/SKILL.md` 或 `<skills 目录>/github-operate/SKILL.md`。文件夹名必须和文件开头那行 `name:` 一致，不一致客户端直接拒收。zip 解压出来就是这个结构，不用改名。

仓库里放的是同一批文件，点开任意一个用右上角的下载图标，拿到的字节一致。

## github-onboard

用在 agent 手上还没有 GitHub 认的凭证的时候，到真读通一次为止。

- 没账号：指到 https://github.com/signup，字段被拒的时候照页面上的报错往下走。不列密码规则，那些一直在变。
- 有账号：https://github.com/login。登录成功与否看 https://github.com/settings/profile 能不能打开，登录框消失不算数。
- 客户端能不能走完 OAuth，看它有没有自带注册好的 GitHub App。桌面 IDE 都有，通用 MCP 连接器没有，重试也不会多出一个。
- 走不了 OAuth 就建 fine-grained token：这个任务对应哪个权限，以及哪五件事无论勾什么权限都做不到。
- 四条通道：MCP + OAuth、MCP + header 里带 token、REST、要 clone 和 push 时走 git。
- 最后一步拿凭证真读一次：有仓库就读那个仓库，账号下一个仓库都没有就读仓库列表，空数组也算通过。

## github-operate

用在 agent 已经能读仓库、现在要改东西的时候。

- 读：仓库列表、目录树、文件内容。内容超过 1 MB 要走 raw，JSON 形式在 1 MB 处截断。
- 改一个文件：先读当前 `sha`。更新不带 `sha` 返回 409，新建时传空 `sha` 返回 422。
- 一次改多个文件：走 Git Data API 合成一个提交，带 `base_tree`，ref 移动前重读一次 head。
- 分支、PR、Issue、workflow、Release。三种合并方式各自会留下什么，选之前说清。
- 删除：先给编号清单，每项写清会被删掉什么，再说明清单之外还有什么，拿到明确的同意再动，一次删一项。删仓库、删分支、删文件、删本地目录，各自单独确认。

## 两个一起用

`github-onboard` 在第一次读通仓库时结束，`github-operate` 从那儿开始，可以用同一个 token。两个都不需要另一个先装上。

会话里第一个请求要带仓库名，后面的可以省。

有一点要注意：走 OAuth 拿到的凭证只在那一个客户端里可用。之后要在 shell 里调 REST 或跑 git，还是得单独建一个 token。

## 这个端点会给出四种回答

GitHub 的 MCP 端点看请求里带了什么，回什么。四种里有三种看着像 token 的问题，只有 token 本身被拒那一种真的是。

### 401 bad request: missing required Authorization header

服务端没收到 Authorization 头。

多数情况是客户端在走 OAuth。GitHub 没有实现动态客户端注册，客户端注册不了自己，拿不到 token，于是发出去的请求里没有这个头。换新 token 不会改变这一点，这次请求里本来就没有 token。改成把 PAT 写在头里：

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

token 到了，服务端把它拒了。过期、被撤销，或者这个端点不接受这种类型。

看到期时间和权限，然后重新签发。

### 400 bad request: Authorization header is badly formatted

头在，值是坏的。`Bearer` 后面是空的，或者 scheme 不是 Bearer。

token 本身可能是好的，坏的是带它的那个头。

### 200 带 mcp-session-id

通了。响应头里有 `mcp-session-id`，body 里有 `serverInfo`。客户端只报成功、这两样拿不出来，那是客户端自己的判断，不是端点的回答。

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

Apache-2.0，见 [LICENSE](LICENSE) 和 [NOTICE](NOTICE)。不是 GitHub 官方项目，与 GitHub, Inc. 无隶属关系。
