---
name: github-operate
description: Does the work once the agent can already read a repository. Use when the agent needs to list repositories, read or change files, create branches, commit, open or merge a pull request, file or close an issue, inspect or trigger a workflow run, cut a release, set topics or other repository settings, or delete a repository, branch, file or directory. Also trigger on 提交代码 / 建分支 / 开 PR / 提 issue / 看 CI / 发 release / 改仓库设置 / 加 topics / 删仓库 / gh CLI / api.github.com / Git Data API / blobs / trees / workflow_dispatch / Actions logs. Assumes a working credential; when there is none, use github-onboard first.
license: Apache-2.0
metadata:
  author: 88Lookingupatthestars88
  version: "1.5"
---

# GitHub Operations

Getting a credential is github-onboard's job. This file starts once one works.

## Step 0 — What you can call

Three routes reach the same API, and which one exists decides the shape of every command below. Check before writing one:

- MCP tools are loaded. Take tool names from the server rather than from this file, and use this route when a single tool call covers the whole job.
- A shell with `gh` installed. Confirm with `gh --version`.
- A shell without `gh`, where HTTPS works. A bearer header on a plain request does everything the other two do.

Every command below is written in the `gh` form. The other two routes take the same parameters in a different wrapper. MCP arguments mirror the endpoint's fields, so a tool asking for `path`, `content`, `message`, `sha` and `branch` is doing what Step 3 does, whatever name the server gave it. Raw HTTPS replaces `gh api <path>` with a request to `https://api.github.com/<path>` plus `Authorization: Bearer`, and replaces `-f k=v` with a JSON body field; `--jq` is a client-side filter with no server equivalent.

Pick one route and keep it for the session. Switching halfway means re-reading the object through the new route before continuing. An OAuth credential living inside an MCP client never reaches a shell, so a shell-only job with no token sends you back to github-onboard.

The `sh` fences assume a POSIX shell. The wrapper adapts; the parameters do not.

- `base64 -w0` is GNU. macOS `base64` wraps by default. `openssl base64 -A` works on both, and on Windows where OpenSSL is installed.
- Heredocs do not exist in `cmd.exe` or PowerShell. Write the JSON to a file and pass its path to `--input`.
- `jq` may be absent where `gh` is present. `gh --jq` carries its own filter engine, so nothing here calls `jq` directly.
- `-f` turns every value into a string, `-F` keeps booleans and numbers typed. `-f draft=false` arrives as the string `false`, which a boolean field rejects. Use `-F` for booleans and numbers, or send the whole body through `--input`.

## Step 1 — Who you are acting as

Two reads, once per session:

```sh
gh api user --jq .login
gh api repos/<owner>/<repo> --jq '{perm: .permissions, default: .default_branch, private: .private}'
```

`permissions` reports `admin`, `maintain`, `push`, `triage` and `pull` as booleans. Read them rather than probing with a write, because a later 403 is answered by one of these values. Write access and repository visibility are separate: `push: true` on a public repository still means every read is anonymous unless the token travels with it.

## Step 2 — Reading

```sh
gh repo list --limit 30                     # repositories the token can see
gh api 'repos/<owner>/<repo>/git/trees/<branch>?recursive=1' --jq '.tree[].path'
gh api repos/<owner>/<repo>/contents/<path> --jq .content | base64 -d
gh api repos/<owner>/<repo>/contents/<path> -H 'Accept: application/vnd.github.raw'
```

A whole directory tree is one `git/trees?recursive=1` call, and `truncated: true` means it hit a limit: drop `recursive` and walk directory by directory. `contents` in JSON form returns base64 and stops at 1 MB, while the raw media type reaches 100 MB; past that, fetch through `git/blobs/<sha>` or clone. `raw.githubusercontent.com` serves cached content, so a file pushed seconds ago can return the previous version — read through the API with `?ref=<sha>` when freshness matters.

Base64 from `contents` arrives wrapped in newlines. `base64 -d` accepts them, macOS spells the flag `-D`, and `openssl base64 -d -A` works wherever OpenSSL exists. A commit's file list without the diff comes from `gh api repos/<owner>/<repo>/commits/<sha> --jq '.files[] | {filename, status, additions, deletions}'`.

`gh api` returns one page of 30 and follows nothing. `--paginate` walks the `Link` header and prints every page, and `per_page=100` cuts the number of calls. A count taken from one page is a page size; issues, pull requests and workflow runs all need this before anything is counted or deleted.

