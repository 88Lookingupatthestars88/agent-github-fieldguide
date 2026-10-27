---
name: github-operate
description: Operates on a GitHub repository once a credential works. Covers reading files and trees, commits, branches, pull requests, issues, workflow runs, releases, repository settings, the community profile, security audit, and deletion. Use for any read or write against api.github.com or the gh CLI. 提交 / 建分支 / 开 PR / 提 issue / 看 CI / 发 release / 改仓库设置 / 健康分 / 审核 / 删仓库.
license: Apache-2.0
metadata:
  author: 88Lookingupatthestars88
  version: "1.2.0"
  source: https://github.com/88Lookingupatthestars88/agent-github-fieldguide
---

# GitHub Operations

A working credential is assumed. When there is none, github-onboard comes first. Authentication errors stay there; this file starts at permissions and objects.

## Step 0 — What you can call

Three routes reach the same API, and which one exists decides the shape of every command below.

- MCP tools are loaded. Take tool names from the server, and use this route when one call covers the job.
- A shell with `gh`. Confirm with `gh --version`.
- A shell without `gh`, where HTTPS works. A bearer header does everything the other two do.

Every command below is written in the `gh` form. Raw HTTPS replaces `gh api <path>` with a request to `https://api.github.com/<path>` plus `Authorization: Bearer`, and replaces `-f k=v` with a JSON body field. `--jq` is a client-side filter with no server equivalent.

Pick one route and keep it for the session. Switching halfway means re-reading the object through the new route.

- `base64 -w0` is GNU; macOS wraps, and `openssl base64 -A` works on both.
- No heredocs in `cmd.exe` or PowerShell. Write the JSON to a file and pass `--input`.
- `gh --jq` carries its own filter, so nothing here calls `jq`.
- `-f` makes every value a string and `-F` keeps booleans typed. `-f draft=false` arrives as a string and a boolean field rejects it.

## Step 1 — Who you are acting as

```sh
gh api user --jq .login
gh api repos/<owner>/<repo> --jq '{perm: .permissions, default: .default_branch, private: .private}'
```

`permissions` reports `admin`, `maintain`, `push`, `triage` and `pull` as booleans. Read them rather than probing with a write. Write access and visibility are separate: `push: true` on a public repository still means every read is anonymous unless the token travels with it.

## Step 2 — Reading

```sh
gh repo list --limit 30
gh api 'repos/<owner>/<repo>/git/trees/<branch>?recursive=1' --jq '.tree[].path'
gh api repos/<owner>/<repo>/contents/<path> -H 'Accept: application/vnd.github.raw'
gh api repos/<owner>/<repo>/contents/<path> --jq .content | base64 -d
```

A whole tree is one `git/trees?recursive=1` call, and `truncated: true` means it hit a limit: drop `recursive` and walk directory by directory. `contents` in JSON form returns base64 and stops at 1 MB; the raw media type reaches 100 MB; past that, use `git/blobs/<sha>`. `raw.githubusercontent.com` serves cached content, so read through the API with `?ref=<sha>` when freshness matters.

`gh api` returns one page of 30 and follows nothing. `--paginate` walks the `Link` header and `per_page=100` cuts the calls. A count from one page is a page size, which matters before counting or deleting anything.

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

A 404 on the read means the file does not exist, so the PUT carries no `sha` at all. An empty `sha=""` is a bad value and returns 422. Pass `branch` every time, or the write lands on the default branch.

The `message` becomes permanent history, so one line saying what changed and why.

Past about 1 MB the base64 string exceeds the argument-list limit: write the JSON to a file and pass `--input -`. A path holding spaces, `#`, `?` or non-ASCII characters has to be percent-encoded inside the URL, or the write lands on a different path.

A `.md` file gets checked before the write. See *Markdown bound for a repository*.

## Step 4 — Changing several files at once

Three or more files justify this route, and so does any change where applying half of it leaves the repository broken. The shape is N blob writes, one tree, one commit and one ref move, after two reads.

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

The heredocs are unquoted on purpose, letting the shell substitute; base64 uses only `A-Z a-z 0-9 + / =`, so nothing collides with expansion. A tree entry carrying `"sha": null` deletes that path, and a path left out of `tree` stays as it was.

