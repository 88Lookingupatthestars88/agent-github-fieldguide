---
name: github-operate
description: Does the work once the agent can already read a repository. Use when the agent needs to list repositories, read or change files, create branches, commit, open or merge a pull request, file or close an issue, inspect or trigger a workflow run, cut a release, change repository settings, or delete a repository, branch, file or directory. Also trigger on 提交代码 / 建分支 / 开 PR / 提 issue / 看 CI / 发 release / 删仓库 / gh CLI / api.github.com / Git Data API / blobs / trees / workflow_dispatch / Actions logs. Assumes a working credential; when there is none, use github-onboard first.
license: Apache-2.0
metadata:
  author: 88Lookingupatthestars88
  version: "1.1"
---

# GitHub Operations

Nothing here covers obtaining a credential. If the agent has no working one, use `github-onboard` first.

## Step 0 — What you can call

Three routes to the same API. Find out which exists before writing a command:

- MCP tools are loaded. Take tool names from the server, not from this file. Use these when one call covers the whole job.
- A shell with `gh` installed. Confirm with `gh --version`.
- A shell without `gh`, but HTTPS works. A bearer header on a plain request does everything the other two do.

Every command in this file is written in the `gh` form. The other two routes take the same parameters in a different wrapper:

- MCP: the arguments mirror the endpoint's fields. When a tool asks for `path`, `content`, `message`, `sha` and `branch`, it is doing what Step 3 does; the name it carries is the server's choice.
- Raw HTTPS: replace `gh api <path>` with a request to `https://api.github.com/<path>` plus `Authorization: Bearer`. Replace `-f k=v` with a JSON body field. `--jq` is a client-side filter and has no server equivalent.

Pick one route and keep it for the session. Switching mid-change means re-reading the object through the new route before continuing.

An OAuth credential that lives inside an MCP client does not reach a shell. If there is no token and the shell is the only route that can do the job, go back to `github-onboard`.

The `sh` fences assume a POSIX shell. Adapt the wrapper, keep the parameters.

- `base64 -w0` is GNU. macOS `base64` wraps by default; `openssl base64 -A` works on both and on Windows when OpenSSL is installed.
- Heredocs do not exist in `cmd.exe` or PowerShell. Write the JSON to a file and pass its path to `--input`.
- `jq` may be absent even where `gh` exists. `gh --jq` carries its own filter engine, so the commands here never call `jq` directly.

## Step 1 — Who you are acting as

Two reads, once per session:

```sh
gh api user --jq .login
gh api repos/<owner>/<repo> --jq '{perm: .permissions, default: .default_branch, private: .private}'
```

`permissions` reports `admin`, `maintain`, `push`, `triage`, `pull` as booleans. Read them instead of probing with a write. A later 403 is answered by one of these values.

Write access and repository visibility are different things. `push: true` on a public repository still means every read is anonymous unless the token is sent.

## Step 2 — Reading

```sh
gh repo list --limit 30                     # repositories the token's account owns
gh api 'repos/<owner>/<repo>/git/trees/<branch>?recursive=1' --jq '.tree[].path'
gh api repos/<owner>/<repo>/contents/<path> --jq .content | base64 -d
gh api repos/<owner>/<repo>/contents/<path> -H 'Accept: application/vnd.github.raw'
```

Which call to use:

- A whole directory tree is one `git/trees?recursive=1` call. `truncated: true` means the tree hit its limit; drop `recursive` and walk directory by directory.
- `contents` in JSON form returns base64 and stops at 1 MB. The raw media type reaches 100 MB. Past that, fetch the blob through `git/blobs/<sha>` or clone.
- `raw.githubusercontent.com` serves cached content. A file pushed seconds ago can return the previous version. Read through the API with `?ref=<sha>` when freshness matters.
- Base64 from `contents` arrives wrapped. `base64 -d` accepts the newlines; macOS spells the same flag `-D`. `openssl base64 -d -A` works everywhere OpenSSL exists.
- A commit's file list without the diff: `gh api repos/<owner>/<repo>/commits/<sha> --jq '.files[] | {filename, status, additions, deletions}'`.

## Step 3 — Changing one file

```sh
SHA=$(gh api repos/<owner>/<repo>/contents/<path>?ref=<branch> --jq .sha)
gh api -X PUT repos/<owner>/<repo>/contents/<path> \
  -f message='<what changed and why>' \
  -f content="$(base64 -w0 <local-file>)" \
  -f sha="$SHA" \
  -f branch='<branch>'
```

Updating an existing file requires `sha`. Creating a new one must omit it, so when the file does not exist yet, skip the read: an empty `sha=""` is sent as a field and returns 422, while a missing field is correct.