## Step 3 — Changing one file

```sh
# new file: send no sha field
gh api -X PUT repos/<owner>/<repo>/contents/<path> \
  -f message='<what changed and why>' \
  -f content="$(base64 -w0 <local-file>)" \
  -f branch='<branch>'

# existing file: read the sha, then send it
SHA=$(gh api repos/<owner>/<repo>/contents/<path>?ref=<branch> --jq .sha)
gh api -X PUT repos/<owner>/<repo>/contents/<path> \
  -f message='<what changed and why>' \
  -f content="$(base64 -w0 <local-file>)" \
  -f sha="$SHA" \
  -f branch='<branch>'
```

The read decides which block applies. `GET /contents/<path>?ref=<branch>` answering 404 means the file does not exist, so the PUT carries no `sha` field at all; answering 200 means it does, and the `sha` comes from that same response. An empty `sha=""` is a field with a bad value, and returns 422.

Pass `branch` every time. Without it the write lands on the default branch, which is rarely what a session about to open a pull request wants.

The `message` becomes permanent history, so one line saying what changed and why. `update` and `fix` leave the next reader nothing to work with.

Content is base64 without line breaks; the encoder table in Step 0 covers the platform differences. Past about 1 MB the base64 string exceeds the argument-list limit, so write the JSON to a file and pass `--input -`, which substitutes the same way.

A path holding spaces, `#`, `?` or non-ASCII characters has to be percent-encoded inside the URL. A space sent raw becomes a different path, and the write then lands on the wrong file or returns 404.

A `.md` file is rendered wherever the repository shows it, so it gets checked before the write. See *Markdown bound for a repository*.

## Step 4 — Changing several files at once

Three or more files justify this route, and so does any change where applying half of it leaves the repository broken. One or two files are cheaper through Step 3. The shape is N blob writes, one tree, one commit and one ref move, after two reads.

Arrays of objects do not survive `-f` flags, so the JSON travels through `--input`. The heredocs below are unquoted on purpose, letting the shell substitute the variables; base64 uses only `A-Z a-z 0-9 + / =`, so nothing in the payload collides with shell expansion.

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

A tree entry carrying `"sha": null` deletes that path. A path left out of `tree` stays as it was, because the new tree is `base_tree` plus the entries listed.

Drop `base_tree` and the call means the opposite: the tree becomes exactly what you listed, and every other path in the repository is deleted. The call still answers 201, and the response lists only the entries you sent, so nothing in it marks the paths that disappeared.

Send `base_tree` on every call meant to modify an existing repository, then count the result: `GET /git/trees/<new>?recursive=1` should show the previous blob count plus the paths you added, minus the paths you deleted. A count taken from the request you sent only repeats what you asked for.

When one change has to move both a branch and a tag, move the branch first and confirm it before touching the tag. A tag moved first stays on the new commit even if the branch move is rejected, so it ends up pointing at a commit no branch reaches, and `PATCH /git/refs/tags/<tag>` takes `force: true` so nothing objects.

`force: false` makes step 4 fail when someone pushed first. With `true` the move overwrites their commit and the API still reports success.

`git/ref/heads/<branch>` returns a commit SHA, and feeding that to `base_tree` returns 422; the tree SHA comes from the second read.

Deleting files this way is destructive, so Step 10 applies: list every path that goes away and wait for a named confirmation.

## Step 5 — Branches

```sh
HEAD=$(gh api repos/<owner>/<repo>/git/ref/heads/<base> --jq .object.sha)
gh api -X POST repos/<owner>/<repo>/git/refs -f ref='refs/heads/<new>' -f sha="$HEAD"
```

`ref` needs the full `refs/heads/` prefix; `heads/<new>` returns 422.

A repository with no commits has no default branch and no head SHA. There the first commit is Step 3 with `branch` set to the intended default branch name and no `sha`, and GitHub creates the branch along with it.

Deleting a branch is Step 10. `PATCH /git/refs` with `force: true` and `git push --force` rewrite history that others may already hold, and both count as destructive there.

## Step 6 — Pull requests

```sh
gh pr create --repo <owner>/<repo> --base <base> --head <branch> --title '<...>' --body '<...>'
gh pr view <n> --repo <owner>/<repo> --json state,mergeable,mergeStateStatus,files
gh pr merge <n> --repo <owner>/<repo> --squash
```

What each merge method leaves behind:

