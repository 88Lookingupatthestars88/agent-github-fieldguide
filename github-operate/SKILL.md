---
name: github-operate
description: Operates on a GitHub repository once a credential works. Covers reading files and trees, commits, branches, pull requests, issues, workflow runs, releases, repository settings, the community profile, auditing what a repository exposes, taking an identity out of history, and deletion. Use for any read or write against api.github.com or the gh CLI. 提交 / 建分支 / 开 PR / 提 issue / 看 CI / 发 release / 改仓库设置 / 健康分 / 审核 / 查历史里的邮箱 / 删仓库.
license: Apache-2.0
metadata:
  author: 88Lookingupatthestars88
  version: "1.3.0"
  source: https://github.com/88Lookingupatthestars88/agent-github-fieldguide
---

# GitHub Operations

A working credential is assumed. Without one, github-onboard comes first. Authentication failures stay there; this file starts at permissions and objects.

## Step 0 — What you can call

Three routes reach the same API, and the one that exists decides the shape of every command below.

- MCP tools are loaded. Take names from the server, and use this route where one call covers the job.
- A shell with `gh`. Confirm with `gh --version`.
- A shell without `gh`, where HTTPS works. A bearer header covers what the other two do.

Commands below are written in the `gh` form. Raw HTTPS replaces `gh api <path>` with a request to `https://api.github.com/<path>` plus `Authorization: Bearer`, and `-f k=v` with a JSON body field. `--jq` is a client-side filter with no server equivalent. Pick one route and keep it for the session; switching halfway means reading the object again through the new route.

Four platform facts break a command that ignores them: `base64 -w0` is GNU and macOS wraps, while `openssl base64 -A` runs on both; `cmd.exe` and PowerShell have no heredocs, so the JSON goes to a file and travels as `--input`; `gh --jq` carries its own filter, so nothing here calls `jq`; `-f` turns every value into a string and `-F` keeps booleans typed, so `-f draft=false` arrives as a string that a boolean field rejects.

## Step 1 — Who you are acting as

```sh
gh api user --jq .login
gh api repos/<owner>/<repo> --jq '{perm: .permissions, default: .default_branch, private: .private}'
```

`permissions` reports `admin`, `maintain`, `push`, `triage` and `pull` as booleans. Read them rather than probing with a write. Write access and visibility are separate: `push: true` on a public repository still makes every read anonymous unless the token travels with it.

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

A 404 on the read means the file is absent, so the PUT carries no `sha` at all. An empty `sha=""` is a bad value and returns 422, and leaving `sha` off a file that exists returns 422 with `"sha" wasn't supplied`. Pass `branch` every time, or the write lands on the default branch.

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

Compare that against the previous blob paths, plus the ones you added, minus the ones you removed; a count taken from the request only repeats what you asked for.

`git/ref/heads/<branch>` returns a commit SHA, and feeding that to `base_tree` returns 422; the tree SHA comes from the second read. When one change moves both a branch and a tag, move the branch first: a tag moved first stays on the new commit even when the branch move is rejected, and ends up on a commit no branch reaches.

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

A tag names the commit the release page describes, and GitHub builds the source archives from the tag at download time, not from the branch. Read `GET /git/refs/tags/<tag>` and `GET /git/refs/heads/main` before publishing. When they differ, cut a new tag on the current head. Moving an old tag rewrites what its release page describes and what its source archives contain, so a download labelled 1.1 arrives holding newer files. A tag that does not exist is created at `--target`. An existing tag is reused, so a mistyped tag publishes against whatever commit carries it; read `targetCommitish` back.

Attached files are objects of their own and survive a tag move. `gh release upload <tag> <file>` adds one; the same name twice fails 422 `already_exists`, and `--clobber` replaces it. Over REST the host is `uploads.github.com`: `POST /repos/<owner>/<repo>/releases/<id>/assets?name=<name>` with `--data-binary @<file>`. Replacing one means `DELETE /releases/assets/<id>` first, which answers 204.

Publishing 1.3 does not remove 1.2. A release is a link someone may already have open or cited, and the page listing every version is how a reader pins one. `/releases/latest/download/<file>` follows the newest release, so download links in a readme survive a bump without an edit. Deleting a release takes its download links out of service, which makes it a Step 10 item with a numbered list and a named confirmation, even when the agent published the version minutes earlier.

The working tree holds the current version and nothing else. A released version stays downloadable from its release page, so a directory beside the tree goes stale the first time the tree is edited. Build the package in a scratch directory, upload it, verify the bytes, delete it.

`metadata.version` above is the release this copy shipped with and `metadata.source` is where it came from. One call says whether a newer one exists:

```sh
gh api repos/88Lookingupatthestars88/agent-github-fieldguide/releases/latest --jq '.tag_name, .published_at'
```

A tag above that number means this copy is behind. Report both numbers; replacing the file is the user's edit.

## Step 9.5 — Repository settings

None of these fields sits in a commit, so they are set after the files land.

