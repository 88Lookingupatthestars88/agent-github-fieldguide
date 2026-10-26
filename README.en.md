# agent-github-fieldguide

[中文](README.md) · English

Two skills for an agent working on GitHub. `github-onboard` handles sign-up, tokens and the first repository read that works; `github-operate` handles writing files, branching, pull requests and releases. Neither needs the other.

A skill is one folder with a SKILL.md in it, and the client loads it under the folder's name.

## Download

- [github-onboard-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-onboard-SKILL.md) — 17466 B
- [github-operate-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-operate-SKILL.md) — 22269 B
- [agent-github-fieldguide-v1.0.0.zip](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/agent-github-fieldguide-v1.0.0.zip) — 21087 B, both skills with the folders already in place

Each file is one skill's SKILL.md and goes into a folder named after that skill:

```
your skills dir/github-onboard/SKILL.md
your skills dir/github-operate/SKILL.md
```

The folder name has to match the name line at the top of the file, or the client rejects the skill. The zip already carries both folders, so unpacking it into the skills directory is enough.

Where the skills directory lives differs per client. Search its settings for skills, or read its documentation.

## github-onboard

Use this skill when the agent has no credential GitHub will accept. It keeps going until a credential has actually read one repository.

- No account yet: sign up at https://github.com/signup, and correct whatever the page rejects according to the message it shows.
- Already have an account: sign in at https://github.com/login. Sign-in worked if https://github.com/settings/profile opens.
- Whether the client can finish OAuth depends on whether it ships a registered GitHub app. VS Code 1.101 and later, Visual Studio, JetBrains, Eclipse and Cursor do. A generic MCP connector ships none, cannot finish OAuth, and takes a token instead. The cause sits on GitHub's side: there is no dynamic client registration, so a client has nowhere to register itself.
- Without OAuth, this skill builds a fine-grained token: which permission this task needs, and the six operations no permission covers, where the token type has to change.
- It supports four transports: MCP with OAuth, MCP with a token in a header, plain REST, and git for clone and push.
- The last step is one real read. With a repository, it reads that repository. With none, it reads the repository list, where an empty array counts as success.

## github-operate

Use this skill once the agent can already read a repository and something needs to change.

- It reads a repository list, a directory tree, or a file's contents. Content over 1 MB needs the raw media type, and a list call returns 30 items unless `--paginate` asks for the rest.
- Changing one file starts by reading its current `sha`. A new file carries no `sha` field, and an empty one returns 422.
- Changing several files goes through the Git Data API as one commit, with `base_tree` set and the head re-read before the ref moves.
- Branches, pull requests, issues, workflow runs and releases are handled too. What each merge method leaves behind is stated before one is chosen. Release assets are separate objects, so moving the tag does not move them.
- Deletion starts with a numbered list, naming what each item takes down and what the list does not cover. It runs only after an explicit confirmation, one item at a time. A repository, a branch, a file and a local directory are four separate confirmations.

## Using both

`github-onboard` stops at the first repository read that works, and `github-operate` continues from there on the same token.

The first request in a session names the repository, and later ones can leave it out.

A credential obtained through OAuth exists only inside that client. REST calls or git from a shell need a token of their own.

## The endpoint answers four ways

Nothing here is needed to install; come back to it when a connection fails. The GitHub MCP endpoint answers according to what arrived with the request, and three of the four look like a token problem. `token authentication failed` is the only one that is.

### 401 bad request: missing required Authorization header

No Authorization header reached the server.

Most often the client is trying to do OAuth. GitHub does not implement dynamic client registration, so the client cannot register itself, gets no token, and sends the request without a header. A new token changes nothing, since no token took part in that request. Put a PAT in the header instead:

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

The token arrived and the server rejected it. It expired, it was revoked, or this endpoint does not accept that token type.

Read the expiry and the permissions, then decide whether to issue another.

### 400 bad request: Authorization header is badly formatted

The header is there and malformed: `Bearer` with nothing after it, or a scheme other than Bearer. The token may be fine; the header carrying it is not.

### 200 with mcp-session-id

The connection worked. The response carries an `mcp-session-id` header and `serverInfo` in the body. A client that reports success but cannot produce both did not connect.

## Install from source

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

Replace `~/.claude/skills/` with your client's skills directory, or use the download links above if there is no git. A client without skill support can take the file pasted into the system prompt.

## Feedback

Issues are open. If a step did not match what you saw, say which step it was and paste what came back instead, because the exact text shows which of the four causes it was. Replies are not guaranteed.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Not affiliated with GitHub, Inc.