Which case you are in is decided by that first read. A 404 there means new file, and the PUT then carries no `sha` at all.

Always pass `branch`. Without it the write lands on the default branch, which is rarely the intent when a session is about to open a pull request.

The `message` becomes permanent history. One line saying what changed and why; `update` and `fix` leave nothing for the next reader.

Content is base64 without line breaks. See the platform note in Step 0 for the encoder.

A path with spaces, `#`, `?` or non-ASCII characters has to be percent-encoded inside the URL. A space sent raw becomes a different path, and the write succeeds against the wrong file or returns 404.

## Step 4 — Changing several files at once

Atomic, one commit, four writes after two reads. Worth it at three or more files, or whenever a partial apply would leave the repository broken. One or two files are cheaper through Step 3.

Arrays of objects do not survive `-f` flags. Pass JSON with `--input`. The heredocs below are unquoted on purpose so the shell substitutes the variables; base64 uses only `A-Z a-z 0-9 + / =`, so nothing in the payload collides with shell expansion.

```sh
# two reads: head commit, then the tree that commit points at
HEAD=$(gh api repos/<owner>/<repo>/git/ref/heads/<branch> --jq .object.sha)
BASE=$(gh api repos/<owner>/<repo>/git/commits/$HEAD --jq .tree.sha)

# 1. one blob per file, repeat until every file has a SHA
BLOB=$(gh api -X POST repos/<owner>/<repo>/git/blobs --input - --jq .sha <<JSON
{"content": "$(base64 -w0 a.txt)", "encoding": "base64"}
JSON
)

# 2. one tree listing every path you touch, rooted at the current tree
TREE=$(gh api -X POST repos/<owner>/<repo>/git/trees --input - --jq .sha <<JSON
{"base_tree": "$BASE",
 "tree": [{"path": "a.txt", "mode": "100644", "type": "blob", "sha": "$BLOB"},
          {"path": "gone.txt", "mode": "100644", "type": "blob", "sha": null}]}
JSON
)

# 3. commit pointing at the new tree
NEW=$(gh api -X POST repos/<owner>/<repo>/git/commits --input - --jq .sha <<JSON
{"message": "<what changed and why>", "tree": "$TREE", "parents": ["$HEAD"]}
JSON
)

# 4. move the ref, refusing if it moved meanwhile
gh api -X PATCH repos/<owner>/<repo>/git/refs/heads/<branch> --input - <<JSON
{"sha": "$NEW", "force": false}
JSON
```

A tree entry with `"sha": null` deletes that path. A path left out of `tree` is unchanged; the new tree is `base_tree` plus the entries listed.

That last sentence holds only because `base_tree` is in the request. Omit `base_tree` and the call means the opposite thing: the tree becomes exactly what you listed, and every other path in the repository is deleted. The call still returns 201, and the response lists only the entries you sent, so nothing in it marks the ones that went away.

Send `base_tree` on every call meant to modify an existing repository, and count the result after any tree write: `GET /git/trees/<new>?recursive=1` should show the previous blob count plus the paths you added, minus the paths you deleted. A count taken from the request you sent is not a check.

When one change has to move both a branch and a tag, move the branch first and confirm it before touching the tag. A tag moved first stays on the new commit even if the branch move is rejected, and it then points at a commit that no branch reaches. `PATCH /git/refs/tags/<tag>` takes `force: true`, so nothing rejects that mistake.

`force: false` makes step 4 fail when someone pushed first. With `true` the move overwrites their commit and the API still reports success.

`git/ref/heads/<branch>` returns a commit SHA. Feeding it to `base_tree` returns 422; the tree SHA comes from the second read.

Deleting files this way is destructive. Step 10 applies: list every path that goes away, wait for a named confirmation.

## Step 5 — Branches

```sh
HEAD=$(gh api repos/<owner>/<repo>/git/ref/heads/<base> --jq .object.sha)
gh api -X POST repos/<owner>/<repo>/git/refs -f ref='refs/heads/<new>' -f sha="$HEAD"
```

`ref` needs the full `refs/heads/` prefix. `heads/<new>` returns 422.

A repository with no commits has no default branch and no head SHA. There the first commit is Step 3 with `branch` set to the intended default branch name and no `sha`; GitHub creates the branch with that commit.

Deleting a branch is Step 10. `PATCH /git/refs` with `force: true` and `git push --force` rewrite history that others may already have; both count as destructive there.

## Step 6 — Pull requests