- merge keeps every commit and adds a merge commit, so the base branch shows the branch topology.
- squash collapses to a single commit, and the original commits stop being reachable from the base branch.
- rebase replays the commits onto the base with new SHAs, and anyone holding the old ones now has a divergent branch.

`--delete-branch` deletes the head branch as part of the merge, which makes it a deletion and therefore Step 10. Run the merge without the flag unless the user already agreed to losing the branch, then delete it as its own confirmed item.

Without the flag the head branch stays. Deleting a branch that still holds unmerged commits drops those commits from the branch list; they stay reachable by SHA through the API until garbage collection runs, which has no user-facing trigger.

`mergeable: false` with `mergeStateStatus: dirty` means conflicts against the base. Resolution happens on the head branch, since the API has no server-side conflict resolution.

## Step 7 — Issues

```sh
gh issue create --repo <owner>/<repo> --title '<...>' --body '<...>'
gh issue comment <n> --repo <owner>/<repo> --body '<...>'
gh issue close <n> --repo <owner>/<repo> --reason completed
```

Issues need `issues: write` on a fine-grained token. A pull request conversation accepts the same comment endpoint, but a token scoped only to issues fails there, which reads like a permissions bug and is not one.

Closing is reversible through `gh issue reopen`. REST has no endpoint that deletes an issue, so do not promise one.

## Step 8 — Workflows

```sh
gh run list --repo <owner>/<repo> --limit 10 --json databaseId,status,conclusion,headBranch,event
gh run view <id> --repo <owner>/<repo> --log-failed
gh run watch <id> --repo <owner>/<repo>
gh workflow run <file>.yml --repo <owner>/<repo> --ref <branch>
```

`conclusion` reads `success`, `failure`, `cancelled`, `skipped`, or `null` while the run is still going, so `null` with `status: in_progress` is not a failure.

Logs come back as a zip behind a redirect. `gh run view --log` and `--log-failed` unpack them; a raw HTTP client has to follow the 302 and unzip, because the archive read as text is binary noise rather than an error message.

`workflow_dispatch` needs the workflow file to exist on the default branch even when `--ref` names another one. A 404 on dispatch usually means the workflow lives only on a feature branch, so merge it first or send the event another way.

Triggering needs `actions: write` and reading logs needs `actions: read`. A push authenticated with a PAT starts workflows; a push made with a workflow's own `GITHUB_TOKEN` does not, deliberately, so that a workflow cannot trigger itself in a loop. A CI chain stopping after the first job is behaving as designed.

## Step 9 — Releases

```sh
gh release create <tag> --repo <owner>/<repo> --target <sha> --title '<...>' --notes '<...>'
gh release view <tag> --repo <owner>/<repo> --json tagName,targetCommitish,isDraft,assets
```

A tag that does not exist is created at `--target`. A tag that does exist is reused, so a mistyped tag publishes against whatever commit already carries it. Read the release back and compare `targetCommitish` against the commit you meant. `--draft` keeps the release invisible until someone publishes it, and `--prerelease` marks it without hiding it.

What goes in the notes is a separate question with its own rules. See *The words that ship*.

A tag fixes two things. It sets the commit the release page describes, and it sets what the automatically generated Source code archives contain, because GitHub builds those from the tag when someone downloads them rather than from the branch. A tag sitting behind `main` therefore serves old code as the download while the hand-uploaded assets hold the new files, and the release page carries both under one version number. Read `GET /git/refs/tags/<tag>` and `GET /git/refs/heads/main` before publishing. When the two SHAs differ, cut a new tag on the current head; moving the old one keeps the assets in place and leaves the earlier archives holding files nobody can reach.

Attached files are objects of their own, not part of the tree, so moving the tag leaves them where they were. `gh release upload <tag> <file> --repo <owner>/<repo>` adds one, the same name twice fails 422 `already_exists`, and `--clobber` replaces it. Over REST the host is `uploads.github.com` rather than `api.github.com`: `POST /repos/<owner>/<repo>/releases/<id>/assets?name=<name>` with `--data-binary @<file>` and a `Content-Type` header. Replacing one means `DELETE /releases/assets/<id>` first, which returns 204 with no body.

Deleting a release leaves the tag behind. Removing both takes two calls, and both belong to Step 10.

## Step 9.5 — Making the repository findable

A repository nobody can search for is a repository nobody installs. None of these fields sits in a commit, so they are set after the files land.

