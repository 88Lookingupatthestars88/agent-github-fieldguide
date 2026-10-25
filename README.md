# agent-github-fieldguide

Two skills for agents that work with GitHub. Take one, or both.

## Download

Three files, pick what you need:

- [github-onboard-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-onboard-SKILL.md) — 14 KB
- [github-operate-SKILL.md](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/github-operate-SKILL.md) — 18 KB
- [both, as one zip](https://github.com/88Lookingupatthestars88/agent-github-fieldguide/releases/download/v1.0.0/agent-github-fieldguide-v1.0.0.zip) — 19 KB

Each single file is the `SKILL.md` of one skill. Put it in a folder named after the skill and rename it:

```
<skills dir>/github-onboard/SKILL.md
<skills dir>/github-operate/SKILL.md
```

The folder name has to match the `name:` line at the top of the file. A client that finds `github-onboard/SKILL.md` inside a folder called `onboard` will reject it.

The zip already has the folder structure, so unzipping into your skills directory needs no renaming.

You can also open either file in the repository and use the download icon on the right. Same file.

## github-onboard

For when the agent has no working way into GitHub yet. It runs until the agent has read one of your repositories, and then it stops.

- No account: points you at the signup page, then reads the error back to you when a field gets rejected. It does not recite password rules; those change.
- Has an account: sign in, then load the profile settings page. That page loading is the proof, not the sign-in form disappearing.
- Works out whether your client can finish OAuth. Desktop IDEs ship a registered GitHub app. Most generic MCP connectors do not, and retrying never adds one.
- When OAuth is not on the table, builds a fine-grained token: which permission to tick for the task in front of you, and the five things a fine-grained token cannot do no matter what you tick.
- Connects through one of three transports: MCP with OAuth, MCP with a token in a header, plain REST.
- Reads a real repository at the end. Nothing counts as done before that.

## github-operate

For once the agent can read a repository and you want it to change something.

- Reads: repository list, file tree, file contents. Anything over 1 MB needs the raw media type; the JSON form stops there.
- Writes one file, reading the current `sha` first. An update without it returns 409, a made-up one returns 422.
- Writes several files as one commit through the Git Data API, and refuses if the branch moved while it was working.
- Branches, pull requests, issues, workflow runs, releases. Tells you what each merge method leaves behind before you pick one.
- Deletes. Lists every item that would go, says what the list does not cover, waits for an explicit yes, then removes one item. A repository, a branch, a file and a local directory are four separate confirmations, because they are not the same size of mistake.

## The two together

`github-onboard` stops at the first successful repository read. `github-operate` picks up the next request.

First run after installing: the agent asks which repository. After that it works from what you ask for.

Install one if that is all you need. They do not depend on each other.

## Why the credential part is worth reading

The GitHub MCP endpoint returns four different answers depending on what arrived with the request. Three of them look like a token problem, and only one of them is.

### 401 bad request: missing required Authorization header

No Authorization header reached the server.

Most often the client is trying to do OAuth. GitHub does not implement dynamic client registration, so the client cannot register itself, gets no token, and sends the request without a header. A new token will not help, since no token took part in that request. Put a PAT in the header instead:

```json
{ "Authorization": "Bearer github_pat_xxx" }
```

### 401 unauthorized: token authentication failed

The token arrived and the server rejected it. Expired, revoked, or a type this endpoint does not accept.

Check the expiry date and permissions, then issue a new token.

### 400 bad request: Authorization header is badly formatted

The header is present and its value is malformed. `Bearer` with nothing after it, or a scheme other than Bearer.

The token itself may be fine. What surrounds it is not.

### 200 with mcp-session-id

The connection worked. The response carries an `mcp-session-id` header and `serverInfo` in the body. A client that reports success without both is reporting its own state.

## Install from source

```sh
git clone https://github.com/88Lookingupatthestars88/agent-github-fieldguide
cp -r agent-github-fieldguide/github-onboard ~/.claude/skills/
cp -r agent-github-fieldguide/github-operate ~/.claude/skills/
```

Replace `~/.claude/skills/` with your client's skills directory. A client with no skills support can take SKILL.md pasted into the system prompt.

## Feedback

Issues are open. If a step in either skill did not match what you saw, say which step and what came back instead. The exact response text matters more than a description of it.

There is no support promise attached to this. It is two files and one person.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Not affiliated with or endorsed by GitHub, Inc.
