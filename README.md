# ShipIt Sprint: DevOps Warriors

> DevOps Bootcamp · Team Activity
> Build, ship and run a website together: Jira, Slack, Git, GitHub Actions, Nginx and shell scripts.

Team name: **DevOps Warriors** (technical name `falcon`, Jira project key `DEVOPS`).

Replace `DOMAIN` and `SERVER_IP` with your team's real values.

## Team

| Member | GitHub username | Role |
| --- | --- | --- |
| Manhal Malik | nahalmalik | Team Lead |
| Ather Ullah | atherullah | Server Admin |
| Zahoor Ahmed | zahoorahmad334 | Pipeline Engineer |
| Safyan Talha | safyantahir03-collab | Scripter |

**Roles**

| Role | Leads |
| --- | --- |
| Team Lead | The Jira board, the repository, merging pull requests, keeping time |
| Server Admin | The team server: users, permissions, Nginx, SSH access for GitHub Actions |
| Pipeline Engineer | The GitHub Actions workflows and the Slack notifications |
| Scripter | The shell scripts and cron jobs |

Each member leads one area, but everyone works in every phase.

## Links

| What | Where |
| --- | --- |
| Live site | http://DOMAIN |
| Status page | http://DOMAIN/live/status.html |
| Traffic report | http://DOMAIN/live/report.html |
| Repository | https://github.com/atherullah/shipit-falcon |
| Jira board | https://nahalsworkspace-39376139.atlassian.net/jira/software/projects/DEVOPS/boards/2?filter=&groupBy=none (project key `DEVOPS`) |
| Slack | https://app.slack.com/client/T0CJBT87XA6/C0CJA8Z3QVC (channels `#falcon`, `#falcon-dev`, `#falcon-alerts`) |

## Repository layout

```
.
├── site/                      # The website (deployed to the server)
│   ├── index.html
│   ├── style.css
│   └── members/*.html         # One page per team member
├── scripts/
│   ├── team.conf              # Team name and domain
│   ├── check-site.sh          # Checks titles, links and file sizes
│   ├── deploy.sh              # Runs on the server: new release, switch, cleanup
│   ├── rollback.sh            # Points `current` back one release
│   ├── status.sh              # Writes the live status page (cron, every minute)
│   └── traffic-report.sh      # Writes the traffic report (cron, every 5 minutes)
└── .github/workflows/
    ├── check.yml              # Runs on every pull request
    ├── deploy.yml             # Runs on every push to main (and by hand)
    └── rollback.yml           # Run by hand from the Actions tab
```

## How we work

- **All team communication is on Slack**, even when sitting next to each other. Use threads, and huddles when needed.
- **Every piece of work is a Jira issue** with an assignee. Issues move through To Do → In Progress → In Review → Done.
- **Nobody pushes directly to `main`.** Every change goes through a pull request.
- **Every pull request needs one approval** from a member who didn't write it, and the reviewer requests at least one change before approving.
- **Branch names and PR titles start with the Jira key**, for example branch `DEVOPS-4-ali-page` and PR title `DEVOPS-4 Add Ali's page`.
- **Website changes go through Git**, never by editing files on the server. Server problems are fixed on the server.
- **No secrets** (passwords, private keys, webhook URLs) in the repository, Slack or Jira.

## Server

- Each member has their own Linux user and logs in with their own SSH key. Password login is off.
- A `team` group holds all members. A separate `deploy` user runs deployments.
- Only ports **22** (SSH) and **80** (HTTP) are open.
- Nginx serves `/var/www/falcon/current` at `http://DOMAIN` and `/var/www/falcon/live` at `http://DOMAIN/live/`. The bare IP does not show the site.
- Access log: `/var/log/nginx/falcon.access.log`

```
/var/www/falcon/
├── current -> releases/<timestamp>-<commit>   # live site (symlink)
├── releases/                                  # last 5 releases kept
├── upload/                                    # files uploaded by the workflow
├── live/                                      # status.html, report.html
├── bin/                                       # deployed scripts
└── deploys.log                                # one line per deploy / rollback
```

## CI/CD

