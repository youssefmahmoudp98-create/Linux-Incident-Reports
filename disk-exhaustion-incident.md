## Disk Exhaustion — Snap Discord

### Executive Summary

A Linux filesystem became severely exhausted on an Ubuntu system. Investigation revealed that `/var/log` had grown to approximately 31 GB due to excessive logging caused by a Snap-packaged Discord installation repeatedly triggering AppArmor denials. The repeated security-policy denials were continuously recorded by the system logging infrastructure, causing abnormal disk consumption and eventually exhausting the filesystem. This report documents the investigation, root cause, remediation, and the preventative measures that would be appropriate in a production environment.

### Goal

Investigate a severe disk exhaustion issue on Ubuntu, determine what was consuming the disk, identify why `/var/log` had grown to approximately 31 GB, trace the excessive logging back to the Snap-installed Discord application, and understand the relationship between Snap confinement, AppArmor denials, and system log growth.

### What was done

**Checked filesystem-level disk usage**

```bash
df -h
```

-> checked the mounted filesystems and confirmed that the root filesystem was running critically low on available space

`df` reports disk-space usage for mounted filesystems. The `-h` option means human-readable, causing sizes to be displayed using units such as MB and GB instead of raw block counts.

This established **which filesystem was running out of space**, but it did not yet explain which directory or file was responsible.

---

**Inspected the largest directories under the root filesystem**

```bash
sudo du -h --max-depth=1 / | sort -hr
```

-> inspected disk usage at the root directory level and identified `/var` as a major consumer of storage

`du` calculates disk usage by examining files and directories and adding their storage usage together.

`--max-depth=1` limited the output to the immediate children of `/`, preventing the command from producing an enormous recursive listing.

`sort -hr` sorted the results in human-readable reverse order so that the largest directories appeared first.

The investigation narrowed the problem from the entire filesystem to `/var`.

---

**Inspected the contents of `/var`**

```bash
sudo du -h --max-depth=1 /var | sort -hr
```

-> narrowed the disk usage inside `/var` and identified `/var/log` as the major source of the abnormal disk usage

This showed that `/var/log` was responsible for a very large portion of the storage consumption.

At this point, the investigation moved from:

```text
Which filesystem is full?
```

to:

```text
Which directory is consuming the space?
```

---

**Inspected the storage distribution inside `/var/log`**

```bash
sudo du -h --max-depth=1 /var/log | sort -hr
```

-> inspected the directories inside `/var/log` to determine where most of the logging data was stored

The size of `/var/log` was approximately **31 GB**.

This was the first major discovery.

`/var/log` contains logs produced by the operating system, services, applications, security mechanisms, and other components. While log files naturally grow over time, a directory reaching approximately 31 GB on a normal desktop system is a strong indication that something is generating log messages at an abnormal rate.

---

**Listed the largest individual log files**

```bash
sudo ls -lhS /var/log
```

-> listed the files in `/var/log` ordered by size so that unusually large log files could be identified

This helped distinguish between a directory that was large because of many unrelated files and a particular log file that had grown disproportionately large.

---

**Inspected recent system log entries**

```bash
sudo tail /var/log/syslog
```

-> inspected recent system log messages and looked for repeated events that could explain the abnormal growth

This step was important because the size of a log file alone does not reveal why it became large.

A large log can result from many different causes, including:

- normal messages accumulated over a long period
- repeated application errors
- authentication failures
- kernel messages
- service failures
- security-policy violations
- a process generating the same event continuously

Reading the recent entries provided the evidence needed to determine which type of problem was occurring.

---

**Searched for Discord-specific AppArmor messages**

```bash
grep 'snap.discord.discord' /var/log/syslog
```

-> isolated log entries associated with the AppArmor profile of the Discord Snap

`grep` searches text for a specified pattern.

In this case, `snap.discord.discord` corresponds to the AppArmor profile associated with the Discord Snap, allowing the investigation to separate Discord-related security events from the other messages contained in the system log.

The output contained repeated AppArmor denial messages associated with Discord.

The important structure of the messages was conceptually similar to:

```text
apparmor="DENIED"
profile="snap.discord.discord"
operation="ptrace"
comm="Discord"
```

