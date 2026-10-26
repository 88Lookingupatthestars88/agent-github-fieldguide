---
name: github-onboard
description: Gets a user onto GitHub and leaves the agent holding a credential it has proven works. Use when the user wants to sign up for GitHub, sign in, create a personal access token or PAT, grant an agent access to a repository, or asks why GitHub returns unauthorized, bad credentials, 401, 403, 404, "could not read Username", or "Resource not accessible by personal access token". Covers MCP with OAuth, MCP with a PAT, plain REST, and git. Trigger on 注册 GitHub / 登录 GitHub / 建 token / 配置 GitHub 凭证 / token 怎么弄 / 让我能读你的仓库 / GitHub 连不上 / GitHub token setup / GitHub PAT / MCP unauthorized.
license: Apache-2.0
metadata:
  author: 88Lookingupatthestars88
  version: "1.2"
---

# GitHub Onboarding

The job ends when a credential has actually read the user's repository. Anything after that belongs to an operation skill.

Quoted lines below are prompt templates. Ask them in whatever language the user is writing.

## Step 0 — What can you run here

Check your own tool list. Three capabilities matter: shell commands, HTTP requests, file read/write. Missing some is fine; route around them.

If the harness hides its tools, say so and try once. Do not ask the user what environment they are in; they usually cannot tell.

Never claim a capability you have not exercised in this session.

## Step 1 — Where the user stands

Ask both questions in one message:

> Do you already have a GitHub account?
> If yes, which repository do you want me to reach? Give me the owner/name or the URL.

Branches:

- No account → Step 2.
- Has account, names a repository → Step 3.
- Has account, no repository yet → Step 3, then verify against the account instead of a repo (Step 4).
- Has account, something worked before and now errors → get the exact error text first, then read Failures.
- Answers vaguely → assume an account exists. Recovering one is quicker than creating a duplicate, and the later steps will expose the truth anyway.

## Step 2 — Sign up or sign in

Both happen in a browser, done by the user. Email verification, codes and human checks are theirs to complete. If they ask you to do it, say the verification step needs them and you will carry everything else.

New account: https://github.com/signup
Existing account: https://github.com/login

Do not recite password rules, field requirements or verification options. They change, and different accounts see different screens. Point at the page; when the user is stuck, ask for the exact error line and work from that.

Worth saying once, because people hit it: the email must actually receive mail, and some providers drop GitHub silently. A rejected password means read the hint on the page.

Signed in is confirmed when https://github.com/settings/profile loads.

Then settle one thing before touching credentials:

> Do you want me to be able to read your repositories, or also write to them?

Reading needs less. Default to read when the answer is unclear, since widening later costs one edit.

## Step 3 — Connect

Route before reading further, so the simple cases stay short:

- User asks for a token, or already has one → 3.4.
- User wants a one-click authorization and the client is a first-party IDE or Cursor → 3.2.
- User wants one-click but the client has no registered app → 3.1, then 3.3.
- Task needs clone, push or history → 3.5 after the credential works.

### 3.1 Decide whether OAuth is possible at all

GitHub publishes no OAuth dynamic client registration, and the reason is checkable in three requests. The unauthenticated 401 from the endpoint carries `resource_metadata="https://api.githubcopilot.com/.well-known/oauth-protected-resource/mcp/"`. That document answers 200 and names `https://github.com/login/oauth` as its authorization server. That server's `/.well-known/oauth-authorization-server` answers 404, so there is no `registration_endpoint` and a client has nowhere to register itself. OAuth therefore completes only when the client already ships a registered GitHub OAuth app.

Two signals tell you which kind of client this is, and both are visible without the user's help:

- The client lists GitHub as a built-in MCP server, or offers a one-click GitHub install. A registered app exists; OAuth will complete.
- Adding the server makes the client ask for a client ID, client secret or callback URL. No app; OAuth cannot finish.

Neither signal visible: have the user add the server and attempt the authorization, then describe what appeared. A GitHub consent screen means the app exists. A form demanding a client ID means it does not. You cannot click through this step yourself; the user does the clicking and reports back.

VS Code 1.101 and later, Visual Studio, JetBrains, Eclipse and Cursor ship one. GitHub states the requirement in its own README: every host application has to configure a GitHub App or OAuth App of its own. Generic remote-MCP connector UIs, self-hosted clients and mobile apps have none, and for those the PAT route is the one that works.

If the user refuses a token and their client has no registered app, say plainly that those are the only two doors: either the client supports OAuth through a pre-registered app, or something has to carry a credential. Offer the shortest token path rather than leaving the refusal hanging.

