# Reaqt Remediation Playbooks

Ansible playbooks for event-driven remediation with **Reaqt**, the event
listener and rules engine in the [Ascender Pro](https://ciq.com/products/ascender-pro)
suite from CIQ. Reaqt watches incoming events (syslog, journald, webhooks),
matches them against rule sets, and reacts by launching an Ascender job
template — these playbooks are what those jobs run.

## How it works

1. A host logs something alarming ("No space left on device", an OOM kill, a
   failed service, ...) and the event reaches Reaqt.
2. A Reaqt rule matches the message and launches the Ascender job template for
   [reaqt.yml](reaqt.yml), providing:
   - a **limit** set to the affected host (`event.payload.hostname`),
   - an extra var **`message`** carrying the original log message,
   - an extra var **`task_file`** naming which remediation task file to run.
3. Play 1 of `reaqt.yml` runs on localhost and validates that the limit and
   both extra vars actually arrived, and that the task file exists in
   [tasks/](tasks/) — a misconfigured rule fails fast instead of running
   against every host.
4. Play 2 runs on the affected host and includes the named task file, which
   collects diagnostics, applies any safe remediation, and emails a report via
   [tasks/tools/mail.yml](tasks/tools/mail.yml) (mail server settings come from
   an Ascender credential; each task file passes only `subject` and `body`).

## What we react to

Each task file handles one class of problem. Rules match common kernel,
systemd, and daemon log messages for that class — a sample of the patterns is
listed here; the full curated sets are in the rule conditions.

| Problem | Example messages matched | Task file | Remediation |
|---|---|---|---|
| DNS resolution | "Temporary failure in name resolution", "no servers could be reached" | [dns_failure.yml](tasks/dns_failure.yml) | restart local resolver, re-test |
| Auth / SSH (burst: 5 in 1 min) | "Failed password for", "Invalid user", "account is locked" | [auth_failure.yml](tasks/auth_failure.yml) | report only |
| Disk usage | "Disk Warning", "No space left on device" | [disk_warning.yml](tasks/disk_warning.yml) | prune old kernels, /tmp, logs, journal, dnf cache, docker/podman |
| systemd service failure | "Failed to start", "Failed with result 'exit-code'" | [service_failure.yml](tasks/service_failure.yml) | report only (status + journal) |
| Memory exhaustion | "Out of memory: Killed process", "oom-killer" | [memory_exhaustion.yml](tasks/memory_exhaustion.yml) | restart safe services |
| CPU / lockups | "watchdog: BUG: soft lockup", "blocked for more than 120 seconds" | [cpu_exhaustion.yml](tasks/cpu_exhaustion.yml) | restart runaway service if safe-listed |
| Storage I/O errors | "I/O error, dev sdb", "sense key: Not Ready" | [storage_io_failure.yml](tasks/storage_io_failure.yml) | report only, local disks probed passively (no SMART self-tests, SAN left alone) |
| Time sync | "No suitable source for synchronisation", "Source ... offline" | [time_sync_failure.yml](tasks/time_sync_failure.yml) | restart chronyd if unsynchronised |
| Software RAID | "md/raid1:md0: Disk failure", "mdadm: ... failed" | [raid_failure.yml](tasks/raid_failure.yml) | report only |
| LVM | "Failed to activate logical volume", "PV ... not found" | [lvm_failure.yml](tasks/lvm_failure.yml) | retry activation (vgchange -ay) |
| Kernel issues | "kernel panic", "Oops:", "general protection fault" | [kernel_issue.yml](tasks/kernel_issue.yml) | report only |
| FD exhaustion | "Too many open files", "EMFILE" | [fd_exhaustion.yml](tasks/fd_exhaustion.yml) | restart the FD hog if safe-listed |
| PID exhaustion | "fork: Resource temporarily unavailable", "Cannot fork" | [pid_exhaustion.yml](tasks/pid_exhaustion.yml) | report only |

Every task file emails a report of what it found and did, even when a step
fails partway. Thresholds, age limits, test domains, and the safe-to-restart
service list are variables with sensible defaults, overridable per rule via
extra vars.

## Example rule

This is the Disk Warning rule as it appears in a Reaqt rule set export. The
`condition` matches the log message, and `provide` wires up everything play 1
validates: the limit and the `message` / `task_file` extra vars.

```yaml
- name: Disk Warning
  enabled: true
  triggers:
    throttle:
      enabled: true
      minutes: 5
    burst:
      enabled: false
      count: 5
      minutes: 10
  provider:
    name: Ascender Server
    template: 34
  rule:
    - name: Disk Warning
      condition:
        - >-
          'Disk Warning' in event.payload.message or
          'No space left on device' in event.payload.message or
          'Failed to create new system journal' in event.payload.message or
          'disk full' in event.payload.message|lower or
          'out of disk space' in event.payload.message|lower
      provide:
        limit: "{{ event.payload.hostname }}"
        extra_vars:
          message: "{{ event.payload.message }}"
          task_file: disk_warning.yml
```

A complete working rule set — all of the rules above plus a catch-all logger,
ready to import into Reaqt — is in
[reaqt_import/development_servers.yml](reaqt_import/development_servers.yml).

## Repository layout

```
reaqt.yml                Ascender job template playbook: validate, then include the task file
tasks/                   one remediation task file per problem class
tasks/tools/mail.yml     shared email task (community.general.mail)
reaqt_import/            example Reaqt rule set export/import files
collections/             collection requirements (community.general)
```