```sh
gh pr create --repo <owner>/<repo> --base <base> --head <branch> --title '<...>' --body '<...>'
gh pr view <n> --repo <owner>/<repo> --json state,mergeable,mergeStateStatus,files
gh pr merge <n> --repo <owner>/<repo> --squash
```

What each merge method leaves behind:

- merge keeps every commit and adds a merge commit; the base branch shows the branch topology.
- squash collapses to a single commit; the original commits are no longer reachable from the base branch.
- rebase replays the commits onto the base with new SHAs; anyone holding the old ones now has a divergent branch.

`--delete-branch` deletes the head branch as part of the merge, which is a deletion and therefore Step 10. Run the merge without the flag unless the user already agreed to losing the branch; then delete it as its own confirmed item.

Without the flag the head branch stays. Deleting a branch that still holds unmerged commits drops those commits from the branch list; they stay reachable by SHA through the API until garbage collection runs, which has no user-facing trigger.

`mergeable: false` with `mergeStateStatus: dirty` means conflicts against the base. Resolution happens on the head branch; the API has no server-side conflict resolution.

## Step 7 — Issues

```sh
gh issue create --repo <owner>/<repo> --title '<...>' --body '<...>'
gh issue comment <n> --repo <owner>/<repo> --body '<...>'
gh issue close <n> --repo <owner>/<repo> --reason completed
```

Issues need `issues: write` on a fine-grained token. A pull request conversation accepts the same comment endpoint, but a token scoped only to issues fails there, which reads like a permissions bug and is not.

Closing is reversible through `gh issue reopen`. REST has no endpoint that deletes an issue; do not promise one.

## Step 8 — Workflows

```sh
gh run list --repo <owner>/<repo> --limit 10 --json databaseId,status,conclusion,headBranch,event
gh run view <id> --repo <owner>/<repo> --log-failed
gh run watch <id> --repo <owner>/<repo>
gh workflow run <file>.yml --repo <owner>/<repo> --ref <branch>
```

`conclusion` reads `success`, `failure`, `cancelled`, `skipped`, or `null` while the run is still going. `null` with `status: in_progress` is not a failure.

Logs come back as a zip behind a redirect. `gh run view --log` and `--log-failed` unpack them. A raw HTTP client must follow the 302 and unzip; the archive read as text is binary noise, not an error message.

`workflow_dispatch` needs the workflow file to exist on the default branch, even when `--ref` names another branch. A 404 on dispatch usually means the workflow only lives on a feature branch, so either merge it first or send the event another way.

Triggering needs `actions: write`. Reading logs needs `actions: read`.

A push authenticated with a PAT starts workflows. A push made with a workflow's own `GITHUB_TOKEN` does not, which is deliberate: it prevents a workflow from triggering itself in a loop. A CI chain that stops after the first job is behaving as designed.

## Step 9 — Releases

```sh
gh release create <tag> --repo <owner>/<repo> --target <sha> --title '<...>' --notes '<...>'
gh release view <tag> --repo <owner>/<repo> --json tagName,targetCommitish,isDraft,assets
```

If the tag does not exist, it is created at `--target`. If it does exist, it is reused, so a mistyped tag publishes against whatever commit already carries it. Read the release back and compare `targetCommitish` against the commit you meant.

`--draft` keeps the release invisible until someone publishes it. `--prerelease` marks it without hiding it.

Deleting a release leaves the tag behind. Removing both takes two calls, and both belong to Step 10.

## Step 10 — Deleting and other irreversible changes

Covers: deleting a repository, a branch, a file, a directory, or a release; changing visibility; changing the default branch; force-pushing; transferring ownership.

### The rule

A user asking for a deletion is a request to plan one, not permission to run one. Do not call any endpoint that removes something until a confirmation arrives, and only for the items that confirmation names.

### Sequence

1. Read the current state of everything in scope and state it back in one short block: repository, branch, path, size, last commit, visibility.
2. Produce a numbered list of what would be removed, one line per item. Include every item the phrasing could reach, not just the first one that came to mind. Mark items you believe are out of scope, so the user can rule them in.
3. State what is not covered by the list — other branches, forks, local copies, open pull requests, releases, tags, CI artifacts.
4. State where a copy exists. When none exists, say that outright instead of implying one.
5. Ask the user to name the numbers. "All" still needs the list acknowledged once, since the list is what will actually run and "all" is a shorthand for it.
6. Execute one item at a time. After each, read the object back and report the status code and what the API now returns.
7. Stop after the named items. Do not continue to the rest because they were in the same list.

Words that count as confirmation: 确认, 删, 做吧, 可以删, yes, go ahead, delete it. Words that do not: ok, 好的, 嗯, a question about something else, silence. A confirmation for item 2 covers item 2 only.