What each costs. OAuth takes a click and grants the scopes the server challenges for; that metadata document lists the candidates — `repo`, `read:org`, `read:user`, `user:email`, `read:packages`, `write:packages`, `read:project`, `project`, `gist`, `notifications` — and `repo` alone reaches every repository the account can. A fine-grained PAT takes about two minutes and can be pinned to named repositories and named permissions. Pick OAuth when the user wants speed on their own machine, a PAT when they want the reach limited. Say which one you took and why, in one clause.

One consequence that matters later: an OAuth credential lives inside the MCP client. MCP tool calls can use it, but a shell command on the machine cannot. If the agent will also call the REST API or run git, that work needs its own PAT even after OAuth succeeds.

### 3.2 MCP over OAuth

The remote endpoint is `https://api.githubcopilot.com/mcp/`, verified 2026-10. Confirm it answers before wiring anything:

```sh
ENDPOINT=https://api.githubcopilot.com/mcp/
curl -s -D - -o /dev/null -X POST "$ENDPOINT" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}'
```

A 401 with a `www-authenticate` header is the healthy answer: the header names the flow and links a metadata document listing the scopes and bearer methods it accepts. Treat that document as authoritative over this file. A 404 means the path moved, and https://github.com/github/github-mcp-server carries the current one. Narrower tool surfaces live under the same path, for example `/mcp/x/repos` for repository tools only. `000` means the host is unreachable from here.

The body of that 401 is a diagnosis, not noise. Four inputs produce four distinct answers, measured 2026-10:

```
no Authorization header      401  bad request: missing required Authorization header
                                  www-authenticate: error="invalid_request",
                                    error_description="No access token was provided in this request"
empty "Bearer " value        400  bad request: Authorization header is badly formatted
                                  www-authenticate: error="invalid_token",
                                    error_description="Invalid token"
wrong or expired token       401  unauthorized: token authentication failed
                                  www-authenticate: error="invalid_token",
                                    error_description="Token is not authorized"
non-Bearer scheme            400  bad request: Authorization header is badly formatted
```

The two middle cases share a status code and separate by body text. `token authentication failed` means a token arrived and the server rejected it — expired, revoked, or a type this endpoint does not accept. `missing required Authorization header` means nothing arrived, which is a client wiring problem, not a credential problem. Do not tell the user to reissue a token on the second one; it is not the same failure.

`/_ping` is not a free liveness check; it returns 401 without the same Authorization header.

Then let the client run the flow. You confirm the result; the client performs the exchange.

### 3.3 MCP over a PAT

Same endpoint as 3.2, with the credential in a header:

```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": { "Authorization": "Bearer <token>" }
    }
  }
}
```

Local Docker variant, when the client only accepts commands: the `docker run` line sits in the README at https://github.com/github/github-mcp-server. It needs a running Docker daemon and takes a PAT, not OAuth.

Three signals together mean the connection is real: HTTP 200, an `mcp-session-id` response header, `serverInfo` in the body. A client reporting "connected" without all three is not connected.

### 3.4 REST over a PAT

The route that works when the user wants a token, when MCP is unavailable, or when the agent needs to act from a shell rather than from inside a client.

Making a token:

- https://github.com/settings/personal-access-tokens/new — fine-grained, the default
- https://github.com/settings/tokens/new — classic; `repo` covers every private repository in full

Six jobs have no fine-grained equivalent, so no permission unlocks them and the token type has to change: writing to a public repository the account is not a member of, acting as an outside collaborator or repository collaborator, reaching more than one organization with a single token, Packages, the Checks API, and a Project owned by a user account.

Four things people expect on that list are not on it. Fine-grained tokens carry `gists`, `notifications`, `profile` and `keys` permissions, so a Gist, a notification, a profile edit and an SSH key all work without switching to classic.

Grant by task, not by convenience: contents and issues for reading, pull requests only if PRs are in scope, actions and workflows for CI, administration only when repository settings must change. Administration carries deletion with it, so mention that cost once and let the user decide.

Permission names drift. Check https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens instead of trusting a list here.

Confirming the token works:

```sh
curl -sS -H "Authorization: Bearer $TOKEN" https://api.github.com/user
```

A login name in the response proves the credential. With `gh` available, `gh api user --jq .login` answers the same question.

Use `gh api user`, not `gh auth status`. The status command contacts `github.com`, so a blocked web host makes it fail while `api.github.com` works perfectly.

When a call comes back `403 Resource not accessible by personal access token`, the message covers four states: a permission that was never ticked, one left at read where the call writes, a repository missing from the token's selection, and an organization that has not approved the token. The response header separates them. Every REST response carries `x-accepted-github-permissions`, naming the permission and level the endpoint checks — `contents=write`, `administration=read`, `secrets=read`.