Drop `base_tree` and the call means the opposite: the tree becomes exactly what you listed and every other path is deleted. It still answers 201, and the response lists only what you sent, so nothing in it marks the paths that disappeared. Send `base_tree` on every call meant to modify an existing repository, then verify by path set:

```sh
gh api 'repos/<owner>/<repo>/git/trees/<new-tree>?recursive=1' --jq '.tree[]|select(.type=="blob")|.path' | sort > got.txt
```

Compare that against the previous blob paths plus the ones you added, minus the ones you removed. A count taken from the request only repeats what you asked for.

When one change moves both a branch and a tag, move the branch first and confirm it. A tag moved first stays on the new commit even when the branch move is rejected, and ends up on a commit no branch reaches. `git/ref/heads/<branch>` returns a commit SHA, and feeding that to `base_tree` returns 422; the tree SHA comes from the second read.

## Step 5 — Branches

```sh
HEAD=$(gh api repos/<owner>/<repo>/git/ref/heads/<base> --jq .object.sha)
gh api -X POST repos/<owner>/<repo>/git/refs -f ref='refs/heads/<new>' -f sha="$HEAD"
```

`ref` needs the full `refs/heads/` prefix. A repository with no commits has no head SHA, so the first commit is Step 3 with `branch` set to the intended default name and no `sha`.

## Step 6 — Pull requests

```sh
gh pr create --repo <owner>/<repo> --base <base> --head <branch> --title '<...>' --body '<...>'
gh pr view <n> --repo <owner>/<repo> --json state,mergeable,mergeStateStatus,files
gh pr merge <n> --repo <owner>/<repo> --squash
```

What each merge method leaves behind:

- merge keeps every commit and adds a merge commit.
- squash collapses to one commit, and the originals stop being reachable from the base branch.
- rebase replays the commits with new SHAs, so anyone holding the old ones has a divergent branch.

`--delete-branch` makes the merge a deletion, so it belongs to Step 10. Without it the head branch stays.

`mergeable: false` with `mergeStateStatus: dirty` means conflicts against the base, and resolution happens on the head branch because the API has no server-side resolution.

## Step 7 — Issues

```sh
gh issue create --repo <owner>/<repo> --title '<...>' --body '<...>'
gh issue comment <n> --repo <owner>/<repo> --body '<...>'
gh issue close <n> --repo <owner>/<repo> --reason completed
```

Issues need `issues: write`. A pull request conversation accepts the same comment endpoint, but a token scoped only to issues fails there. Closing reverses through `gh issue reopen`. REST has no endpoint that deletes an issue.

## Step 8 — Workflows

```sh
gh run list --repo <owner>/<repo> --limit 10 --json databaseId,status,conclusion,headBranch,event
gh run view <id> --repo <owner>/<repo> --log-failed
gh workflow run <file>.yml --repo <owner>/<repo> --ref <branch>
```

`conclusion` reads `success`, `failure`, `cancelled`, `skipped`, or `null` while running, so `null` with `status: in_progress` is not a failure.

Logs arrive as a zip behind a redirect. `--log` and `--log-failed` unpack them; a raw HTTP client has to follow the 302 and unzip, because the archive read as text is binary noise.

`workflow_dispatch` needs the workflow file on the default branch even when `--ref` names another one. A 404 on dispatch means the workflow lives only on a feature branch.

Triggering needs `actions: write` and reading logs needs `actions: read`. A push authenticated with a token starts workflows; a push made with a workflow's own `GITHUB_TOKEN` does not, so a CI chain stopping after its first job is behaving as designed.

## Step 9 — Releases

```sh
gh release create <tag> --repo <owner>/<repo> --target <sha> --title '<...>' --notes '<...>'
gh release view <tag> --repo <owner>/<repo> --json tagName,targetCommitish,isDraft,assets
```

A tag that does not exist is created at `--target`. An existing tag is reused, so a mistyped tag publishes against whatever commit carries it. Read the release back and compare `targetCommitish`.

