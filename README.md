# ghday

A local page that tells me what to do on GitHub today.

I open it in the morning and a few times during the day. It reads my GitHub
activity through the `gh` CLI and sorts it by what I have to do next:
release, answer, fix, merge, review. It also writes my daily standup and
shows a few stats about my week and month.

It is a single Python script with no dependency outside the standard library.
It runs on my machine, listens only on `127.0.0.1`, and talks to GitHub with
the `gh` login I already have.

## Requirements

- Python 3.9 or newer
- [GitHub CLI](https://cli.github.com/) logged in (`gh auth status`)
- `notify-send` for desktop notifications (optional)
- systemd user session to run it as a service (optional)

### Token scopes

ghday reads from GitHub. The only write is the "Mark all read" button, which
marks notifications read. It needs a classic token with these scopes:

| Scope | What it gives | Without it |
|---|---|---|
| `repo` | Issues and PRs of private repositories, and the notifications | Only public repositories show, and private contributions are not counted |
| `read:org` | Review requests sent to a team | Team review requests may be missing |
| `read:project` | The Status column of Projects v2 | The status badge is missing, the rest works |

A default `gh auth login` gives `repo` and `read:org`. Add the last one with:

```sh
gh auth refresh -s read:project
```

`repo` also allows writing, because classic tokens have no read only scope for
private repositories. Fine-grained tokens do not work: the notifications API
refuses them, and they are limited to one owner.

## Install

```sh
install -m 755 ghday ~/bin/ghday
```

Run it once to build a static page and open it in the browser:

```sh
ghday
```

Or run it as a small local server that refreshes in the background:

```sh
ghday serve        # http://127.0.0.1:8765/
ghday serve 9000   # another port
```

To start the server with the session, install the user unit:

```sh
cp ghday.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now ghday
journalctl --user -u ghday      # logs
```

The server keeps the code in memory. After editing the script, restart it
with `systemctl --user restart ghday`.

## What the page shows

### Today

A strip of counters on top, one per kind of action, then one card per kind.

| Card | What lands there |
|---|---|
| To release | QA issues marked `verified` that I worked on. The package waits for my release. The repository to release comes from my merged PRs linked to the issue. |
| In testing | QA issues marked `testing` that I worked on. Nothing to do yet, they move to "To release" once verified. |
| To review | Review requests, and PRs I already reviewed that got new commits since. |
| To answer | My PRs with changes requested, or a comment or review I did not answer. Issues where someone commented after me. An approval never counts as something to answer. |
| To fix | My PRs with failing CI or a merge conflict. QA issues I opened with nobody assigned, because they would skip the QA flow. |
| To merge | My PRs approved, with green CI (or no CI) and nothing left to answer. |
| New issues | Issues assigned to me in the last 3 days. |
| Waiting on others | My other open PRs, grouped by the reviewer they wait for, oldest first. PRs with no reviewer requested are flagged. |

A red dot marks what is new since my last visit. Reloads within 30 minutes
count as the same visit.

Each line has two buttons on hover. "Tomorrow" hides it until 7:00 the next
day. "Hide" hides it until something new happens on it, like a comment or a
commit. "Show all" brings everything back.

The right column lists the issues assigned to me, newest first, then two
folded sections: issues I opened or that mention me, and unread
notifications that are not already on the page.

### Daily

A standup text ready to paste in a chat channel, with a copy button.

- Yesterday: PRs merged and opened, reviews done, QA changes on my issues.
  On Monday it covers Friday. A PR title landing in more than two
  repositories is folded into one line.
- Today: the action cards written as sentences.
- Blockers: PRs waiting for review and issues in QA testing.

### Stats

On top, this week and this month side by side, against the same point of the
previous period, so Monday morning does not look like a drop.

Below, two columns on a wide screen. The left one has my median review
response time, from a review request to my first review after it, then PRs
merged, reviews done and issues delivered over the last 12 weeks. The right
one lists my activity per repository for the month. On a narrow screen they
stack.

Stats are cached for one hour because they cost most of the API calls. The
cache holds the rendered section, so after updating the script, delete
`~/.local/state/ghday/stats.json` to see a new layout at once.

### History

Server mode only. One month at a time, with arrows to go back. It opens on
the previous month.

- PRs merged, PRs opened, reviews done, issues delivered and my median review
  response time, against the month before.
- PRs merged and reviews done over the 12 months up to that one.
- The lists behind the numbers: PRs merged by repository, issues delivered
  with the repositories of my PRs, PRs I reviewed.

A month is loaded only when I open it. A closed month no longer changes on
GitHub, so it is computed once and kept on disk for good. The current month
follows the same one hour rule as Stats. To recompute a month, delete its file.

## The QA flow

The script follows the NethServer way of working in `NethServer/dev`. A
developer is unassigned when the issue moves to `testing`, but once QA marks
it `verified` the developer still has to release the package. So the script
does not rely on the current assignee. It looks at the issue timeline and
keeps the issues I was assigned to at some point, or that one of my PRs
references.

The repository and labels are set at the top of the script:

```python
QA_REPO = "NethServer/dev"
QA_LABELS = ("testing", "verified")
```

## Notifications

In server mode, every refresh compares the action items with the previous
run and sends a desktop notification for each new one. "To release" items use
the critical urgency, so they stay on screen until I close them. The first run
only records what exists, so a restart does not replay everything.

## Files

Everything lives in my home directory, nothing is sent anywhere except the
GitHub API calls made by `gh`.

| Path | Content |
|---|---|
| `~/.local/state/ghday/state.json` | Last visit time |
| `~/.local/state/ghday/stats.json` | Stats cache (1 hour) |
| `~/.local/state/ghday/notified.json` | Items already notified |
| `~/.local/state/ghday/hidden.json` | Hidden and snoozed items |
| `~/.local/state/ghday/months/YYYY-MM.json` | History, one file per month |
| `~/.local/share/ghday/index.html` | Page built by `ghday` and `ghday build` |

## Limits

GitHub search returns at most 1000 results, so the stats stop counting past
that. The page reads the first 60 assigned issues, 60 open PRs and 40 review
requests, which is plenty for one person.