| Field | Endpoint | Permission |
|---|---|---|
| description, homepage | `PATCH /repos/<owner>/<repo>` | Administration: write |
| has_discussions, has_issues, has_wiki, has_projects | `PATCH /repos/<owner>/<repo>` | Administration: write |
| default_branch | `PATCH /repos/<owner>/<repo>` | Administration: write |
| topics | `PUT /repos/<owner>/<repo>/topics` | Administration: write |
| social preview image | none | none |

Topics are the one field that does not go through `PATCH /repos/<owner>/<repo>`. That request accepts the field, answers 200 and returns the old list unchanged, so a caller trusting the status code reports success over a write that never happened. Use `PUT /repos/<owner>/<repo>/topics` with `{"names": [...]}`, which replaces the entire list.

Three rules come back as 422 with `must start with a lowercase letter or number, consist of 50 characters or less, and can include hyphens`:

- At most 20 topics on a repository.
- Lowercase letters, digits and hyphens only. `Agent Skills` is rejected.
- The order you send is not kept, and no topic weighs more than another. The response comes back alphabetised.

Read the field back and compare it against what you sent, because the wrong endpoint answers 200 while changing nothing.

Which topics to pick is a question about which list a person scrolls, not about keywords. A topic holding a hundred thousand repositories buries a new entry on page forty; one holding six thousand leaves it on the first page for months. Two accurate small topics beat six broad ones, and search ranks on the description and the name far more than on topics.

The social preview image has no REST endpoint and no GraphQL mutation. `GET /repos/<owner>/<repo>` omits the field, GraphQL exposes it read-only as `openGraphImageUrl`, and `gh repo edit` carries no flag for it. The user opens `https://github.com/<owner>/<repo>/settings`, finds Social preview and picks a file; say that rather than scripting around it. The image wants 1280×640, PNG or JPEG, under 1 MB, and it is what every link to the repository renders as in a chat client or on a social timeline.

An issue template lives in `.github/ISSUE_TEMPLATE/` and is a file in the tree, so Step 3 or Step 4 writes it. Two files do the work: one or more `*.yml` forms, and a `config.yml` whose `blank_issues_enabled: false` removes the empty option and whose `contact_links` sends questions somewhere else. The community profile endpoint reports `issue_template` as missing when the directory holds only a form, so `config.yml` is what closes that gap.

A public repository is ready when the files sit on the default branch, the release tag sits on that same commit, the assets read back with the expected bytes, the topics are set, the description is filled, an issue template sits beside a `config.yml`, and Discussions is either configured or deliberately off. Every item is confirmed by reading it back, because a 200 is not evidence that a field changed.

## Step 10 — Deleting and other irreversible changes

Covers deleting a repository, a branch, a file, a directory or a release; changing visibility; changing the default branch; force-pushing; transferring ownership.

### The rule

A user asking for a deletion is asking for a plan, and the plan runs only after a confirmation arrives, only for the items that confirmation names.

### Sequence

1. Read the current state of everything in scope and state it back in one short block: repository, branch, path, size, last commit, visibility.
2. Produce a numbered list of what would be removed, one line per item. Include every item the phrasing could reach rather than the first one that came to mind, and mark the ones you believe are out of scope so the user can rule them in.
3. State what the list does not cover: other branches, forks, local copies, open pull requests, releases, tags, CI artifacts.
4. State where a copy exists, and say outright when none does.
5. Ask the user to name the numbers. "All" still needs the list acknowledged once, since the list is what will run.
6. Execute one item at a time, reading each object back afterwards and reporting the status code with what the API now returns.
7. Stop after the named items, even when the rest sat in the same list.

Words that count as confirmation: 确认, 删, 做吧, 可以删, yes, go ahead, delete it. Words that do not: ok, 好的, 嗯, a question about something else, silence. A confirmation for item 2 covers item 2 alone.

Shape of the message, filled in from the reads above:

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

- The remote repository. `DELETE /repos/<owner>/<repo>` removes the repository, its issues, its pull requests and its releases, and every fork loses its parent link.
- Something inside the remote repository: a branch (`DELETE /git/refs/heads/<branch>`), a file or directory (Step 4 with `sha: null`, one commit), a release (`DELETE /releases/<id>`), a tag (`DELETE /git/refs/tags/<tag>`).
- A path on the local machine. `rm -rf` on a workspace directory, a clone, a cache. Nothing on GitHub changes, and GitHub cannot restore it.