| Field | Endpoint | Permission |
|---|---|---|
| description, homepage | `PATCH /repos/<owner>/<repo>` | Administration: write |
| has_discussions, has_issues, has_wiki, has_projects | `PATCH /repos/<owner>/<repo>` | Administration: write |
| default_branch | `PATCH /repos/<owner>/<repo>` | Administration: write |
| topics | `PUT /repos/<owner>/<repo>/topics` | Administration: write |
| social preview image | no endpoint | none |

Topics do not go through `PATCH /repos/<owner>/<repo>`. That request accepts the field, answers 200 and returns the old list unchanged, so a caller trusting the status code reports success over a write that never happened. Use `PUT` with `{"names": [...]}`, which replaces the whole list.

Three rules come back as 422 with `must start with a lowercase letter or number, consist of 50 characters or less, and can include hyphens`: at most 20 topics; lowercase letters, digits and hyphens only; order is not kept, and the response comes back alphabetised with no topic weighing more than another.

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

The percentage does not divide evenly by the number of entries. On this repository the observed steps were 42 with a readme and a licence, 71 after CONTRIBUTING and CODE_OF_CONDUCT, 85 after the pull request template, and 100 after Pages. Read `files` for what is missing rather than deriving it from the number.

`issue_template` takes one file at `.github/ISSUE_TEMPLATE.md`. A form at `.github/ISSUE_TEMPLATE/report.yml` with a `config.yml` beside it gives a working chooser and leaves that field `null`, so write both. `documentation` takes a wiki page or a Pages site, and `has_wiki: true` on its own leaves the field `null`.

```sh
gh api -X POST repos/<owner>/<repo>/pages -f 'source[branch]=main' -f 'source[path]=/'
```

Pages publishes the whole branch and Jekyll renders `README.md` as the index. `documentation` fills on the next read, and `health_percentage` lags several minutes, so a read taken straight after still returns the old number.

A run ending in `Page build failed`, `duration: 0` and a 500 on the deployment had no content problem. GitHub's deployment service dropped it, and the next push runs it again.

The percentage counts files. A `CONTRIBUTING.md` with no build steps in it scores the same as one that has them.

## Step 9.7 — Auditing a repository

Run this before a repository goes public, and again whenever a token, a collaborator or a workflow changes. Several of these answer 404 or 403 when the feature is off, so read the field as well as the status code.

| What | Call | Value that needs action |
|---|---|---|
| A secret in the tree | `GET /secret-scanning/alerts` | any alert. A private repository answers 403; fall back to `git/trees?recursive=1` and grep the blobs |
| The identity on each commit | `git log --all --format='%an <%ae> \| %cn <%ce>'` | an address the author did not mean to publish |
| Paths that used to exist | `git log --all --diff-filter=D --name-only --pretty=format:` | a path name describing internal work |
| Commit messages | `git log --all --format=%s` | `probe:`, `wip`, `test`, anything naming a private plan |
| The licence carried | `GET /repos/<owner>/<repo>` → `.license.spdx_id` | `null`, which means nobody may copy it however the readme reads |
| Workflow code that can move | `GET /actions/permissions` → `sha_pinning_required` | `false`, which lets `uses: owner/action@v4` run whatever that tag points at that day |
| What a workflow token may do | `GET /actions/permissions/workflow` → `default_workflow_permissions` | `write`, which lets any workflow push to any branch |
| The default branch under a mistake | `GET /branches/<branch>/protection` | 404, meaning unprotected and open to force-pushes from any token holding write |
| Private vulnerability reporting | `GET /private-vulnerability-reporting` | `{"enabled": false}`, which leaves no private route for a report, and a `SECURITY.md` naming an address does not open one |
| Dependency alerts | `GET /vulnerability-alerts`, `GET /dependabot/alerts` | 404 and 403, both meaning off |
| What Pages serves | `GET /pages` → `public` | `true` on a private repository, which serves its files to anyone |
| What else reaches the repository | `GET /keys`, `GET /hooks` | an entry nobody remembers adding, which survives a token rotation |
| What the account publishes beside it | `/users/<owner>/events/public`, `/users/<owner>/{keys,gists,starred,social_accounts}` | an old push head, or an account detail the owner forgot |

The tree API reads file content and nothing else, so a scan built only on `git/trees` returns zero findings on a repository that publishes a personal address. Author and committer are fields on the commit object. `GET /commits` normalises every date to `Z`, so the author's time offset never appears there; it survives only in `git log` and in `search/commits`.

Three channels keep an old SHA or an old identity reachable after it has left the files, and each closes on its own schedule:

- `GET /commits/<old-sha>` answers 200 until GitHub collects the objects. Only a request to support clears that sooner.
- `search/commits?q=author-email:<old>` keeps returning the old rows until the index rebuilds. On a 34-commit repository this took twelve minutes.
- `/users/<owner>/events/public` keeps the push heads recorded before a rewrite for ninety days.

Two switches need `Administration: write` and take an empty body:

