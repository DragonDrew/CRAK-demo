# CRAK: CRON Rsync Automated Keeper

**CRAK is a web UI for backups across all your machines.** Set up your backups once in the browser, and CRAK puts them into each machine's own cron, collects every result, and tells you when something goes wrong. Backups are plain files that rsync writes, so you can still browse them with `ls` if you ever stop using CRAK.

If you run a few Linux boxes and a NAS, you probably have rsync scripts in cron somewhere, a few of them failing silently. CRAK keeps the scripts on each machine (so backups still run when CRAK is down), and gives you one place to see every backup, every result and every restore.

These screenshots come from a clean CRAK **0.11.0** install with made-up hosts (`demo-web`, `demo-pi`, `demo-nas`), a made-up destination (`Demo NAS`), made-up people (`maria`, `sam` and `pat`, with `demo-admin` as the owner) and made-up data. They're in dark mode (the default).

---

## The pitch in one screen

- **Backups run from cron on each machine**, not from the CRAK server. So they keep running even when CRAK is down.
- **CRAK does the boring bits**: it builds the rsync commands, writes the cron lines, copies them to each host, and keeps the history.
- **Nothing is locked in.** Each job is a readable shell script on its host, and the backups are ordinary folders on the destination.
- **You can see everything in one place**: every run, its result in plain words, its log, and every alert, and who is looking at each one.

---

## Features

### Backup jobs
- Build a job in the browser: tick folders in a tree, pick a host and a destination, and see the exact script and cron line before you save.
- **Mirror** (one copy, updated each run) or **snapshots** (a dated copy each run, hard-linked so unchanged files take no extra space, with retention per job or in Settings).
- **Run as** another user on the host, with that user's own SSH key.
- **Before and after steps**, such as stopping containers for a consistent copy.
- **Notes** on each job, shown on its page and written as comments in the host's cron file.
- **Dry run** shows what would be copied, changed or deleted. It writes nothing except the job's empty folder on the destination.
- **Job groups** back up the same folder on several hosts with one definition, starting them a few minutes apart.
- **Host-to-host** copies, one-way, into a folder on another host.
- **Revisions**: every change to a job, who made it and what changed, with **Roll back to this revision**.
- **Pause** a job, a host, a destination, a group or everything: for 1, 4 or 24 hours, until you resume, or until a time you choose.

### Working with many jobs (new in 0.11)
- **A row menu on every row.** Each list row has a **⋯** button, and right-clicking the row opens the same menu. It holds the actions of the object, for your role.
- **Bulk actions on jobs.** Tick several jobs and run them, verify them, pause, resume, enable or disable them together. A confirm lists what each job will do and what it will skip, and the results come back job by job.
- **Search and filters** on the jobs, hosts, destinations and scripts lists. The filters are in the address, so a link shows the same list.
- **Saved views** for the filters you use most, kept per user on the server and optionally the default for their list.
- **Command palette** (Ctrl+K): jump to any page, job, host or destination, or run an action on it, through the same confirm as its page.
- **Keyboard shortcuts** for the common moves (`?` lists them), switchable per user.

### Scheduled tasks and the script library
- A library of scripts with versions, variables and defaults.
- Built-in templates for PostgreSQL, MySQL/MariaDB, SQLite and Docker volume dumps.
- A scheduled task is a script plus its values, and runs from cron on the host like any other job.
- **Import** existing cron jobs, including their scripts, as jobs or tasks.

### Monitoring and alerts
- Every run has a result, statistics, and a searchable log you can download. A failure says what went wrong, in plain words.
- Run-time history per job, and a flag for runs that are much quicker or slower than usual.
- **Verify** compares a backup with its source by checksum, on demand or on a schedule, and reports damaged files.
- Alerts for backups that failed, didn't run, were lost, or are damaged; hosts that are offline; destinations that can't be reached; and a job whose "run as" user is missing.
- **Acknowledge and assign** alerts. Someone can say they're looking at one, and assign it to another Operator or Admin, so it's clear who has it. The assignee is emailed if an email channel is set up.
- **Notification channels**, up to 20: Discord, Slack, Microsoft Teams, ntfy and email. Each channel chooses which alerts it gets, and whether it also gets news such as pauses.
- **Prometheus metrics** at `/metrics`, optionally behind a token.

