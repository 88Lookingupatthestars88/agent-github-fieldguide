# agent-github-fieldguide

[中文](README.md) · English

Two skills for an agent working on GitHub. `github-onboard` covers sign-up, tokens and the first successful read; `github-operate` covers writes, branches, pull requests and releases. Neither needs the other.

A skill is a folder with a `SKILL.md` in it, loaded by the client under the folder's name.

## Download

- [github-onboard-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-onboard-SKILL.md) — 17466 B
- [github-operate-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-operate-SKILL.md) — 22269 B
- [agent-github-fieldguide-v1.0.0.zip](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/agent-github-fieldguide-v1.0.0.zip) — 21087 B, both skills with the folders already in place

Each file is one skill's `SKILL.md`. Save it as `<skills dir>/github-onboard/SKILL.md` or `<skills dir>/github-operate/SKILL.md`. The folder name has to match the `name:` line at the top; a client that finds a mismatch rejects the skill. Unzipping the archive produces the same layout.

Where the skills directory is depends on the client: search its settings for skills, or for agent skills.

## github-onboard

Use it when the agent has no credential GitHub will accept. The skill ends at the first read that works.

- No account: https://github.com/signup. When a field is rejected, change it according to the message on the page.
- Has an account: https://github.com/login. Sign-in succeeded if https://github.com/settings/profile opens.
- Whether the client can finish OAuth depends on whether it ships a registered GitHub app. VS Code 1.101 and later, Visual Studio, JetBrains, Eclipse and Cursor do; a generic MCP connector has to bring its own, and without one the token route is what is left. The reason is on GitHub's side: there is no dynamic client registration, so a client has nowhere to register itself.
- Without OAuth, a fine-grained token: which permission the task needs, and the six operations no permission covers, where the token type has to change.
- Four transports: MCP with OAuth, MCP with a token in a header, REST, git for clone and push.
- The last step is one real read: the repository if there is one, otherwise the repository list, where an empty array counts.

## github-operate

Use it when the agent can already read a repository and something needs to change.

- Reads: repository list, file tree, file contents. Content over 1 MB needs the raw media type. A list call returns 30 items unless `--paginate` asks for the rest.
- One file, written after its current `sha` is read. A new file carries no `sha` field; an empty one returns 422.
- Several files in one commit through the Git Data API, with `base_tree` set and the head re-read before the ref moves.
- Branches, pull requests, issues, workflow runs, releases. What each merge method leaves behind is stated before one is chosen. Release assets are separate objects: moving the tag does not move them.
- Deletions get a numbered list with what each item takes down, a statement of what the list does not cover, and an explicit confirmation before the first call. Repository, branch, file and local directory are four separate confirmations.

## Using both

`github-onboard` ends at the first successful repository read. `github-operate` starts from there and can reuse the same token.

The first request in a session names the repository; later ones can leave it out.

A credential obtained through OAuth exists only inside that client. REST calls or git from a shell need a token of their own.

## The endpoint answers four ways

Nothing here is needed to install. Come back to it when a connection fails. The GitHub MCP endpoint answers according to what arrived with the request, and three of the four look like a token problem; `token authentication failed` is the only one that is.

### 401 bad request: missing required Authorization header

No Authorization header reached the server.

Most often the client is trying to do OAuth. GitHub does not implement dynamic client registration, so the client cannot register itself, gets no token, and sends the request without a header. A new token changes nothing, since no token took part in that request. Put a PAT in the header instead:

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

The token arrived and the server rejected it. Expired, revoked, or a type this endpoint does not accept.

Read the expiry and the permissions, then decide whether to issue another.

### 400 bad request: Authorization header is badly formatted

The header is present and malformed: `Bearer` with nothing after it, or a scheme other than Bearer. The token may be fine; the header carrying it is not.

### 200 with mcp-session-id

The connection worked. The response carries an `mcp-session-id` header and `serverInfo` in the body. A client reporting success without both did not connect.

## Install from source

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

Replace `~/.claude/skills/` with your client's skills directory. Without git, use the download links above. A client with no skills support can take SKILL.md pasted into the system prompt.

## Feedback

Issues are open. If a step did not match what you saw, say which step it was and paste what came back instead. The exact text shows which of the four causes it was. Replies are not guaranteed.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Not affiliated with GitHub, Inc.