A tag fixes two things: the commit the release page describes, and what the generated source archives contain, because GitHub builds those from the tag on download rather than from the branch. Read `GET /git/refs/tags/<tag>` and `GET /git/refs/heads/main` before publishing. When they differ, cut a new tag on the current head; moving an old one leaves the earlier archives holding files nobody can reach.

Attached files are objects of their own, not part of the tree, so moving the tag leaves them in place. `gh release upload <tag> <file>` adds one, the same name twice fails 422 `already_exists`, and `--clobber` replaces it. Over REST the host is `uploads.github.com`: `POST /repos/<owner>/<repo>/releases/<id>/assets?name=<name>` with `--data-binary @<file>`. Replacing one means `DELETE /releases/assets/<id>` first, which returns 204.

### A new version does not replace the old one

Publishing 1.2 does not remove 1.1. A release is a download link someone may already have open, cited, or built into their own setup, and the page listing every version is what lets a reader pin one and choose the upgrade.

So a version bump is one call:

```sh
gh release create <new-tag> --repo <owner>/<repo> --target <sha> --title '<...>' --notes '<...>'
```

The old tag keeps its commit and its source archives keep building from it, which is correct: someone downloading 1.1 wants 1.1. A tag behind `main` is not a defect to fix by moving; it is what a released version is. Fixes go into a new tag, and the old one stays readable so the difference between them can be seen.

`latest` moves to the newest release by itself, and `/releases/latest/download/<file>` follows it, so a readme's download links survive a bump without an edit.

Removing a release, or a tag with it, is a deletion and needs a named confirmation under Step 10, even when the agent published that version minutes earlier and even when the replacement is byte-identical. Withdraw a version that leaks a credential or breaks machines; say which and why, then wait.

### One copy on disk

The working tree holds the current version and nothing else. A released version lives on its release page, where the tag, the source archives and the assets stay downloadable, so a directory beside the tree adds nothing and goes stale the first time the tree is edited. Two copies of one version number, one of them stale, is how a reader ends up comparing the wrong file.

Build the package in a scratch directory, upload it, verify the bytes, delete it. Reading an old version back means downloading it, not keeping a folder in reserve.

### Is this copy current

`metadata.version` above is the release this copy shipped with, the tag without its `v`, and `metadata.source` is where it came from. One call answers whether it is still the newest:

```sh
gh api repos/88Lookingupatthestars88/agent-github-fieldguide/releases/latest --jq '.tag_name, .published_at'
```

A tag above that number means this copy is older than what shipped. Report both numbers and let the user decide; replacing the file is their edit. The older release stays downloadable, so a pinned version keeps working.

## Step 9.5 — Repository settings

None of these fields sits in a commit, so they are set after the files land.

| Field | Endpoint | Permission |
|---|---|---|
| description, homepage | `PATCH /repos/<owner>/<repo>` | Administration: write |
| has_discussions, has_issues, has_wiki, has_projects | `PATCH /repos/<owner>/<repo>` | Administration: write |
| default_branch | `PATCH /repos/<owner>/<repo>` | Administration: write |
| topics | `PUT /repos/<owner>/<repo>/topics` | Administration: write |
| social preview image | no endpoint | none |

Topics are the field that does not go through `PATCH /repos/<owner>/<repo>`. That request accepts the field, answers 200 and returns the old list unchanged, so a caller trusting the status code reports success over a write that never happened. Use `PUT` with `{"names": [...]}`, which replaces the whole list.

Three rules come back as 422 with `must start with a lowercase letter or number, consist of 50 characters or less, and can include hyphens`:

- At most 20 topics.
- Lowercase letters, digits and hyphens only.
- Order is not kept. The response comes back alphabetised, and no topic weighs more than another.

Read the field back and compare, because the wrong endpoint answers 200 while changing nothing.

Pick topics by which list a person scrolls: one holding a hundred thousand repositories buries a new entry on page forty, one holding six thousand keeps it on the first page. Two accurate small topics beat six broad ones, and search ranks on the name and description far more than on topics.

The social preview image has no REST endpoint and no GraphQL mutation. `GET /repos/<owner>/<repo>` omits the field, GraphQL exposes `openGraphImageUrl` read-only, and `gh repo edit` carries no flag. The user opens `https://github.com/<owner>/<repo>/settings`, finds Social preview and picks a file. 1280×640, PNG or JPEG, under 1 MB.