A request mixing two of these — "delete the test repo and the local clone" — splits into separate numbered items and separate confirmations, because the two failures are not the same size and the second one is unrecoverable.

Before running any local `rm -rf`, list the absolute path and the file count, and confirm the path resolves where you think it does. A relative path matching nothing and a relative path matching the parent both print the same success.

### Permissions

`DELETE /repos/...` needs `delete_repo` on a classic token, or `Administration: write` on a fine-grained one, and missing it returns 403. A fine-grained token can be edited to add the permission and keeps its value; a classic token cannot gain a scope after it is issued, so it has to be regenerated.

Changing visibility to private breaks every anonymous reader: forks, badges, integrations, documentation links. Say it before the call, once. Changing the default branch changes what every clone checks out and what every workflow triggers on, so list the branch that becomes default and the one that stops being it.

Ordinary settings go through the endpoints in Step 9.5: description, homepage and the booleans through `PATCH /repos/<owner>/<repo>`, topics through `PUT /repos/<owner>/<repo>/topics`. Both are reversible, though a changed merge policy alters what every later pull request can do.

The permission each field needs is on the fine-grained list at https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens. `Administration: write` covers the widest set, including deletion, so setting a topic and deleting the repository are the same permission.

## Step 11 — When an action fails

Read the status code first, since the body is often generic. Then read `x-accepted-github-permissions` off the response: every REST response carries it, naming the permission and level the endpoint checks — `contents=write`, `administration=read`, `secrets=read` — which answers the permission question without a guess. A header naming a permission the token already holds means the permission is not the problem, so check the repository selection, then whether an organization has approved the token.

- `401` — the credential is absent, malformed, revoked or expired. This belongs to github-onboard; reissuing is the fix and retrying is not.
- `403` `Resource not accessible by personal access token` — a fine-grained token was refused. The header names the permission; add it, or raise its level. Only when no fine-grained permission exists for the operation does the token type have to change, and github-onboard 3.4 lists those six cases.
- `403` with `X-GitHub-SSO` — the organization has not authorized this token. The user clears it on the organization's SSO page, and nothing on this side changes it.
- `403` carrying `rate limit` in the body — read `X-RateLimit-Reset`, wait, retry once. Secondary limits send the same code without that header and want a pause rather than a retry loop.
- `404` on an object that exists in a browser — the credential lacks access, because GitHub answers 404 instead of 403 on purpose. Recheck the repository selection before telling anyone the object is gone.
- `409` on a contents write — `sha` missing for an update, or stale because the file changed since the read. Re-read and retry once.
- `422` on a ref, tree or commit — malformed input: `ref` without `refs/`, a tree entry without `mode`, a `base_tree` that is a commit SHA, or a parent that is not a commit.
- `422` `Update is not a fast forward` on a ref move — the branch moved after its head was read. Re-read the head, rebuild the tree and the commit on top of it, and move again; `force: true` skips the check and drops whatever arrived in between.
- `5xx` — server side. Four attempts with backoff, then report the failure along with the request that produced it.

A timeout is not a failed request. The work may have completed after the connection dropped, and repeating it then builds a second commit on a parent that is no longer the head. Before repeating any write, read what the write was meant to change — `GET /git/ref/heads/<branch>` against the head you started from — and when it moved, report the write as done without retrying.

After two failures of the same call, stop varying parameters, read the body once and match it against this list.

## Step 12 — Report

After any write, three lines:

1. What changed: object, field, old value, new value.
2. How it was verified: the read that came back and what it showed.
3. Next action: one command, one click, or nothing left.

A bare "done" leaves the reader unable to tell whether anything happened.

## The words that ship

Four things leave the repository carrying text: the commit message, the release notes, the readme, and the body of an issue or a pull request. Each one is read by someone you are not standing next to.

### Commit message

One line, under seventy characters, saying what changed. The diff already shows where, so the line carries the reason. `Update files` and `Fix stuff` restate the diff and add nothing.

### Release notes

The reader is deciding whether to download this version. They have not read the diff and will not read it.

- Say what changed for them, in their words. "Topics can be set now" reaches them, and `PUT /repos/<owner>/<repo>/topics` does not.
- Say what they have to do: download again, copy the folders again, or nothing. When the answer is nothing, write that rather than leaving it to be inferred.
- One paragraph. A release that only repackages an earlier one gets one sentence and says so.
- Field names, endpoint names, permission names and the reason a design was chosen stay out. Those belong in the commit message and in the conversation with the person who asked for the change.
- A change the user cannot see does not get a line in the release notes. Report it in the conversation instead.

