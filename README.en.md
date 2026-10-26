# agent-github-fieldguide

[中文](README.md) · English

Two Agent Skills that let an agent work GitHub on its own. `github-onboard` handles the credential; `github-operate` handles reading, writing, pull requests and releases. Both are plain text and take effect once the client loads them.

## Download

- [github-onboard-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.1.0/github-onboard-SKILL.md) — 17283 B
- [github-operate-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.1.0/github-operate-SKILL.md) — 28551 B
- [agent-github-fieldguide-v1.1.0.zip](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.1.0/agent-github-fieldguide-v1.1.0.zip) — 23294 B, both skills with their folders

## Install

Unzipping gives you two folders. Put both folders into the client's skills directory, like this:

```
your skills dir/github-onboard/SKILL.md
your skills dir/github-operate/SKILL.md
```

The folder name has to match the `name` on the second line of the file, or the client will not load it.

Skills directories differ per client. Search the settings for skills; if there is no such setting, create the directory where the client's documentation says.

Downloading a single SKILL.md works too, as long as you make the folder yourself. A client without skill support can take the whole SKILL.md pasted into its system prompt.

To clone instead:

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

Replace `~/.claude/skills/` with your client's actual path.

## github-onboard

Use it when no working credential exists yet. The job ends at a real repository read through that credential, not at a token in hand.

- No account: sign up at https://github.com/signup and correct whatever the page rejects.
- Already have one: sign in at https://github.com/login. Sign-in worked if https://github.com/settings/profile opens.
- Whether OAuth can finish depends on the client shipping a registered GitHub app. VS Code 1.101 and later, Visual Studio, JetBrains, Eclipse and Cursor do; a generic MCP connector does not, and takes a token instead. The cause sits on GitHub's side: no dynamic client registration, so a client has nowhere to register itself.
- Without OAuth it builds a fine-grained token: the permission follows the task, and six operations have no fine-grained permission at all, so those need a classic token.
- Four transports: MCP over OAuth, MCP with a token in a header, plain REST, and git for clone or push.
- The last step is one real read. With no repository on the account, it reads the repository list, where an empty array also counts.

## github-operate

Use it once the credential works and something has to change.

- It reads a repository list, a directory tree or a file's contents. Content over 1 MB needs the raw media type, and a list call returns 30 items unless paging is asked for.
- Changing one file starts by reading its current `sha`. A new file carries no `sha` field, and an empty one returns 422.
- Changing several files goes through the Git Data API as one commit, with `base_tree` set and the head re-read before the ref moves.
- Branches, pull requests, issues, workflow runs and releases are covered. What each merge method leaves behind is stated before one is chosen, and release assets are separate objects that a moved tag does not carry.
- Deletion starts with a numbered list naming what each item takes down and what the list leaves out. It runs only after an explicit confirmation, one item at a time, and a repository, a branch, a file and a local directory are four separate confirmations.
- Repository settings are covered too: description, homepage, topics, the Discussions switch. Topics go through PUT, because PATCH answers 200 without changing a single field.

## Using both

`github-onboard` stops at the first repository read that works, and `github-operate` continues from there on the same token.

The first request in a session names the repository, and later ones can leave it out.

A credential obtained through OAuth exists only inside that client. REST calls or git from a shell need a token of their own.

## When a connection fails, check these four answers

Nothing here is needed to install. The GitHub MCP endpoint answers according to what arrived with the request, and three of the four look like a token problem.

### 401 bad request: missing required Authorization header

No Authorization header reached the server. Most often the client is still trying to do OAuth: GitHub has no dynamic client registration, so the client cannot register itself, gets no token, and sends the request without a header. A new token changes nothing, because no token took part in that request. Put a PAT in the header instead:

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

The token arrived and the server rejected it: expired, revoked, or a type this endpoint does not accept. Read the expiry and the permissions, then decide whether to issue another.

### 400 bad request: Authorization header is badly formatted

The header is there and malformed: `Bearer` with nothing after it, or a scheme other than Bearer. The token may be fine; the header carrying it is not.

### 200 with mcp-session-id

The connection worked. The response carries an `mcp-session-id` header and `serverInfo` in the body, and a client that reports success without producing both did not connect.

## Feedback

Issues are open. If a step did not match what you saw, say which step it was and paste what came back instead; the exact text shows which of the four causes it was.

Usage questions belong in Discussions under Q&A, where an answer can be marked as accepted. Replies are not guaranteed.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Not affiliated with GitHub, Inc.