An issue template is a file in the tree, so Step 3 or Step 4 writes it. Where it sits decides whether the community profile records one, which Step 9.6 covers.

## Step 9.6 — The community profile

`GET /repos/<owner>/<repo>/community/profile` backs the Community Standards panel. `health_percentage` is the number that panel shows, and each entry under `files` holds a URL or `null`. The `null` entries are the files still missing.

| Field | File that clears it |
|---|---|
| `readme` | `README.md` |
| `license` | `LICENSE` |
| `contributing` | `CONTRIBUTING.md` or `.github/CONTRIBUTING.md` |
| `code_of_conduct` | `CODE_OF_CONDUCT.md` or `.github/CODE_OF_CONDUCT.md` |
| `issue_template` | `.github/ISSUE_TEMPLATE.md` |
| `pull_request_template` | `.github/PULL_REQUEST_TEMPLATE.md` |
| `documentation` | a wiki page, or a Pages site |

GitHub's help page gives four items at twenty-five points each. The number that comes back does not divide by four, so `files` is the part to read.

`issue_template` takes one file at `.github/ISSUE_TEMPLATE.md`. A form at `.github/ISSUE_TEMPLATE/report.yml` with a `config.yml` beside it gives a working chooser and leaves that field `null`, so write both.

`documentation` takes a wiki page or a Pages site. `has_wiki: true` on its own leaves the field `null`.

```sh
gh api -X POST repos/<owner>/<repo>/pages -f 'source[branch]=main' -f 'source[path]=/'
```

Pages publishes the whole branch and Jekyll renders `README.md` as the index. `documentation` fills on the next read. `health_percentage` lags several minutes, so a read taken straight after still returns the old number.

A run ending in `Page build failed`, `duration: 0` and a 500 on the deployment had no content problem. GitHub's deployment service dropped it, and the next push runs it again.

The percentage counts files. A `CONTRIBUTING.md` with no build steps in it scores the same as one that has them.

## Step 9.7 — Auditing a repository

Run this before a repository goes public, and again whenever a token, a collaborator or a workflow changes. Several of these features answer 404 when they are off, so read the field rather than the status code.

1. **A credential in the tree or in history.** `GET /secret-scanning/alerts` answers on a public repository with no setup; a private one needs the permission and answers 403 without it. Where it is unavailable, walk `git/trees?recursive=1` and grep the blobs, then check history separately, because deleting a file leaves the blob reachable.
2. **The licence actually carried.** `GET /repos/<owner>/<repo>` → `.license.spdx_id`. `null` means no licence file exists however the readme reads, and no licence means nobody may copy it. Apache-2.0 with a `NOTICE` file means `NOTICE` travels with every redistribution.
3. **Whether workflows run code that can move.** `GET /actions/permissions` → `sha_pinning_required`. False by default, so `uses: owner/action@v4` executes whatever that tag points at on the day it runs. Only a full forty-character SHA is fixed, so pin third-party actions to one.
4. **What a workflow's token may do.** `GET /actions/permissions/workflow` → `default_workflow_permissions`. `read` is the floor; `write` lets any workflow push to any branch. A job needing more requests it in its own `permissions:` block.
5. **Whether the default branch survives a mistake.** `GET /branches/<branch>/protection` answering 404 means unprotected, and an unprotected branch takes force-pushes and deletions from any token holding write.
6. **Whether a stranger can report a vulnerability privately.** `GET /private-vulnerability-reporting` reads `{"enabled": false}` on a new repository, and a `PUT` with `{"enabled": true}` turns it on. A `SECURITY.md` naming an address does not.
7. **Whether advisories arrive.** `GET /vulnerability-alerts` answers 404 and `GET /dependabot/alerts` answers 403 when off. Alerts and security updates are separate switches, both off on a new repository: `PUT /vulnerability-alerts` for the first, `security_and_analysis.dependabot_security_updates` for the second. With no dependency file the pair is moot.
8. **What Pages serves.** `GET /pages` → `visibility`. A private repository with a public Pages site exposes its files whatever the repository setting says.
9. **What else reaches the repository.** `GET /keys` and `GET /hooks`. Both returning `[]` is normal; a deploy key or a webhook nobody remembers adding survives a token rotation.
10. **How far the token reaches.** No endpoint lists a token's scopes. `x-accepted-github-permissions` names what each call needed, so the union across a session is the set it used. Compare that against what the session was for: `administration: write` carries deleting the repository.

