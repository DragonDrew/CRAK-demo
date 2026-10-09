# CRAK: CRON Rsync Automated Keeper

**CRAK is a web UI for backups across all your machines.** Set up your backups once in the browser, and CRAK puts them into each machine's own cron, collects every result, and tells you when something goes wrong. Backups are plain files that rsync writes, so you can still browse them with `ls` if you ever stop using CRAK.

These screenshots come from a test setup with made-up hosts (`demo-web`, `demo-pi`, `demo-nas`), a made-up destination (`Demo NAS`) and made-up data. They're from CRAK **0.5.0**, in dark mode (the default).

---

## The big idea

- **Backups run from cron on each machine**, not from the CRAK server. So they keep running even when CRAK is down.
- **CRAK does the boring bits**: it builds the rsync commands, writes the cron lines, copies them out to each host, and keeps the history.
- **Nothing is locked in.** Each job is a readable shell script on its host, and the backups are ordinary folders on the destination.

---

## What it does

### The dashboard

![Dashboard](screenshots/01-dashboard.png)

One page for "is anything on fire?": hosts online, open alerts, jobs whose last run went wrong, every run of the last 30 days, and each job's last result and next run. Alerts are things like a failed backup, a job that didn't run, a job whose **Run as** user is missing, or CRAK's own data not being backed up. Resolved alerts stay under *Recently resolved* until you've seen them.

### Hosts

![Hosts](screenshots/02-hosts.png)

Every machine running the CRAK agent, with its status, OS, agent version, and how many jobs are deployed. Adding a host gives you a small compose file to run the agent, plus a one-time token.

![Host detail](screenshots/03-host-detail.png)

Each host page shows what that machine is ready for: rsync, SSH, cron and flock. It shows the cron file and job scripts CRAK has put on it, and any cron jobs it found that CRAK didn't create, which you can import as scheduled tasks with one click. The danger zone holds **Replace with…**, **Revoke credential** and **Remove host**.

![Replace with…](screenshots/20-host-replace.png)

**Replace with…** moves every job of a host to another machine, typically the one that replaced a dead one. The jobs keep their history, notes, pauses and destination folders, so the backups carry on where they were. Afterwards it tells you which destinations still need the new machine's key.

![Host, second machine](screenshots/22-host-pi.png)

### Destinations

![Destinations](screenshots/04-destinations.png)

A destination is where backups land: a NAS, or any machine with SSH and rsync. CRAK doesn't need to be installed on it.

![Destination detail](screenshots/05-destination-detail.png)

For each host you install a key once (you type the password one time and CRAK doesn't store it), then **Test connection** checks everything a backup needs: login, rsync, write access, hard links for snapshots, free space, and extended attributes. Jobs that run as another user log in with that user's own key, which is listed under its host.

### Backup jobs

![Jobs](screenshots/06-jobs.png)

The jobs list shows every backup at a glance, with its host, destination and last result. Scheduled tasks and CRAK's own backup are in the same list.

![Job detail](screenshots/07-job-detail.png)

Each job has its own page with:

- What it backs up and where it goes
- The schedule, with the next run time, and a **Verify** line showing the last check
- **Notes**: free text about the job, which is also written into the host's cron file, so it's there on the machine too
- A run-time chart, so unusually slow runs stand out
- The full run history, with what was transferred and how long each run waited for a free slot
- Revisions, so you can see what changed and when

**Run now**, **Verify now**, **Edit**, **Restore**, **Pause…** and **Disable** are right at the top.

![Snapshot job](screenshots/17-job-appdata.png)

Snapshot jobs keep a dated copy each run, hard-linked so unchanged files don't take extra space, and the job page lists the snapshot each run made. How many to keep is set per job or in Settings.

![Pause](screenshots/23-job-pause.png)

**Pause…** stops a job's scheduled runs until you resume it, for an hour, for a day, or until a time you choose. You can pause a job, a host, a destination, a group, or everything at once. **Run now** still works while paused.

![Job builder](screenshots/08-job-builder.png)

Creating or editing a job is a form with a live preview. You tick folders in a tree, pick a host and destination, and choose between:

- **Mirror**: one copy, updated each run
- **Snapshots**: a dated copy each run, with retention

The options are plain switches, each showing the rsync flag it turns on, and **Run as** lets a job back up as another user on the host. On the right, **What will run** shows the exact script and cron line before anything is saved, and the next runs. The timeline of other jobs on the same destination helps you avoid overlaps. **Verify the backup** can compare the backup with the source, monthly or on its own schedule.

![Job script](screenshots/09-job-script.png)

Every job's script is readable. If you need to tweak something by hand, you can edit it here, and CRAK keeps a revision for each change.

### Job groups

![Job group](screenshots/10-group-detail.png)

Back up the same folder on several machines with one job. The group starts them a few minutes apart, so they don't all hit the NAS at once. Editing the group updates every host's copy.

### Scheduled tasks and scripts

![Scripts](screenshots/11-scripts.png)

CRAK has a library of scripts you can reuse. There are built-in templates for PostgreSQL, MySQL/MariaDB, SQLite and Docker volume dumps, so a database backup doesn't mean writing shell from scratch.

![Script detail](screenshots/12-script-detail.png)

Each script has versions, variables with defaults, and a preview that shows exactly what a task will run. A scheduled task is a script plus its values, and it runs from cron on the host like everything else.

![Task](screenshots/16-task-detail.png)

Scheduled tasks show up in the same jobs list, with their own run history.

### Run results

![Run detail](screenshots/15-run-detail.png)

Every run has a result (what rsync reported, exit code, timings), a statistics panel, and the full log, which you can search or download. When a run fails, the reason is in plain words.

![Failed run](screenshots/18-run-failed.png)

Here the job runs as `archive`, which can't read a root-only file. The run says so, names the file and suggests the fix, instead of leaving you to decode exit code 23.

![Run as a missing user](screenshots/21-job-old-projects.png)

If a job's **Run as** user doesn't exist on its host, CRAK holds that one job back, raises an alert, and shows why at the top of the job page. The host's other jobs still deploy. **Check again** looks once more after you've created the user.

### Restore

![Restore](screenshots/13-restore.png)

Pick the host a backup came from, the job, and the copy to restore from. The folder tree shows files where they were on the original machine, so you tick what you want back and restore it in place or into another folder.

![Restore to another host](screenshots/19-restore-to-host.png)

**Restore to** can be another machine, such as a replacement for a dead one. That machine reads the backup with its own key, so the page warns you if CRAK hasn't installed the key on the destination yet. The dry run, overwrite option and "where to" choice work as they do for the original host.

### Settings

![Settings](screenshots/14-settings.png)

Settings are grouped into four parts: backups (how long runs are kept, snapshot retention, and CRAK's own backup), notifications and alerts (Discord, or just the dashboard), integrations (Prometheus metrics, and export and import as YAML), and the account (password and audit log). The audit log records every change, with who made it and from where.

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

## Good to know

- It's built for **Linux** machines (the agent is Linux-only, and the images are `linux/amd64`).
- It's meant for a **private network**. There's one admin login.
- Releases are versioned, so updates are deliberate. The current release is **0.5.0**.
- Everything is open for you to read: the scripts, the cron lines and the backups themselves.