### Users and sign-in
- Several people can sign in, with three roles: **Viewer** (sees everything, changes nothing), **Operator** (also runs, verifies, pauses and handles alerts) and **Admin** (changes anything, restores files, manages users).
- A **Users** page for Admins, with search, filters, and each person's details, role, two-factor state and sign-in method.
- **Two-factor sign-in** with an authenticator app, with recovery codes.
- **Single sign-on** through an identity provider (OpenID Connect), with the provider's groups mapped to CRAK's roles.
- **Personal API tokens** for scripts and tools, each with a role and an expiry.
- An **audit log** of every change and sign-in, with who made it and from where, kept for a year. Filter it by action, by person and by date.

### Log forwarding
- The audit log, and optionally alerts and run results, go to a **syslog** server (RFC 5424, over UDP, TCP or TLS) or to a **SIEM** over HTTPS, as JSON, NDJSON or Splunk HEC.
- If a destination can't be reached, CRAK keeps the events for up to 7 days and sends them once it can.

### Restore
- Pick the host a backup came from, the job, and the copy (a snapshot or the mirror).
- Browse the backup where the files were on the original machine, and tick what you want back.
- Restore in place, into another folder, or **onto another host**, such as the machine that replaced a dead one.
- A dry run first, and overwriting existing files is off by default. A restore never deletes anything.

### Hosts and destinations
- Add a host with a compose file and a one-time token. CRAK shows what each host is ready for (rsync, SSH, cron, flock).
- A destination is any machine with SSH and rsync, such as a NAS. CRAK doesn't need to be installed on it.
- **Install key** once per host (the password is typed once, never stored), then **Test connection** checks login, rsync, write access, hard links, free space and extended attributes.
- CRAK retries a destination that doesn't answer straight away, such as a NAS whose disks are asleep, and only fails a run after four tries.
- **Replace with…** moves a dead host's jobs to its replacement, with their history, notes, pauses and destination folders.