This established a direct connection between the abnormal log growth and the Discord process running inside Snap confinement.

---

**Verified that Discord was installed through Snap**

```bash
snap list
```

-> confirmed that Discord was installed as a Snap package

This distinction was important because the installation mechanism determines how the application is isolated and which security controls are applied around it.

The problematic Discord instance was running through Snap's confinement model.

---

**Removed the Snap version of Discord**

```bash
sudo snap remove discord
```

-> removed the Discord Snap and therefore removed the Snap confinement path that was producing the repeated AppArmor denial events

After removing the Snap installation, a non-Snap installation method could be used instead so that Discord would no longer operate through the same confinement path that was generating the repeated denials.

---

### Investigation and Analysis

After discovering that the filesystem was running out of space, the first objective was not to immediately delete files.

The investigation followed a progressive narrowing process:

```text
Filesystem
    ↓
Directory
    ↓
Log directory
    ↓
Large log file
    ↓
Repeated log message
    ↓
Application
    ↓
Security mechanism
    ↓
Root cause
```

This approach prevents premature cleanup from destroying the evidence needed to identify the cause.

`df -h` established the filesystem-level problem.

`du` was then used to determine which directories were consuming that space.

The investigation narrowed the problem to `/var/log`, which had grown to approximately 31 GB.

At this point, the important question became:

**What process is continuously producing these log entries?**

Instead of deleting `/var/log`, the contents of the logs were inspected.

The repeated entries showed that Discord was triggering AppArmor denials.

This was the key transition from **storage investigation** to **root-cause investigation**.

---

### Root Cause

The root cause was the interaction between the Discord Snap, Snap confinement, and AppArmor.

The mechanism was:

```text
Discord running as a Snap
        ↓
Snap confinement applied
        ↓
AppArmor profile restricts certain operations
        ↓
Discord attempts an operation that is not allowed
        ↓
AppArmor denies the operation
        ↓
The denial is recorded by the Linux logging infrastructure
        ↓
Discord continues attempting the operation
        ↓
Another denial is generated
        ↓
The process repeats
        ↓
The log receives another denial
        ↓
Log files continuously grow
        ↓
/var/log consumes large amounts of disk space
        ↓
The filesystem eventually runs out of free space
```

The important point is that AppArmor was not necessarily malfunctioning.

AppArmor was enforcing a security policy by denying an operation outside the permitted security boundary.

The operational problem came from the **frequency and repetition of those denials**.

A security-policy decision that is individually correct can still contribute to a resource-exhaustion problem if the same denied action occurs continuously and every denial is logged.

---

### Understanding `ptrace`

The AppArmor messages included:

```text
operation="ptrace"
```

`ptrace` is a Linux process-tracing mechanism.

Under permitted conditions, it allows one process to inspect or control aspects of another process.

Discord can perform system or process observation for features such as detecting running applications or games.

When Discord operates under Snap confinement, the AppArmor profile can restrict these operations.

The resulting behavior can therefore be represented as:

```text
Discord:
"Can I perform this process-observation operation?"

AppArmor:
"No. The current policy does not allow it."

Linux logging:
"Record the denial."

Discord:
"Try again."

AppArmor:
"No."

Linux logging:
"Record another denial."

Discord:
"Try again."

AppArmor:
"No."

Linux logging:
"Record another denial."
```

This can continue repeatedly while Discord is running.

The important distinction is that the **security decision** and the **operational consequence** are separate problems.

The security decision was:

```text
AppArmor -> deny operation
```

The operational consequence was:

```text
Repeated denial
    ↓
Repeated log entry
    ↓
Large log growth
    ↓
Disk exhaustion
```

Therefore, the incident demonstrates how a security enforcement mechanism can indirectly contribute to an availability problem.

---

### Possible Snap Interface Mitigation

One possible mitigation for the Discord Snap is:

```bash
snap connect discord:system-observe
```

-> connects the `system-observe` interface to Discord, granting the application the associated system-observation capability and reducing the specific confinement denials

However, this solution introduces a security trade-off.

Granting an additional Snap interface changes what the application is allowed to observe or access.

Therefore, the correct security question is not simply:

```text
"Does this stop the log spam?"
```

It is:

```text
"Is the additional capability necessary, and is granting it acceptable?"
```

