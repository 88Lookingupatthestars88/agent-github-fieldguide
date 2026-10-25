# agent-github-fieldguide

给 agent 用的两个 GitHub skill。一个负责把凭证弄通，一个负责用凭证干活。

## 装

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

`~/.claude/skills/` 换成你客户端实际的 skills 目录。客户端不支持 skill 的话，把 SKILL.md 粘进系统提示也能用。

## 同一个端点，四种回答

连 `https://api.githubcopilot.com/mcp/` 失败时，返回的内容取决于请求里带了什么。四种里有三种看着都像 token 出了问题。

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

## 两个 skill 各管什么

`github-onboard` 管到「凭证能读仓库」为止。注册、登录、判断客户端能不能走 OAuth、OAuth 还是 fine-grained token、三条通道（MCP + OAuth、MCP + PAT、纯 REST）。

`github-operate` 从那之后接手。读文件、改文件、一次改多个文件的原子提交、分支、PR、Issue、workflow、Release，以及删除之前的确认：先列要删的东西，等明确的同意，一次只删一项。

说「注册 GitHub」「token 怎么弄」「让我读你的仓库」，onboard 接手。说「提交」「开个 PR」「删仓库」，operate 接手。

## 许可

Apache-2.0，见 [LICENSE](LICENSE) 和 [NOTICE](NOTICE)。

与 GitHub, Inc. 无关联，未获其背书。文中的 GitHub 仅指操作对象。
