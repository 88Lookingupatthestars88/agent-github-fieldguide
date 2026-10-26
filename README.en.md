# agent-github-fieldguide

[中文](README.md) · English

Two skills for an agent working on GitHub. `github-onboard` covers sign-up, tokens and the first successful read; `github-operate` covers writes, branches, pull requests and releases. Neither needs the other.

## Download

- [github-onboard-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-onboard-SKILL.md) — 14788 B
- [github-operate-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-operate-SKILL.md) — 20246 B
- [agent-github-fieldguide-v1.0.0.zip](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/agent-github-fieldguide-v1.0.0.zip) — 19974 B, both skills with the folders already in place

Each file is one skill's `SKILL.md`. Save it as `<skills dir>/github-onboard/SKILL.md` or `<skills dir>/github-operate/SKILL.md`. The folder name has to match the `name:` line at the top of the file; a client that finds a mismatch rejects the skill. Unzipping the archive into the skills directory produces the same layout without the renaming.

The repository holds the same files; the download icon on the right of either one gives identical bytes.

## github-onboard

Use it when the agent has no credential GitHub will accept. The skill ends at the first read that works.

- No account: send the user to https://github.com/signup, then work from the error text when a field is rejected. Password rules are not listed, because they change.
- Has an account: https://github.com/login. Sign-in is confirmed by https://github.com/settings/profile loading; the form going away confirms nothing.
- Whether the client can finish OAuth depends on whether it ships a registered GitHub app. Desktop IDEs do; generic MCP connectors do not, and retrying does not add one.
- Without OAuth, a fine-grained token: which permission matches the task, and the five operations no permission unlocks.
- Four transports: MCP with OAuth, MCP with a token in a header, REST, git for clone and push.
- The last step is one real read: the repository if there is one, otherwise the repository list, where an empty array counts.

## github-operate

Use it when the agent can already read a repository and something needs to change.

- Reads: repository list, file tree, file contents. Content over 1 MB needs the raw media type; the JSON form stops at 1 MB.
- A single file is written after its current `sha` is read. An update without `sha` returns 409; a new file sent with an empty `sha` returns 422.
- Several files go in one commit through the Git Data API, with `base_tree` set and the head re-read before the ref moves.
- Branches, pull requests, issues, workflow runs, releases. Merge methods are described by what each one leaves behind, before any is chosen.
- Deletions get a numbered list of every item that would go, a statement of what the list does not cover, and an explicit confirmation before the first call. Repository, branch, file and local directory are four separate confirmations.

## Using both

`github-onboard` ends at the first successful repository read. `github-operate` starts from there and can reuse the same token. Neither needs the other installed.

The first request in a session names the repository; later ones can leave it out.

One thing to know: a credential obtained through OAuth exists only inside that client. REST calls or git from a shell still need a token of their own.

## The endpoint answers four ways

The GitHub MCP endpoint answers according to what arrived with the request. Three of the four look like a token problem; only the token-authentication one actually is.

### 401 bad request: missing required Authorization header

No Authorization header reached the server.

Most often the client is trying to do OAuth. GitHub does not implement dynamic client registration, so the client cannot register itself, gets no token, and sends the request without a header. A new token does not change this, because no token took part in that request. Put a PAT in the header instead:

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

The token arrived and the server rejected it. Expired, revoked, or a type this endpoint does not accept.

Check the expiry date and permissions, then issue a new token.

### 400 bad request: Authorization header is badly formatted

The header is present and its value is malformed. `Bearer` with nothing after it, or a scheme other than Bearer.

The token itself may be fine; the header carrying it is not.

### 200 with mcp-session-id

The connection worked. The response carries an `mcp-session-id` header and `serverInfo` in the body. If a client reports success without both, that report describes the client, not the endpoint.

## Install from source

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

Replace `~/.claude/skills/` with your client's skills directory. A client with no skills support can take SKILL.md pasted into the system prompt.

## Feedback

Issues are open. If a step did not match what you saw, say which step it was and paste what came back instead. The exact text shows which of the four causes it was.

Replies are not guaranteed.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Not affiliated with or endorsed by GitHub, Inc.