### Operations and settings
- **Settings** in six sections: General (run history, snapshots, CRAK's own backup), Notifications, Sign-in, Integrations (log forwarding, Prometheus, export and import), Audit log, and Your account (appearance, password, two-factor sign-in, API tokens, keyboard shortcuts and saved views). Each section has its own address, so a link opens it.
- **YAML export and import** of jobs, groups, destinations and scripts, so a job set can live in Git.
- **CRAK backs itself up**, on a schedule, to a destination you choose.
- **HTTPS** through a reverse proxy, with a security guide.

---

## How it works

It's a few pieces working together:

1. **The CRAK server** runs the web UI and keeps the database. It has no backup work of its own.
2. **An agent** runs on every machine you want to back up. It reports the machine's status, and it puts CRAK's jobs into that machine's cron.
3. **Cron on each machine** runs the jobs on schedule. It works even if the CRAK server is down.
4. **Each run** writes its result and log back to the server, which shows them in the history and raises alerts when something needs attention.
5. **Destinations** are the NAS or other machine where the backups are stored. They just need SSH and rsync.

Every change goes through the server, which sends it to the agent. CRAK never runs arbitrary commands on your machines: the agent only does a fixed set of actions, such as running a job, testing a destination, or installing a key.

---

## The demo

### Dashboard

![Dashboard](screenshots/01-dashboard.png)

One page for "is anything on fire?": hosts online, open alerts, jobs whose last run went wrong, every run of the last 30 days, and each job's last result and next run. The open alert here has been acknowledged by the owner and assigned to Sam, with a note saying why.

### Hosts

![Hosts](screenshots/02-hosts.png)

Every machine running the CRAK agent, with its status, OS, agent version, and how many jobs are deployed.

![Host detail](screenshots/03-host-detail.png)

Each host shows what it's ready for, the cron file and job scripts CRAK has put on it, and any cron jobs CRAK didn't create. Those can be imported with one click.

![Second host](screenshots/22-host-pi.png)

### Destinations

![Destinations](screenshots/04-destinations.png)

![Destination detail](screenshots/05-destination-detail.png)

Install a key once per host, then **Test connection** checks everything a backup needs.

### Backup jobs

![Jobs](screenshots/06-jobs.png)

![Job detail](screenshots/07-job-detail.png)

Each job has its own page with what it backs up, its schedule and next run, its notes, a run-time chart, the full run history, and every revision of its script. **Run now**, **Verify now**, **Restore** and **Pause** are at the top.

![Snapshot job](screenshots/17-job-appdata.png)

Snapshot jobs list the snapshot each run made.

![Pause](screenshots/23-job-pause.png)

**Pause…** stops a job's scheduled runs until you resume it, or until the time you choose. **Run now** still works while paused.

![Job builder](screenshots/08-job-builder.png)

The builder has a live preview. You tick folders in a tree, choose the switches (each showing the rsync flag it turns on), and **What will run** shows the exact script and cron line, with the next runs, before anything is saved.

![Job script](screenshots/09-job-script.png)

Every job's script is readable. If you tweak it by hand, CRAK keeps a revision for each change.

### Working with many jobs (0.11)

![Row menu on the jobs list](screenshots/32-jobs-row-menu.png)

Each row's **⋯** menu offers that job's actions for your role: **Run now**, **Verify now**, **Pause…**, **Restore**, **Edit**, and **Disable** or **Delete job…** at the bottom. Right-clicking the row gives the same menu.

![Bulk selection](screenshots/33-jobs-bulk-select.png)

Tick two jobs and a bar appears above the list with the actions for both. The confirm lists what each job will do and what it will skip, and the results come back job by job.

![Filters and saved views](screenshots/34-jobs-filters-views.png)

The search box and the menus of tick boxes narrow the list, and **Views** keeps the filters you use, optionally as the default for the list.

![Command palette](screenshots/35-command-palette.png)

**Ctrl+K** finds anything: here, the job `photos`. **Tab** lists its actions, and the action runs through the same confirm as its page.

![Keyboard shortcuts](screenshots/36-keyboard-shortcuts.png)

**?** lists the keyboard shortcuts. Each user can switch them off under Your account.

### Job groups

![Job group](screenshots/10-group-detail.png)

One backup definition on several hosts, starting a few minutes apart so they don't all hit the NAS at once. Editing the group updates every host's copy.

### Scheduled tasks and scripts

![Scripts](screenshots/11-scripts.png)

A library of reusable scripts, with built-in templates for database dumps.

![Script detail](screenshots/12-script-detail.png)

Each script has versions, variables with defaults, and a preview of exactly what a task will run.

![Task](screenshots/16-task-detail.png)

A scheduled task is a script plus its values, with its own run history.

### Run results

![Run detail](screenshots/15-run-detail.png)

Every run has a result, statistics, and the full log, which you can search or download.

![Failed run](screenshots/18-run-failed.png)

A job running as `archive` can't read a root-only file. The run names the file and says what to do, instead of leaving you to decode exit code 23.

![Held-back job](screenshots/21-job-legacy-sync.png)

A job whose "run as" user has been removed from its host is held back: CRAK raises an alert and shows why at the top of the job page. The host's other jobs still deploy.

### Restore

![Restore](screenshots/13-restore.png)

The restore takes four steps: choose the backup and copy, tick the files from the tree of where they were, choose where they go, then check and restore.

![Restore to another host](screenshots/19-restore-to-host.png)

Here, the first step is open: the backup is the `app-data` snapshot from `demo-web`, and the tree shows the folder as it was on that machine. The next step is **Where to**, which can be another machine, such as the replacement for a dead one. That machine reads the backup with its own key, and the page warns if the key isn't on the destination yet.

### Alerts

![Alerts](screenshots/31-alerts.png)

Open alerts, failures first. The one here has been acknowledged by the owner, with a note, and assigned to Sam.

### Themes

Each user picks a theme and a colour mode (light, dark or their device's), and a density (comfortable or compact). These three show the same dashboard in the themes built in or imported, in each of their modes and densities.

![CRAK theme](screenshots/theme-crak.png)

CRAK, the built-in theme: light and dark, comfortable and compact.

![Terminal theme](screenshots/theme-terminal.png)

Terminal, the built-in monospace theme. It has no light mode, so only the dark versions are shown.

![Amber Mono theme](screenshots/theme-amber-mono.png)

Amber Mono, imported from a tweakcn export, in light and dark, comfortable and compact.

### Settings

![Settings](screenshots/14-settings.png)

Settings is in six sections, as tabs along the top, each with its own address. This is **General**, where run history, snapshots to keep and CRAK's own backup are set.

#### Notifications and alerts

![Notification channels](screenshots/24-settings-notifications.png)

Each channel has its own alerts and its own news setting. This demo has no channels yet: **Add** offers Discord, Email, Microsoft Teams, ntfy and Slack. A channel shows **on**, **off**, **failing** (with the last error, and it keeps trying) or **stopped** (the service said the webhook is gone). The alert thresholds below it are set for the demo.

#### Integrations

![Log forwarding](screenshots/25-settings-log-forwarding.png)

Syslog or a SIEM receives the audit log, and optionally alerts and run results, as they happen. This demo has nothing forwarded yet: **Add** offers syslog (over UDP, TCP or TLS) or an HTTP SIEM. Prometheus metrics and the YAML export and import are here too.

#### Users

![Users](screenshots/26-users.png)

Everyone who can sign in, with their role, two-factor state and sign-in method. Search and filter the list, and each row's **⋯** menu holds the actions for that person.

![Row menu on a user](screenshots/37-users-row-menu.png)

A person's row menu: their details, editing them, changing their role, resetting their password or two-factor sign-in, and disabling or deleting the account.

![User details](screenshots/38-user-details.png)

The details panel shows the person's email, two-factor and sign-in method, their API tokens, and their most recent audit entries. A new person gets a temporary password and must choose their own at first sign-in.

#### Sign-in

![Single sign-on](screenshots/27-settings-single-sign-on.png)

Sign in through an identity provider that speaks OpenID Connect, with the provider's groups mapped to roles. The demo has no provider, so this card shows the settings only. Single sign-on has been tested with a test provider, not yet with a live one.

#### Your account

![Two-factor sign-in](screenshots/28-settings-two-factor.png)

Two-factor sign-in with an authenticator app, with 10 single-use recovery codes. An Admin can also require it for everyone.

![API tokens](screenshots/29-settings-api-tokens.png)

A token for a script or tool, with its own role and expiry. It's shown once when it's made, and an Admin can see and revoke everyone's.

![Saved views](screenshots/39-settings-saved-views.png)

Every saved view you have, for each list, with the filters it holds. Open one to apply it, or delete it.

#### Audit log

![Audit log](screenshots/30-settings-audit-log.png)

Sign-ins and changes, newest first, kept for a year. Filter by action, by person or by a range of days, and export everything that matches as CSV.

---

## What it isn't

- **Not a backup format.** The backups are plain folders on your destination, so you don't need CRAK to read them.
- **Not for other platforms.** The server and agents are Linux amd64 only.
- **Not for the internet.** It's meant for a private network, or behind a reverse proxy with HTTPS. Roles apply to the whole install, not per host or per job.
- **Not all tested against live services.** Slack, Microsoft Teams, single sign-on and SIEM receivers have been checked only against local copies or test providers. Each one can be tried from its own card in Settings.