Shape of the message, filled in from the two reads above:

```
You asked to remove: <restate the request in one line>

What is there now:
  <owner/repo>, <branch or path>, <size or commit count>, last commit <sha> <date>, <visibility>

Would be removed:
1. <object> — <what goes with it>
2. <object> — <what goes with it>
3. <object> — <marked out of scope, listed so you can rule it in>

Not in this list: <other branches, forks, local copies, releases, tags>
A copy exists at: <path> | none

Reply with the numbers to remove. Everything not named stays.
```

### Which scope the request actually means

The same sentence can point at three different places, and deleting the wrong one is how a working copy disappears:

- The remote repository. `DELETE /repos/<owner>/<repo>` removes the repository, its issues, pull requests, releases, and every fork loses its parent link.
- Something inside the remote repository. A branch (`DELETE /git/refs/heads/<branch>`), a file or directory (Step 4 with `sha: null`, one commit), a release (`DELETE /releases/<id>`), a tag (`DELETE /git/refs/tags/<tag>`).
- A path on the local machine. `rm -rf` on a workspace directory, a clone, a cache. Nothing on GitHub changes, and GitHub cannot restore it.

When a request mixes two of these — "delete the test repo and the local clone" — split them into separate numbered items and treat them as separate confirmations. A single yes does not cover both, because the two failures are not the same size and the second one is unrecoverable.

Before running any local `rm -rf`, list the absolute path and the file count, and confirm the path resolves where you think it does. A relative path that matched nothing, or matched the parent, both print the same success.

### Permissions

`DELETE /repos/...` needs `delete_repo` on a classic token, or `Administration: write` on a fine-grained one. Missing it returns 403, and no scope added afterwards fixes a token issued without that permission; it has to be reissued.

Changing visibility to private breaks every anonymous reader: forks, badges, integrations, documentation links. Say it before the call, once.

Changing the default branch silently changes what every clone and every workflow triggers on. List the branch that becomes default and the one that stops being it.

Ordinary settings — description, homepage, topics, merge options — go through `PATCH /repos/<owner>/<repo>`. They are reversible, though a changed merge policy alters what every later pull request can do. Visibility and default branch stay under this section; the rest is an ordinary write with a report at the end.

The permission each field needs is on the fine-grained list at https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens. `Administration: write` covers the widest set, including deletion.

## Step 11 — When an action fails

Read the status code first; the body is often generic.

- `401` — the credential is absent, malformed, revoked or expired. This belongs to `github-onboard`; reissuing is the fix, retrying is not.
- `403` `Resource not accessible by personal access token` — the token type cannot do this. Change type, do not add scopes.
- `403` with `X-GitHub-SSO` — the organization has not authorized this token. The user clears it on the organization's SSO page; nothing on this side changes it.
- `403` with `rate limit` in the body — read `X-RateLimit-Reset`, wait, retry once. Secondary limits send the same code without that header and want a pause, not a retry loop.
- `404` on an object that exists in a browser — the credential lacks access. GitHub answers 404 instead of 403 on purpose. Recheck repository selection before telling anyone the object is gone.
- `409` on a contents write — `sha` missing for an update, or stale because the file changed since you read it. Re-read and retry once.
- `422` on a ref, tree or commit — malformed input: `ref` without `refs/`, a tree entry without `mode`, a `base_tree` that is a commit SHA, or a parent that is not a commit.
- `422` `Update is not a fast forward` on a ref move — the branch moved after you read its head. Re-read the head, rebuild the tree and the commit on top of it, and move again. Setting `force: true` skips the check and drops whatever arrived in between.
- `5xx` — server side. Four attempts with backoff, then report the failure with the request that produced it.

A timeout is not a failed request. The work may have completed after the connection dropped, and repeating it then builds a second commit on a parent that is no longer the head. Before repeating any write, read what the write was meant to change — `GET /git/ref/heads/<branch>`, compared against the head you started from. If it moved, the write landed. Report it as done and do not retry.

After two failures of the same call, stop varying parameters. Read the body once and match it against this list.

## Step 12 — Report

After any write, three lines:

1. What changed — object, field, old value, new value.
2. How it was verified — the read that came back and what it showed.
3. Next action — one command, one click, or nothing left.

A bare "done" is indistinguishable from an attempted change that failed silently, for whoever reads this session later.

## Credential hygiene

Do not echo a token, print its length, place it in a clone URL, or commit it. Pass it through an environment variable the shell already holds.

If a token reaches a shared log, a chat transcript, or a committed file, say so once and recommend rotation.