**On every pull request** (`check.yml`): runs `scripts/check-site.sh` on `ubuntu-latest`. It fails on:
- an HTML page without a `<title>`
- a broken link to a file inside the site (external links, `#` anchors and `/live/` are skipped)
- any file larger than 1 MB

A red check means don't merge.

**On every push to `main`** (`deploy.yml`, also runnable by hand):
1. Upload `site/` and `scripts/` to the server over SSH as `deploy`.
2. Create a new release folder under `releases/`.
3. Replace `__VERSION__` in every page with the commit, who pushed it and the time.
4. Point `current` at the new release.
5. Append a line to `deploys.log`.
6. Keep only the 5 newest releases.
7. Copy the scripts to `bin/` and install the `deploy` user's crontab.
8. Run a health check from the GitHub runner: it fails unless `http://DOMAIN` answers and shows the new commit.
9. Post the result (success or failure, commit, author, run link) to `#falcon-alerts`.

Two deployments never run at the same time.

**Rollback** (`rollback.yml`, by hand from the Actions tab): points `current` at the release before the live one, logs it, and posts who pressed the button to `#falcon-alerts`. If there is no older release it says so and fails.

### GitHub Actions configuration

| Type | Name | Purpose |
| --- | --- | --- |
| Secret | `DEPLOY_SSH_KEY` | Private key of the deploy-only key pair |
| Secret | `SSH_KNOWN_HOSTS` | Server host key (host key checking stays on) |
| Secret | `SLACK_WEBHOOK_URL` | Slack alerts webhook |
| Variable | `DEPLOY_HOST` | The server's address |

## Scheduled scripts

| Script | Output | Schedule |
| --- | --- | --- |
| `status.sh` | `/live/status.html`: hostname, uptime, load, memory, disk, Nginx state, live release, last 5 deploys, red WARNING if disk > 80% or Nginx is down | Every minute |
| `traffic-report.sh` | `/live/report.html`: total requests, 404 count, top 5 pages, top 5 visitors, top 5 missing pages (excludes `/live/`) | Every 5 minutes |

## Local setup

```bash
git clone https://github.com/atherullah/shipit-falcon.git
cd shipit-falcon
git checkout -b DEVOPS-<number>-<short-name>

# make your change, then
git add .
git commit -m "DEVOPS-<number> <what you changed>"
git push origin DEVOPS-<number>-<short-name>
# open a pull request, ask for review in #falcon
```

To open the site from your laptop, map the domain to the server in your hosts file:

```
SERVER_IP  DOMAIN
```

## Incident process

When the trainer posts "Incident time" in `#falcon`:

1. **Notice it** using `#falcon-alerts`, the site, the status page, the traffic report, the Actions tab and server logs.
2. **Start a Slack thread** in `#falcon` beginning with `INCIDENT:` and what you see. All incident work happens there.
3. **Create a Jira Bug** with the same title and a link to the thread, in the Phase 4 epic.
4. **Fix it the right way**: website changes by pull request, server problems on the server.
5. **Write it up** in the Jira issue: What we saw, Cause, Fix, Prevention, Time.

## Timeline

| Phase | Time | Focus |
| --- | --- | --- |
| 0 | 25 min | Slack, Jira, team server, repository (A0–A2) |
| 1 | 35 min | Git collaboration: pages, conflicts, reviews, undo (A3–A6) |
| 2 | 35 min | Automated deployment: SSH access, checks, deploy, rollback (A7–A10) |
| 3 | 30 min | Shell scripts: status page, traffic report, self-deploying scripts (A11–A13) |
| 4 | 25 min | Incident round |
| 5 | 15 min | Demo (3 minutes per team) |

## Scoring

| Item | Points |
| --- | --- |
| Each of A0–A13 done and shown | 1 each (14) |
| Jira board kept up to date | 2 |
| All communication on Slack | 2 |
| Each incident fixed and written up | 2 each |
| Every member opened a PR in every phase | 2 |
| Each direct push to `main` | −1 |
| Each hint during the incident round | −1 |
| A secret posted in Slack, Jira or the repository | −2 |

---

_Activity created by Haroon Khan, CTO, AntonX Private Limited._
