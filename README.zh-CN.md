# agent-github-fieldguide

两个 skill，放进 agent 的 skills 目录就能用。

**[下载 v1.0.0](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/latest)** —— 解压得到 `github-onboard/` 和 `github-operate/`，各自复制进你客户端的 skills 目录。

## github-onboard：让 agent 进到你的仓库里

管到「agent 真的能读一个仓库」为止。

- 没账号：在浏览器里注册。skill 只指路和读报错，不背密码规则，那些东西一直在变。
- 有账号：登录，然后靠打开个人设置页确认真的进去了。
- 判断你的客户端到底能不能走 OAuth。第一方 IDE 自带注册好的 GitHub App，通用 MCP 连接器没有，重试多少次也没用。
- 走不了 OAuth 就建 fine-grained token：哪个任务勾哪个权限，以及哪五种事 fine-grained token 永远做不到。
- 三条接入通道任选：MCP 走 OAuth、MCP 在 header 里带 token、纯 REST。
- 最后一定去读一个真实仓库。没被用过的 token 证明不了任何事。

## github-operate：让 agent 干活不闯祸

管凭证通了之后的所有动作。

- 读：列仓库、走目录树、取文件。大文件走 raw，JSON 形式到 1MB 就断。
- 改一个文件：先读当前 `sha`。不带它更新返回 409，瞎编一个返回 422。
- 一次改多个文件：走 Git Data API 合成一个原子提交，期间有人推过就直接拒绝。
- 分支、PR、Issue、workflow、Release。包括三种合并方式各自会留下什么——这三种事后都改不回来。
- 删除：先列出要删的东西，再说明清单之外还有什么，等一句明确的同意，然后一次只删一项。删仓库、删分支、删文件、删本地目录，得分四次确认，因为造成的损失不是一个量级。

## 两个 skill 的分界

`github-onboard` 在第一次读仓库成功时交棒。`github-operate` 不讲凭证，`github-onboard` 不讲操作。只要一个就装一个。

## 凭证这部分为什么值得看

GitHub 的 MCP 端点会根据请求里带了什么，返回四种不同的回答。三种看着都像 token 出了问题，其实只有一种是。

### 401 bad request: missing required Authorization header

服务端没收到 Authorization 头。

多数情况是客户端在走 OAuth。GitHub 没有实现动态客户端注册，客户端没地方注册自己，拿不到 token，于是发出去的请求里没有这个头。这种时候重新签发 token 没有用，因为这次请求里根本没有 token。改成把 PAT 写在头里：

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

token 到了服务端，被拒了。过期、被撤销，或者这个端点不接受这种类型。

去看到期时间和权限，然后重新签发。

### 400 bad request: Authorization header is badly formatted

头在，值是坏的：`Bearer` 后面是空的，或者 scheme 不是 Bearer。

这种情况 token 本身可能是好的，坏的是外面那层头。

### 200 带 mcp-session-id

通了。响应头里有 `mcp-session-id`，body 里有 `serverInfo`。客户端只显示连上了、但拿不出这两样，那是客户端自己的状态，不是服务端的。

## 装

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

`~/.claude/skills/` 换成你客户端实际的 skills 目录。客户端不支持 skill 的话，把 SKILL.md 粘进系统提示也能用。

## 许可

Apache-2.0，见 [LICENSE](LICENSE) 和 [NOTICE](NOTICE)。与 GitHub, Inc. 无关联，未获其背书。
