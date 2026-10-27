---
name: github-onboard
description: Gets an agent from no credential to a token that has read a real repository. Covers sign-up, sign-in, two-factor enrolment, OAuth versus personal access token, MCP wiring, and the errors that follow a failed authentication. Reading and writing after that belong to github-operate. 注册 / 登录 / 两步验证 / 建 token / token 怎么弄 / GitHub 连不上 / 401 / unauthorized.
license: Apache-2.0
metadata:
  author: 88Lookingupatthestars88
  version: "1.2.0"
  source: https://github.com/88Lookingupatthestars88/agent-github-fieldguide
---

# GitHub Onboarding

Scope is one credential that reads a real repository. Everything after that is github-operate.

## Step 0 — What you can run

Three capabilities decide the route: shell, HTTP, file read and write. Check which exist before planning, and never claim one you have not used this session. Do not ask the user what environment they are in, because they usually cannot tell.

## Step 1 — Where the user stands

Ask both in one message:

> Do you already have a GitHub account?
> If yes, which repository should I reach? Owner/name or URL.

- No account — Step 2 creates one, then 2.5, then 3.
- Account already exists — Step 2 is a sign-in and nothing else, then 2.5, then 3. Nothing in Step 3 works without a session: a token cannot be issued and a repository cannot be created while signed out.
- Account, and a repository is named — the same path, and Step 4 verifies against that repository.
- Account, no repository yet — the same path, and Step 4 verifies against the account.
- Something worked before and now errors — get the exact error text, then read Errors.
- Vague answer — assume an account exists. Reading Errors settles it faster than a duplicate account does.

## Step 2 — Sign up or sign in

Both happen in a browser and both are the user's to finish. Email verification, codes and human checks stay with them. An existing account lands here for one reason only: to obtain a session. Everything in Step 3 needs that session, and nothing in Step 3 can substitute for it.

New account: https://github.com/signup
Existing account: https://github.com/login

Signed in is confirmed when https://github.com/settings/profile loads.

Then settle scope before touching credentials:

> Do you want me to read your repositories, or also write to them?

Default to read when the answer is unclear; widening later costs one edit.

## Step 2.5 — The second factor

Three jobs need two-factor authentication and fail without it: creating a repository, publishing a release, and issuing a token that carries write or administration. A session that only reads can proceed without it, and every other session does this step now.

Read https://github.com/settings/security first. A filled row under Two-factor authentication means the account already has one and this step is done.

Which methods appear there depends on the account. Passkeys need a platform authenticator, the mobile app needs push notifications that arrive, and SMS reaches only the countries listed at https://docs.github.com/en/authentication/securing-your-account-with-two-factor-authentication-2fa/countries-where-sms-authentication-is-supported. Look the user's country up before offering SMS, and move to a TOTP application when it is absent.

Do not recommend a specific app. State the three properties that make one fit and let the user choose:

- It implements RFC 6238, so a secret issued by GitHub produces the right code.
- It exports a vault that restores on a new device.
- It computes offline, with no account login.

The device floor comes from the package's own `minSdkVersion`, not from a store page.

Setup order:

1. Scan the QR code, and copy the setup key printed below it.
2. Enter the six-digit code from the app.
3. Download the sixteen recovery codes. Each works once.
4. Store them in two places that fail independently. A repository is not one of them, and neither is a chat transcript.

The TOTP secret is a permanent second factor for the whole account. It stays out of the repository, out of commits and out of messages. When the agent has to produce codes unattended, treat the file like a token: outside any tree, readable by its owner alone.

Enrolment is confirmed by https://github.com/settings/security showing the row filled in, not by the user reporting success.

## Step 3 — Connect

Route before reading further:

- User asks for a token, or already has one — 3.3.
- User wants one-click authorization on a client that ships a GitHub app — 3.1, then 3.2.
- Client has no registered app — 3.3.
- The job needs clone, push or history — 3.5, after the credential works.

### 3.1 Whether OAuth can finish here

GitHub runs no dynamic client registration, so a client can only complete OAuth if it already ships a registered GitHub app. Two signals separate the clients, and both are visible without the user's help:

- A built-in GitHub MCP entry, or a one-click GitHub install. An app exists.
- A prompt for client ID, client secret or callback URL. No app exists, so OAuth cannot finish.

With neither signal, have the user add the server and report what appeared. A consent screen means the app exists; a form demanding a client ID means it does not. This step is reported by the user, not clicked by the agent.

VS Code 1.101 and later, Visual Studio, JetBrains, Eclipse and Cursor ship one. Generic remote-MCP connectors, self-hosted clients and mobile apps ship none, and take a token instead.

An OAuth credential lives inside the client afterwards. MCP tool calls can use it and a shell cannot, so an agent that also calls REST or runs git needs its own token even after OAuth succeeds.

A fine-grained token pins the reach to named repositories and named permissions and costs about two minutes. Take OAuth for speed on the user's own machine, and a token when the reach has to be limited.

### 3.2 MCP over OAuth

The remote endpoint is `https://api.githubcopilot.com/mcp/`. Confirm it answers before wiring anything:

```sh
ENDPOINT=https://api.githubcopilot.com/mcp/
curl -s -D - -o /dev/null -X POST "$ENDPOINT" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}'
```