Reducing log noise by increasing application privileges is not automatically the best solution.

In this case, removing the problematic Snap package avoided modifying the application's security boundary simply to accommodate behavior that was repeatedly triggering the denials.

---

### Why the Logs Should Not Be Deleted First

A critical troubleshooting principle learned from this incident was:

**Do not immediately delete the evidence when investigating a resource-exhaustion problem.**

The large logs contained the evidence needed to identify the source of the disk consumption.

If the logs had been deleted before being inspected, the investigation could have lost the information required to establish the relationship between:

```text
Discord
    ↓
AppArmor
    ↓
Repeated denials
    ↓
Log growth
    ↓
Disk exhaustion
```

The correct order is therefore:

```text
Investigate
    ↓
Identify the cause
    ↓
Stop the cause
    ↓
Clean up accumulated data
    ↓
Verify that the problem does not return
```

Cleaning the logs is a remediation step.

It is not a substitute for determining why they grew.

---

### What was understood

#### Linux Commands

- `df -h` reports filesystem-level disk usage and is useful for determining which mounted filesystem is becoming full.
- `du` reports disk usage for files and directories and is useful for finding what is consuming space.
- `--max-depth=1` limits `du` output to the immediate directory level being investigated.
- `sort -hr` sorts human-readable size values in reverse order so the largest results appear first.
- `ls -lhS` can be used to display files ordered by size.
- `tail` can be used to inspect recent lines from a log file.
- `grep` can isolate log entries matching a specific process, profile, or text pattern.
- `snap list` can be used to identify applications installed through Snap.
- `snap remove` can be used to remove a Snap package.

#### Security Controls

- Snap applications can operate inside a confinement boundary.
- AppArmor is a Linux security mechanism that restricts what a process is allowed to do according to a defined security profile.
- `apparmor="DENIED"` indicates that an operation was blocked by an AppArmor policy.
- `ptrace` is a Linux process-tracing mechanism that can allow one process to inspect or control another process when permitted.
- Snap interfaces can grant additional capabilities to confined applications.
- Granting additional capabilities changes an application's security boundary and should therefore be evaluated as a security decision rather than treated as a simple configuration change.
- A security control can correctly deny an operation while the resulting enforcement events still create an availability problem.

#### Troubleshooting Methodology

- Start with the broadest layer and progressively narrow the investigation.
- First determine which filesystem is affected.
- Then determine which directory consumes the space.
- Then determine which files consume the directory's space.
- Then inspect the contents of those files.
- Identify the application or process generating the events.
- Identify the system or security mechanism involved.
- Understand the causal chain before performing destructive cleanup.
- Separate the symptom from the mechanism and the root cause.
- Fix the source of the abnormal behavior before cleaning up the accumulated data.
- Verify that the resource usage remains stable after remediation.

The incident can therefore be separated into three levels:

```text
Symptom
    ↓
Disk exhaustion

Immediate mechanism
    ↓
Abnormal growth of /var/log

Root cause
    ↓
Repeated Discord Snap/AppArmor denial events
```

---

### Preventative Measures

In a production environment, preventing the same incident would require controls at several different layers.

**Implement log rotation and retention**

Linux systems should use controlled log rotation rather than allowing logs to grow indefinitely.

For example:

```text
log file
    ↓
rotation
    ↓
compressed historical log
    ↓
retention limit
    ↓
old data automatically removed
```

On Ubuntu, `logrotate` can be used to rotate logs and prevent individual log files from growing without bounds.

In a cloud environment, application and system logs should also be forwarded to a centralized logging platform with explicit retention policies.

For AWS environments, this could involve CloudWatch Logs with an appropriate retention period rather than allowing logs to accumulate indefinitely on an EC2 instance.

---

**Monitor filesystem utilization**

A production system should generate an alert before a filesystem becomes critically full.

For example:

```text
Disk usage
    ↓
70%   -> informational
80%   -> warning
90%   -> critical
95%+  -> emergency response
```

The exact thresholds would depend on the workload, but the important principle is to detect the trend before the filesystem reaches 100%.

For cloud systems, disk-utilization monitoring should be integrated with the platform's observability and alerting system.

---

**Monitor log growth rate**