- Header names a permission the token holds → the permission is not the problem. `GET /repos/<owner>/<repo>` answering 404 while `GET /user` answers 200 means the repository is not in the selection. A 200 on an organization's repository with the call still refused means approval is pending, and an owner clears it at https://github.com/organizations/<org>/settings/personal-access-tokens.
- Header names a permission the token lacks, or a level higher than the one granted → add it or raise it. Settings that look complete usually mean read was ticked where write was needed.
- No permission covers the operation at all → one of the six above. Change token type.
- Header reads `allows_permissionless_access=true` → the endpoint needs no permission, so permission is not the problem however the body reads.

Reissuing with the same settings fixes neither of the first two.

An organization behind SAML needs one more authorization after the token exists, from the organization's SSO page. The tell is a 403 on organization repositories with `X-GitHub-SSO` in the headers, on a token whose scopes look correct.

### 3.5 git

Only for clone, push or history. Keep the token out of the clone URL; it lands in `.git/config` and shell history, and scrubbing that costs more than issuing a new token. Use a credential helper.

One failure that gets misread as an auth problem:

```
fatal: could not read Username for 'https://<host>'
```

Usually the machine rewrites `github.com` to a mirror that does not forward private-repo credentials. Check the git config for `insteadOf` rules before blaming the token. Public clones succeed and private clones fail under that configuration.

SSH needs a public key uploaded through the web UI, so it rarely fits unattended use.

## Step 4 — Prove it reads the repository

A valid token does not prove it can reach the target. Make one read against the repository itself, using the transport you just configured.

Over REST, a 200 is the proof:

```sh
curl -sS -H "Authorization: Bearer $TOKEN" https://api.github.com/repos/<owner>/<repo>
```

Over MCP, call one read-only tool against the same target. `get_me` proves the account on most builds; take the exact tool names from `tools/list` rather than from this file. A JSON-RPC error frame is a failure even when the connection reports healthy; there is no HTTP status to check at this layer.

A 404 is ambiguous on either route: GitHub answers 404 instead of 403 when a credential lacks access, so a missing repository and an unscoped token look identical. Recheck the token's repository selection before telling anyone the repository does not exist.

No repository yet: read `https://api.github.com/user/repos` and accept 200 with an empty array as proof the credential works, then stop.

Read `github-authentication-token-expiration` from the response headers: on a fine-grained or OAuth token it carries the expiry in UTC, and its absence on a classic token means no expiry was set. A new token can take a few seconds to propagate, so retry a 401 once before calling it dead.

## Failures

- `401 Bad credentials` — mistyped, revoked, or expired. Issue a new one.
- `403 Resource not accessible by personal access token` — one message for four causes: no permission was ticked, the permission is set to read where the call writes, the repository is not in the token's selection, or an organization has not approved the token. Read `x-accepted-github-permissions` from the response first; it names the permission and the level. The branches are in 3.4.
- `403` with `X-GitHub-SSO` — organization has not authorized this token yet.
- `404` on a repository that exists — no access. See Step 4.
- `unauthorized: token authentication failed` — a token reached the server and was refused. Expired, revoked, or a token type this endpoint does not accept. Check the expiry date first, then whether the token is fine-grained or an installation token. This is not the same failure as a missing header, and reissuing only helps when the token itself is the problem.
- `bad request: missing required Authorization header` — no credential was sent at all. The client did not attach the header, often because it is still trying to complete OAuth with no registered app behind it. Look at the outgoing request; if there is no `Authorization` header, no amount of reconfiguration on the token side adds one. Switch the client to PAT-in-header mode (3.3) or move to REST (3.4).
- `bad request: Authorization header is badly formatted` — an empty bearer value or a scheme other than `Bearer`. The token may be perfectly valid; the header around it is not.
- `API rate limit exceeded` — `X-RateLimit-Reset` carries the reset timestamp.
- `dial tcp ... i/o timeout` — network layer. Raise the timeout or change transport.
- `RPC failed` / `GitRPC::BadObjectState` — server-side and transient. Retry with backoff, four attempts.

## Handoff

Three lines:

1. State — for example "reads and writes, via REST".
2. Next action — one command or one click.
3. Dead ends hit, if any — for example "client would not send a bearer token, fell back to REST". Skip this line when the path was clean.

## Credential hygiene

Store a token in a file only the owner can read:

```sh
umask 077 && printf '%s' "$TOKEN" > ~/.gh_token
```

On a platform without `umask`, restrict the file through the OS interface instead; the point is that no other account can read it.

Reference the path afterwards. Do not echo the value, print its length, commit it, or put it in a URL.

Users paste tokens into the chat. Accept it, store it, do not repeat it back, and say once at the end that the value now sits in conversation history and is worth rotating.