No inline code in a release body. GitHub offers a machine translation of the page, and in that translation every code span is pushed to the end of its paragraph, so the sentence arrives full of holes with a row of boxes underneath. Write the body in the language its readers use, and spell a value out rather than wrapping it.

### Readme

Two readers arrive thirty seconds apart. The first is deciding what this is and whether it runs on their machine; the second has decided and wants exact steps. The opening paragraph serves the first and the steps sit below it. A readme that opens with architecture loses the first reader and never reaches the second.

### Issue and pull request bodies

Write what happened, what was expected, and the smallest input that shows both. Paste output rather than retyping it, because the exact characters are the diagnosis.

### What reads as machine-written

Eight habits, each removable without losing information:

1. A sentence explaining why the sentence before it was written. Keep the fact, drop the commentary.
2. A balanced pair closing a paragraph. Say the fact once.
3. An opener announcing the point, such as `It is worth noting` or `A word of caution`. Start with the fact.
4. Two words for one thing in the same paragraph.
5. List items starting with a bare noun, leaving the sentence without a subject.
6. Text describing its own existence: `This version rewrites the readme`, `These two skills are for an agent`. Describe the effect instead.
7. Three adjectives in a row.
8. In Chinese, a closing cadence that reverses the sense (`不是…而是…`) and pairs kept only because they balance.

What survives the cut should have a subject, a verb and a value.

## Markdown bound for a repository

A `.md` file is rendered wherever GitHub shows it, and the renderer reads `<` as the start of an HTML tag. A path written as `<your skills dir>/x` leaves a tag the renderer drops, and the code spans sharing that paragraph can be pulled to its end. On a phone that reads as a broken sentence followed by a row of loose boxes.

Four checks, one command each:

```sh
# 1. no < outside a fenced block, after inline code is stripped — this is the one that broke
for f in *.md; do awk '/^[[:space:]]*```/{b=!b; next} !b' "$f" | sed 's/`[^`]*`//g' | grep -n '<' | sed "s|^|$f:|"; done

# 2. fences paired; the count has to be even
grep -cE '^[[:space:]]*```' *.md

# 3. every link closes
a=$(grep -o '](' *.md | wc -l); b=$(grep -oE '\]\([^)]*\)' *.md | wc -l); echo "$a $b"

# 4. a code span alone on its line is a sentence missing its subject
awk '/^[[:space:]]*```/{f=!f; next} !f && /^[[:space:]]*`[^`]+`[[:space:]]*$/{print FILENAME":"FNR": "$0}' *.md
```

Every one of those reads a fence as a line of three backticks at column zero. A fenced block nested under a numbered step is indented by three spaces, and Markdown accepts that, so a block starting at column zero and closing at column three balances nothing: an unclosed indented block counts as zero fences, and the parity check passes while the page renders wrong. The `[[:space:]]*` in each pattern is what makes the nested case count.

Write a path as `your skills dir/github-onboard/SKILL.md`, in a fenced block or inline, and leave the brackets off unless the sentence is about the brackets themselves.

A second failure belongs to the same family. A paragraph of Chinese carrying code spans can render with every span pushed to the paragraph's end, which reads as a sentence missing its nouns followed by a row of boxes. The server-side renderer places them correctly and the HTML shows nothing wrong, so the browser lays out the line afterwards; on a device that does this, replace the spans with bold text or move the value into a fenced block. Read the page on the device that will display it.

Two things the commands do not catch. A list whose items all begin with a noun (`Reads:`, `Writes:`, `Deletes:`) has no subject, and a reader landing in the middle of it has no sentence to parse, so give every item a subject and a verb. And a byte count quoted beside a download link goes stale the moment that file changes: run `wc -c`, paste the number, and recheck it in the same commit that touches the file.

`python3 -m markdown file.md` prints the HTML, after `pip install markdown` on a machine that lacks it. A paragraph holding several `<code>` runs in a row is the failure this section is about, and the HTML shows it before a phone does.

## Credential hygiene

Do not echo a token, print its length, place it in a clone URL, or commit it. Pass it through an environment variable the shell already holds.

If a token reaches a shared log, a chat transcript or a committed file, say so once and recommend rotation.
