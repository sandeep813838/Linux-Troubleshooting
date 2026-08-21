# Linux Troubleshooting: High Load Average but Low CPU Utilization — Explained

A real production incident walkthrough: RHEL 6 server running JBoss, high CPU during a DB2 → PostgreSQL migration. This is the full step-by-step, with every command and how to read its output.

---

## The Question

> "The server is very slow, but CPU utilization is low and the load average is high. What is happening?"

To answer this properly, you need to understand four concepts and how they relate:

**CPU Utilization → Load Average → Process/Thread → I/O**

---

## 1. What is CPU Utilization?

CPU utilization tells you how busy the CPU actually is.

```bash
top
```

Sample output:

```text
%Cpu(s): 70.0 us, 10.0 sy, 0.0 ni, 18.0 id, 2.0 wa
```

| Field | Meaning |
|-------|---------|
| `us` | CPU used by user applications |
| `sy` | CPU used by the kernel |
| `id` | CPU idle |
| `wa` | CPU waiting on I/O |

High `us`/`sy` and low `id` = the CPU is genuinely doing work. But **CPU utilization alone doesn't tell you if the server is healthy** — a server can be CPU-idle and still unusably slow.

---

## 2. What is I/O?

I/O = Input/Output — any read or write between the CPU/memory and another resource:

```text
Application → Read/Write → Disk
Application → Read → NFS → Remote storage
Application → Database → Network → Storage
```

I/O isn't just local disk. It includes SAN/storage, NFS, network, filesystems, and database storage.

---

## 3. Why I/O Can Make a Server Slow

```text
Java thread → "Give me this data" → Storage (slow) → Thread waits
```

While a thread waits on slow storage, the CPU often has nothing useful to execute for it. Result:

```text
CPU utilization → LOW
Server response → VERY SLOW
```

**Low CPU does not automatically mean the server is healthy.**

---

## 4. What is Load Average?

```bash
uptime
```

Sample output:

```text
load average: 18.00, 15.50, 12.20
```

These are the 1-minute, 5-minute, and 15-minute averages of:
- Tasks ready to run and waiting for CPU, **plus**
- Tasks in uninterruptible sleep (commonly waiting on I/O)

**High load average does not automatically mean high CPU utilization** — because load counts I/O-blocked tasks too.

---

## 5. Process vs Thread

- **Process** = a running application (e.g., the JBoss JVM)
- **Thread** = a unit of execution inside that process

```text
Process = Company
Thread  = Worker
```

A JBoss JVM can run hundreds or thousands of threads simultaneously.

---

## Real Production Incident

**Environment:** RHEL 6 server running JBoss
**Symptom:** Sudden high CPU and application performance degradation
**Context:** In progress — a DB2 → PostgreSQL migration

Instead of immediately restarting JBoss, I investigated layer by layer.

### Step 1 — Check Load Average

```bash
uptime
```

```text
load average: 12.5, 10.8, 9.2
```

```bash
nproc
```

```text
8
```

A load of 12+ on an 8-core box is worth investigating further.

### Step 2 — Check CPU

```bash
top
```

Goal: determine whether the CPU is actually overloaded, or whether processes are waiting somewhere else (I/O).

**Decision point:**
- If CPU is low but load is high → suspect I/O, NFS, SAN, disk latency, blocked processes, memory/swap (see the "If CPU Is Low" section below).
- If CPU is high → drill into the process itself (this incident).

In this case, the JBoss JVM was clearly consuming significant CPU, so we drilled into the JVM.

### Step 3 — Identify the JBoss Process

```bash
ps -ef | grep java
```

or find it directly in `top`. Example result:

```text
JBoss PID = 12345
```

At this point we know JBoss is consuming CPU — but not *which thread inside it* is responsible.

### Step 4 — Look at Individual Threads

```bash
top -H -p 12345
```

`-H` tells `top` to break down CPU usage per-thread instead of per-process.

Sample output:

```text
PID      %CPU
12451    85.0
12452    82.0
12453    78.0
12454     2.0
```

The IDs shown are Linux thread IDs (TIDs). We now have specific hot threads to investigate: `12451`, `12452`, `12453`.

### Step 5 — Convert Thread ID to Hex

Java thread dumps identify native thread IDs in hexadecimal, but `top -H` shows decimal. Convert:

```bash
printf '%x\n' 12451
```

```text
3073
```

```text
Linux TID → 12451
Hex       → 0x3073
```

Repeat for each hot thread ID you want to trace.

### Step 6 — Take a Java Thread Dump

```bash
jstack 12345 > /tmp/jstack.out
```

Then search the dump for the hex ID:

```bash
grep -A 30 "nid=0x3073" /tmp/jstack.out
```

This chain connects everything:

```text
top -H → high-CPU thread → Linux TID → decimal→hex → jstack → nid=0x3073 → exact Java thread + its stack trace
```