Monitoring only total disk utilization is useful, but it can identify a problem relatively late.

A stronger control is to monitor the **rate at which logs are being generated**.

For example:

```text
Normal log rate
    ↓
Sudden increase
    ↓
Alert
    ↓
Investigate generating service
```

A sudden increase in AppArmor denials, application errors, authentication failures, or kernel messages can indicate a malfunction or an attack before the disk becomes exhausted.

---

**Centralize logs**

In a production cloud environment, important logs should be forwarded to centralized logging infrastructure.

This provides several advantages:

- log data is not dependent on the local disk remaining healthy
- retention can be managed centrally
- logs can be searched across multiple hosts
- alerts can be generated from log patterns
- a compromised or failed host does not necessarily destroy the only copy of the logs

For AWS workloads, this could involve CloudWatch Logs or another centralized observability platform.

---

**Monitor repeated security denials**

Security-policy denials should not automatically be treated as harmless noise.

A sudden increase in events such as:

```text
AppArmor DENIED
SELinux AVC denial
IAM AccessDenied
Kubernetes admission denial
```

may indicate:

- a broken application
- an incorrect security policy
- an attempted privilege escalation
- a misconfiguration
- malicious activity
- or a combination of security and availability problems

Repeated denials should therefore be observable and investigated when they exceed expected levels.

---

**Apply least privilege when solving policy conflicts**

A production remediation should not simply grant an application additional permissions because doing so makes the errors disappear.

The preferred process is:

```text
Identify denied operation
    ↓
Determine why application needs it
    ↓
Determine whether the operation is actually necessary
    ↓
Evaluate the required privilege
    ↓
Grant the minimum required capability
    ↓
Monitor the result
```

This keeps the remediation aligned with the principle of least privilege.

---

### Vulnerability Type

**Resource Exhaustion / Denial of Service through Excessive Logging**

This incident demonstrates how repeated security-policy events can consume system resources.

The filesystem was the resource that became exhausted, but the underlying chain was:

```text
Application behavior
    ↓
Security-policy denial
    ↓
Repeated logged event
    ↓
Uncontrolled log growth
    ↓
Storage exhaustion
    ↓
Availability impact
```

The vulnerability was therefore not simply that Discord "used too much storage."

The deeper problem was that a continuously repeating application behavior caused continuously repeating security events, and those events were persistently written to disk.

---

### Real World Version

Security engineers regularly investigate resource exhaustion caused by runaway logs, authentication failures, kernel events, application crashes, audit records, security-policy violations, or misconfigured services.

In a production environment, a full filesystem can cause much more than inconvenient loss of storage.

Depending on which filesystem becomes full, applications may be unable to:

- create temporary files
- write databases
- create sockets
- write logs
- rotate logs
- store credentials or state
- install or update packages
- create required runtime files

A full filesystem can therefore cause cascading service failures and become an availability incident.

This is particularly important in cloud environments because ephemeral compute instances, containers, Kubernetes nodes, and application workloads often have finite local storage.

---

### Broader Lesson

The most important lesson from this incident is that troubleshooting should be based on a **chain of evidence** rather than a sequence of random cleanup commands.

The objective is not simply:

```text
"Free some disk space."
```

The objective is:

```text
What consumed the resource?
        ↓
Why did it consume the resource?
        ↓
What process caused the behavior?
        ↓
What system or security mechanism was involved?
        ↓
Why did that mechanism produce the observed result?
        ↓
What change prevents the behavior from recurring?
        ↓
How do we verify that the fix worked?
```

This incident therefore connected several important areas of Linux and security engineering:

```text
Linux filesystems
        ↓
Disk utilization
        ↓
System logging
        ↓
Application behavior
        ↓
Snap confinement
        ↓
AppArmor
        ↓
Security-policy enforcement
        ↓
Resource exhaustion
        ↓
Availability
```

A system is not secure simply because individual security controls are functioning.

Security controls must also be operated in a way that preserves the availability and reliability of the system.

In this incident, the security policy was doing its job by denying an unauthorized operation, but the repeated enforcement events contributed to a severe availability problem through excessive log growth.

That makes the incident a useful example of how **confidentiality, integrity, and availability cannot always be considered independently during security engineering and troubleshooting.**
````