Three lines per finding: the field, its current value, and the one call that changes it. A decision belonging to the owner — which licence, whether the repository should be private — gets named as a decision rather than made.

## Step 10 — Deleting and irreversible changes

Covers deleting a repository, a branch, a file, a directory or a release; changing visibility; changing the default branch; force-pushing; transferring ownership.

### The rule

A request to delete is a request for a plan, and the plan runs only after a confirmation arrives, only for the items that confirmation names.

### Sequence

1. Read the current state of everything in scope and state it back: repository, branch, path, size, last commit, visibility.
2. Number every item that would be removed, one line each, including the ones you believe are out of scope so the user can rule them in.
3. Say what the list leaves out — other branches, forks, local copies, open pull requests, releases, tags, CI artifacts — and where a copy exists, or that none does.
4. Ask the user to name the numbers. "All" still needs the list acknowledged once.
5. Execute one at a time, reading each object back, and stop after the named items.

Words that count as confirmation: 确认, 删, 做吧, 可以删, yes, go ahead, delete it. Words that do not: ok, 好的, 嗯, a question about something else, silence. A confirmation for item 2 covers item 2 alone.

```
You asked to remove: <one line>
Now: <owner/repo>, <branch or path>, <size>, last commit <sha> <date>, <visibility>

Would be removed:
1. <object> — <what goes with it>
2. <object> — <what goes with it>
3. <object> — <out of scope, listed so you can rule it in>

Not in this list: <other branches, forks, local copies, releases, tags>
A copy exists at: <path> | none

Reply with the numbers to remove. Everything not named stays.
```

### Which scope the request means

- The remote repository. `DELETE /repos/<owner>/<repo>` removes issues, pull requests and releases with it, and every fork loses its parent link.
- Something inside it: a branch (`DELETE /git/refs/heads/<branch>`), a file or directory (Step 4 with `sha: null`), a release, a tag.
- A path on the local machine. Nothing on GitHub changes, and GitHub cannot restore it.

A request mixing two of these splits into separate numbered items and separate confirmations, because the two failures are not the same size.

Before any local `rm -rf`, list the absolute path and the file count, and confirm the path resolves where you think. A relative path matching nothing and one matching the parent both print the same success.

### Permissions

`DELETE /repos/...` needs `delete_repo` on a classic token, or `Administration: write` on a fine-grained one. A fine-grained token can be edited to add the permission and keeps its value; a classic token cannot gain a scope after issue and has to be regenerated.

Changing visibility to private breaks every anonymous reader: forks, badges, integrations, documentation links. Changing the default branch changes what every clone checks out and what every workflow triggers on.

Ordinary settings go through Step 9.5 and are reversible, though a changed merge policy alters what every later pull request can do.

## Step 11 — When an action fails

Read the status code first, since the body is often generic. Then read `x-accepted-github-permissions`, which every REST response carries: it names the permission and level the endpoint checks, such as `contents=write` or `administration=read`. A header naming a permission the token already holds means the permission is not the problem, so check the repository selection, then whether an organization has approved the token.

- `401` — the credential is absent, malformed, revoked or expired. github-onboard owns that; reissuing is the fix and retrying is not.
- `403 Resource not accessible by personal access token` — a fine-grained token was refused. The header names the permission; add it or raise its level. Only when no fine-grained permission exists for the operation does the token type change.
- `403` with `X-GitHub-SSO` — the organization has not authorized this token. The user clears it on the organization's SSO page.
- `403` carrying `rate limit` — read `X-RateLimit-Reset`, wait, retry once. Secondary limits send the same code without that header and want a pause.
- `404` on an object that exists in a browser — the credential lacks access, because GitHub answers 404 instead of 403 on purpose.
- `409` on a contents write — `sha` missing, or stale because the file changed since the read. Re-read and retry once.
- `422` on a ref, tree or commit — malformed input: `ref` without `refs/`, a tree entry without `mode`, a `base_tree` that is a commit SHA, a parent that is not a commit.
- `422 Update is not a fast forward` — the branch moved after its head was read. Re-read the head, rebuild on top of it, move again.
- `5xx` — server side. Four attempts with backoff, then report the failure with the request that produced it.