### Step 7 — What We Found

The stack trace pointed to the **JBoss datasource / connection-pool / database connectivity path**.

Because this was during the DB2 → PostgreSQL migration, some database connections were failing. Multiple application threads were repeatedly retrying to obtain a connection:

```text
Thread → request DB connection → fails → retry → fails → retry → ...
(many threads doing this simultaneously)
→ CPU increases → JBoss becomes slow
```

This is a **connection-pool retry storm** — a busy-retry loop across many threads, not a single slow query.

**Key insight:** the CPU spike was the *symptom*, not the *root cause*. The real fix was in:
- JBoss datasource configuration
- PostgreSQL connectivity/network path
- JDBC driver configuration
- Connection pool sizing and retry/backoff behavior

After correcting the underlying connection issue, CPU returned to normal on the next monitoring pass.

---

## If CPU Is Low Instead (High Load + Low CPU)

Suppose instead we'd seen:

```text
Load average → 20
CPU utilization → 10%
```

This points to I/O, not CPU-bound work.

### Check vmstat

```bash
vmstat 1 5
```

Watch:
- `r` — runnable processes
- `b` — processes blocked, waiting on I/O
- `wa` — % CPU time spent waiting on I/O

### Check iostat

```bash
iostat -xz 1 5
```

Watch:
- `await` — average I/O request wait time (ms)
- `%util` — how saturated the device is

High `await`/`%util` = disk or storage is the bottleneck, even though CPU looks idle.

### Find Processes Stuck in I/O Wait

```bash
ps -eo state,pid,ppid,cmd | grep '^D'
```

`D` = uninterruptible sleep, almost always waiting on I/O.

```text
Application → read from NFS → NFS server slow → process waits
→ process enters D state → load average increases → CPU stays idle
```

This is the classic **high load, low CPU** signature.

### If NFS Is Involved

```bash
nfsstat -m
mount | grep nfs
journalctl -k | grep -Ei 'nfs|timeout|error'
```

This helps determine where the bottleneck actually sits:

```text
Linux client → network → NFS server → backend storage
```

NFS or SAN latency can make an application look completely stuck while the server's own CPU is nearly idle — this catches a lot of people off guard.

---

## Troubleshooting Flow (Decision Tree)

```text
                 Server Slow
                      ↓
                   uptime
                      ↓
             Check Load Average
                      ↓
                    top
                      ↓
        ┌─────────────┴─────────────┐
        ↓                           ↓
   CPU High                     CPU Low
        ↓                           ↓
  Find process                Investigate I/O
        ↓                           ↓
   top -H -p <PID>          vmstat / iostat
        ↓                           ↓
   Find hot thread          Disk / NFS / SAN
        ↓                           ↓
   printf '%x' <TID>        Check D-state processes
        ↓                           ↓
   jstack <PID>              Root cause in storage
        ↓                     /network layer
   Find nid=0x<hex>
        ↓
   Read stack trace →
   Find application root cause
```

---

## Command Reference

**General server health**
```bash
uptime
top
nproc
free -h
```

**CPU / process investigation**
```bash
ps -ef
top
top -H -p <PID>
```

**Java/JBoss investigation**
```bash
jstack <PID> > /tmp/jstack.out
printf '%x\n' <TID>
grep -A 30 "nid=0x<hex>" /tmp/jstack.out
```

**I/O investigation**
```bash
vmstat 1 5
iostat -xz 1 5
```

**Blocked processes**
```bash
ps -eo state,pid,ppid,cmd | grep '^D'
```

**NFS investigation**
```bash
nfsstat -m
mount | grep nfs
journalctl -k | grep -Ei 'nfs|timeout|error'
```

---

## The Biggest Lesson

Don't troubleshoot based on the first symptom you see.

❌ **High CPU → restart the app**
✅ **High CPU → identify process → identify thread → understand what the thread is doing → find root cause**

❌ **High load + low CPU → "well, CPU is fine"**
✅ **High load + low CPU → investigate I/O → identify blocked processes → check disk/NFS/SAN/network/storage**

### The mental model, one line each

| Command | Answers |
|---|---|
| `uptime` | How high is the load? |
| `top` | Is CPU actually busy? |
| `ps` / `top` | Which process? |
| `top -H` | Which thread? |
| `jstack` | What is the Java thread doing? |
| `vmstat` / `iostat` | Is I/O causing the slowness? |
| Logs + app/DB checks | What is the actual root cause? |

That's the difference between running Linux commands and actually troubleshooting a production server.

---

`#Linux` `#RHEL` `#LinuxAdministration` `#JBoss` `#Java` `#Troubleshooting` `#SystemAdministration` `#PerformanceTuning` `#IOTroubleshooting` `#NFS` `#DevOps` `#SRE` `#ProductionSupport` `#IncidentManagement`