A 401 carrying a `www-authenticate` header is the healthy answer, and the metadata document it links outranks this file. A 404 means the path moved; https://github.com/github/github-mcp-server carries the current one. Narrower tool surfaces live under the same path, with `/mcp/x/repos` for repository tools only. `000` means the host is unreachable from here.

The body of that 401 separates four cases that two status codes cannot:

```
no Authorization header   401  bad request: missing required Authorization header
empty "Bearer " value     400  bad request: Authorization header is badly formatted
wrong or expired token    401  unauthorized: token authentication failed
non-Bearer scheme         400  bad request: Authorization header is badly formatted
```

`token authentication failed` means a token arrived and the server refused it. `missing required Authorization header` means nothing arrived, so the client is not attaching one and reissuing changes nothing.

Then the client runs the flow and the agent confirms the result.

### 3.3 MCP over a PAT

Same endpoint, credential in a header:

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

A client that accepts only commands takes the local Docker variant documented at https://github.com/github/github-mcp-server. It needs a running Docker daemon and a token.

Three signals together mean the connection is real: HTTP 200, an `mcp-session-id` response header, and `serverInfo` in the body. A client reporting "connected" without all three is not connected.

### 3.4 REST over a PAT

This is the route when the user wants a token, when MCP is unavailable, or when the agent has to act from a shell.

- https://github.com/settings/personal-access-tokens/new — fine-grained, the default
- https://github.com/settings/tokens/new — classic, where `repo` covers every private repository in full

Six operations have no fine-grained equivalent, so no permission unlocks them and the token type has to change: writing to a public repository the account is not a member of, acting as an outside collaborator, reaching more than one organization with one token, Packages, the Checks API, and a project owned by a user account.

Grant by task, never by convenience: contents and issues for reading, pull requests when PRs are in scope, actions and workflows for CI, administration only when repository settings have to change. Administration carries deletion with it.

Permission names drift. Check https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens rather than a list here.

```sh
curl -sS -H "Authorization: Bearer $TOKEN" https://api.github.com/user
```

A login name in the response proves the credential. `gh api user --jq .login` answers the same question and is safer than `gh auth status`, which contacts `github.com` and fails where only `api.github.com` is reachable.

### 3.5 git

Only for clone, push or history. Keep the token out of the clone URL, where it lands in `.git/config` and shell history; use a credential helper.

One failure reads as an auth problem and is not:

```
fatal: could not read Username for 'https://<host>'
```

Usually the machine rewrites `github.com` to a mirror that does not forward private-repo credentials. Check the git config for `insteadOf` rules before blaming the token. Under that configuration public clones succeed and private clones fail.

## Step 4 — Prove it reads the repository

A valid token does not prove reach. Make one read against the target using the transport just configured.

```sh
curl -sS -H "Authorization: Bearer $TOKEN" https://api.github.com/repos/<owner>/<repo>
```

A 200 is the proof. Over MCP, call one read-only tool against the same target and take tool names from `tools/list`; a JSON-RPC error frame is a failure even when the connection reports healthy.

A 404 is ambiguous on either route, because GitHub answers 404 instead of 403 when a credential lacks access. Recheck the token's repository selection before telling anyone the repository does not exist.

With no repository on the account, read `https://api.github.com/user/repos` and accept 200 with an empty array as proof the credential works.

Read `github-authentication-token-expiration` from the response headers. On a fine-grained or OAuth token it carries the expiry in UTC; on a classic token its absence means no expiry was set. A new token can take a few seconds to propagate, so retry a 401 once before calling it dead.

## Errors

Authentication failures only. Permission and object failures belong to github-operate Step 11.

- `401 Bad credentials` — mistyped, revoked or expired. Issue a new one.
- `unauthorized: token authentication failed` — a token reached the server and was refused. Check the expiry date, then the token type. Reissuing helps only when the token itself is the problem.
- `bad request: missing required Authorization header` — no credential was sent. The client is not attaching the header, often because it is still attempting OAuth with no registered app behind it. Switch to 3.3 or 3.4.
- `bad request: Authorization header is badly formatted` — an empty bearer value, or a scheme other than `Bearer`.
- Settings pages answering the 2FA gate instead of the form — the deadline passed with no second factor enrolled. No token clears it. An existing token keeps working; editing one does not, because the settings page sits behind the gate.
- `dial tcp ... i/o timeout` — network layer. Raise the timeout or change transport.

## Is this copy current

`metadata.version` above is the release this copy shipped with, the tag without its `v`, and `metadata.source` is where it came from. The newest release answers one call:

```sh
gh api repos/88Lookingupatthestars88/agent-github-fieldguide/releases/latest --jq '.tag_name, .published_at'
```

A tag above that number means the installed copy is older than what shipped. Report both numbers and let the user decide, because replacing the file is their edit.

## Handoff

Three lines:

1. State, for example "reads and writes, via REST".
2. Next action: one command or one click.
3. Dead ends hit, if any. Skip this line when the path was clean.

## Credential hygiene

Store a token in a file only its owner can read:

```sh
umask 077 && printf '%s' "$TOKEN" > ~/.gh_token
```

On a platform without `umask`, restrict the file through the OS interface. What matters is that no other account can read it.

Reference the path afterwards. Do not echo the value, print its length, commit it, or put it in a URL.

Users paste tokens into the chat. Accept it, store it, do not repeat it back, and say once at the end that the value now sits in conversation history and is worth rotating.
