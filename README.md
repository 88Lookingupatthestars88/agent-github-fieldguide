# agent-github-fieldguide

Two GitHub skills for agents. One gets a credential working. One uses it.

## Install

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

Replace `~/.claude/skills/` with your client's skills directory. If the client has no skills support, paste SKILL.md into the system prompt.

## The MCP endpoint answers four ways

`https://api.githubcopilot.com/mcp/` returns four different responses depending on what arrived with the request. Three of them look like a token problem.

### 401 bad request: missing required Authorization header

No Authorization header reached the server.

Most often the client is trying to do OAuth. GitHub does not implement dynamic client registration, so the client cannot register itself, gets no token, and sends the request without a header. A new token will not help here, since no token took part in the request. Put a PAT in the header instead:

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

The token arrived and the server rejected it. It is expired, revoked, or of a type this endpoint does not accept.

Check the expiry date and permissions, then issue a new token.

### 400 bad request: Authorization header is badly formatted

The header is present but its value is malformed: `Bearer` with an empty value, or a scheme other than Bearer.

In this case the token may be valid and the header around it is not.

### 200 with mcp-session-id

The connection worked. The response carries a `mcp-session-id` header and `serverInfo` in the body. A client that shows success without those two is showing its own state, not the server's.

## What each skill covers

`github-onboard` runs until a credential can read a repository. Sign-up, sign-in, whether the client can do OAuth at all, OAuth versus a fine-grained token, and three transports: MCP over OAuth, MCP over a PAT, plain REST.

`github-operate` takes over from there. Reading files, changing files, atomic multi-file commits, branches, pull requests, issues, workflow runs, releases, and confirmation before deletions: list what would go, wait for an explicit yes, remove one item at a time.

Say 注册 GitHub, token 怎么弄 or 让我读你的仓库 and onboard picks it up. Say 提交, 开个 PR or 删仓库 and operate does.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

Not affiliated with or endorsed by GitHub, Inc. GitHub is named here only to identify what these skills operate on.