A timeout is not a failed request: the work may have completed after the connection dropped, and repeating it builds a second commit on a parent that is no longer the head. Before repeating any write, read what it was meant to change and, when it moved, report the write as done.

After two failures of the same call, stop varying parameters, read the body once and match it against this list.

## Step 12 — Report

After any write, three lines:

1. What changed: object, field, old value, new value.
2. How it was verified: the read that came back and what it showed.
3. Next action: one command, one click, or nothing left.

## The words that ship

Four things leave the repository carrying text: the commit message, the release notes, the readme, and the body of an issue or pull request.

**Commit message.** One line, under seventy characters, saying what changed. The diff already shows where, so the line carries the reason.

**Release notes.** The reader is deciding whether to download. Say what changed for them in their words, say what they have to do, and keep it to one paragraph. Field names, endpoint names and the reason a design was chosen stay out. A change the user cannot see gets no line here; report it in the conversation. No inline code in a release body, because GitHub's machine translation pushes every code span to the end of its paragraph and the sentence arrives full of holes.

**Readme.** Two readers arrive thirty seconds apart. The first is deciding what this is and whether it runs on their machine; the second wants exact steps. The opening paragraph serves the first, the steps sit below it.

**Issue and pull request bodies.** Write what happened, what was expected, and the smallest input that shows both. Paste output rather than retyping it.

### What reads as machine-written

Eight habits, each removable without losing information:

1. A sentence explaining why the sentence before it was written. Keep the fact, drop the commentary.
2. A balanced pair closing a paragraph. Say the fact once.
3. An opener announcing the point, such as `It is worth noting`. Start with the fact.
4. Two words for one thing in the same paragraph.
5. List items starting with a bare noun, leaving the sentence without a subject.
6. Text describing its own existence. Describe the effect instead.
7. Three adjectives in a row.
8. In Chinese, a closing cadence that reverses the sense (`不是…而是…`) and pairs kept only because they balance.

What survives the cut has a subject, a verb and a value.

## Markdown bound for a repository

The renderer reads `<` as the start of an HTML tag, so a path written as `<your skills dir>/x` leaves a tag it drops, and the code spans sharing that paragraph can be pulled to its end. On a phone that reads as a broken sentence followed by a row of loose boxes.

Four checks, one command each:

```sh
# 1. no < outside a fenced block, after inline code is stripped
for f in *.md; do awk '/^[[:space:]]*```/{b=!b; next} !b' "$f" | sed 's/`[^`]*`//g' | grep -n '<' | sed "s|^|$f:|"; done

# 2. fences paired; the count has to be even
grep -cE '^[[:space:]]*```' *.md

# 3. every link closes
a=$(grep -o '](' *.md | wc -l); b=$(grep -oE '\]\([^)]*\)' *.md | wc -l); echo "$a $b"

# 4. a code span alone on its line is a sentence missing its subject
awk '/^[[:space:]]*```/{f=!f; next} !f && /^[[:space:]]*`[^`]+`[[:space:]]*$/{print FILENAME":"FNR": "$0}' *.md
```

The `[[:space:]]*` in each pattern matters: a fenced block nested under a numbered step is indented three spaces, and a pattern anchored at column zero counts it as zero fences while the parity check still passes. Write a path in a fenced block or inline, and leave the brackets off unless the sentence is about the brackets.

A paragraph of Chinese carrying code spans can render with every span pushed to the paragraph's end, which reads as a sentence missing its nouns followed by a row of boxes. The server-side renderer places them correctly, so the browser lays out the line afterwards. On a device that does this, replace the spans with bold text or move the value into a fenced block, and read the page on the device that will display it.

Two things the commands miss. A list whose items all begin with a noun has no subject, so give every item a subject and a verb. A byte count beside a download link goes stale the moment the file changes: run `wc -c`, paste the number, and recheck it in the commit that touches the file.