```sh
curl -X PUT -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/private-vulnerability-reporting
curl -X PUT -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/vulnerability-alerts
```

The first answers 204. The second enables Dependabot alerts; security updates are a separate switch set through `security_and_analysis.dependabot_security_updates` on `PATCH /repos/<owner>/<repo>`. With no manifest in the tree, both are moot.

No endpoint lists a token's scopes. Every authenticated response carries `x-accepted-github-permissions`, which names what that call needed, and the union across a session is the set the token used. Compare that against what the session was for: `administration=write` carries deleting the repository.

Three lines per finding: the field, its current value, and the one call that changes it. A decision belonging to the owner — which licence, whether the repository should be private — gets named as a decision rather than made.

## Step 9.8 — Taking an identity out of history

A commit SHA is computed over its tree, its parents, the author and committer fields, and the message, so changing one address on an old commit changes that SHA and every commit after it. Adding a commit leaves the old ones untouched, and the account's private-email setting applies only to commits made after it is switched on. Count forks first with `GET /repos/<owner>/<repo>/forks`; a fork keeps the old objects, and after one exists the rewrite stops removing the address from anywhere but the parent repository.

```sh
git clone https://github.com/<owner>/<repo> work && cd work
git bundle create ../pre-rewrite.bundle --all && git bundle verify ../pre-rewrite.bundle
git rev-parse 'HEAD^{tree}'                   # compare again after the rewrite
FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch -f --env-filter '
NR="<id>+<login>@users.noreply.github.com"
[ "$GIT_AUTHOR_EMAIL" = "old@example.com" ] && export GIT_AUTHOR_EMAIL="$NR"
[ "$GIT_COMMITTER_EMAIL" = "old@example.com" ] && export GIT_COMMITTER_EMAIL="$NR"
' -- --all
git for-each-ref --format='%(refname)' refs/original | xargs -r -n1 git update-ref -d
```

The filter sets the two address variables and touches nothing else, so dates and names survive. `refs/original/*` has to go, or `git log --all` still reaches the old objects.

Push with one lease per ref, so a ref that moved meanwhile refuses rather than overwrites:

```sh
git push --force-with-lease=refs/heads/main:<old> \
         --force-with-lease=refs/tags/v1.2.0:<old> \
         origin refs/heads/main refs/tags/v1.2.0
```

Four reads decide whether the rewrite worked:

1. `HEAD^{tree}` matches the value read before it. Only the identity changed.
2. `git log --format='%ae' | sort -u` prints one address.
3. `GET /releases` keeps the same `tag_name`, the same asset count and the same `download_count`.
4. `GET /tarball/refs/tags/<tag>` unpacks to a top-level directory ending in the new SHA, which is how the archive on the release page is shown to be new.

A rewritten commit is unsigned, so an Initial commit GitHub signed through the web editor loses its verified badge and reads as Unverified. There is no way back short of signing it again.

Report which of the three residue channels has cleared. Calling the address gone before all three do is a false report.

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

Read the status code first, since the body is often generic. Then read `x-accepted-github-permissions`, which every authenticated response carries, 404 included: it names the permission and level the endpoint checks, such as `contents=read` or `administration=read`. A 401 carries no such header, which is how a bad credential is told apart from a credential that lacks a permission. A header naming a permission the token already holds means the permission is not the problem, so check the repository selection, then whether an organization has approved the token.

- `401` — the credential is absent, malformed, revoked or expired. github-onboard owns that; reissuing is the fix and retrying is not.
- `403 Resource not accessible by personal access token` — a fine-grained token was refused. The header names the permission; add it or raise its level. Only when no fine-grained permission exists for the operation does the token type change.
- `403` with `X-GitHub-SSO` — the organization has not authorized this token. The user clears it on the organization's SSO page.
- `403` carrying `rate limit` — read `X-RateLimit-Reset`, wait, retry once. Secondary limits send the same code without that header and want a pause.
- `404` on an object that exists in a browser — the credential lacks access, because GitHub answers 404 instead of 403 on purpose.
- `409` on a contents write — the `sha` you sent is stale because the file changed since the read. Re-read and retry once.
- `422 Invalid request` — malformed input: `ref` without `refs/`, a tree entry without `mode`, a `base_tree` that is a commit SHA, a parent that is not a commit. A contents write that omits `sha` for a file that exists lands here with `"sha" wasn't supplied`.
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

**Release notes.** The reader is deciding whether to download. Say what changed for them, say what they have to do, and keep it to one paragraph. Field names, endpoint names and the reason a design was chosen stay out, and so does a change the user cannot see; report that one in the conversation. No inline code in a release body, because GitHub's machine translation pushes every code span to the end of its paragraph and the sentence arrives full of holes.

**Readme.** The opening paragraph answers what this is and whether it runs on the reader's machine. Exact steps sit below it.

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

Two things the commands miss: a list whose items all begin with a noun has no subject, and a byte count beside a download link goes stale the moment the file changes, so run `wc -c`, paste the number, and recheck it in the commit that touches the file.
