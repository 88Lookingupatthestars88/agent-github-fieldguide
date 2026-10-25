# agent-github-fieldguide

Two skills that go into an agent's skills folder.

**[Download v1.0.0](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/latest)** — unzip, and you get `github-onboard/` and `github-operate/`. Copy each into your client's skills directory.

## github-onboard — get the agent into your repositories

Ends the moment the agent can actually read a repository.

- No account: sign up in the browser. The skill points at the page and tells you what to do when a field rejects you, without reciting password rules that change.
- Has an account: sign in, then confirm it worked by loading the profile settings page.
- Works out whether your client can do OAuth at all. First-party IDEs ship a registered GitHub app; a generic MCP connector does not, and no amount of retrying fixes that.
- If OAuth is out, walks through a fine-grained token: which permission for which task, and which five things a fine-grained token can never do.
- Connects over one of three transports: MCP with OAuth, MCP with a token in a header, or plain REST.
- Finishes by reading a real repository. A token that was never used proves nothing.

## github-operate — keep the agent from breaking things

Covers everything after the credential works.

- Reading: list repositories, walk a directory tree, fetch a file. Large files go through the raw media type; the JSON form stops at 1 MB.
- Writing one file: reads the current `sha` first, because updating without it returns 409 and inventing one returns 422.
- Writing many files: one atomic commit through the Git Data API, with a refusal if someone else pushed first.
- Branches, pull requests, issues, workflow runs, releases. Includes what each merge method leaves behind, since none of them can be undone afterwards.
- Deletion: lists what would go, states what is not on the list, waits for an explicit yes, then removes one item at a time. A repository, a branch, a file and a local directory are separate confirmations, because the failures are not the same size.

## Where one stops and the other starts

`github-onboard` hands over once a repository read succeeds. `github-operate` never re-explains credentials; `github-onboard` never explains operations. Install one if that is all you need.

## Why the credential part is worth reading

The GitHub MCP endpoint returns four different answers depending on what arrived with the request. Three of them look like a token problem, and only one of them is.

### 401 bad request: missing required Authorization header

No Authorization header reached the server.

Most often the client is trying to do OAuth. GitHub does not implement dynamic client registration, so the client cannot register itself, gets no token, and sends the request without a header. A new token will not help here, since no token took part in the request. Put a PAT in the header instead:

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

The token arrived and the server rejected it. Expired, revoked, or of a type this endpoint does not accept.

Check the expiry date and permissions, then issue a new token.

### 400 bad request: Authorization header is badly formatted

The header is present but its value is malformed: `Bearer` with an empty value, or a scheme other than Bearer.

In this case the token may be valid and the header around it is not.

### 200 with mcp-session-id

The connection worked. The response carries a `mcp-session-id` header and `serverInfo` in the body. A client that shows success without those two is showing its own state, not the server's.

## Install

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

Replace `~/.claude/skills/` with your client's skills directory. If the client has no skills support, paste SKILL.md into the system prompt.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Not affiliated with or endorsed by GitHub, Inc.
