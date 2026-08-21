# NFS Troubleshooting: A Practical, Interview-Ready Guide

The easiest way to understand NFS troubleshooting is to think in terms of:

```
client → network/RPC → NFS server → filesystem/export
```

---

## 1. Hard vs Soft NFS Mounts

Suppose your Linux server mounts:

```bash
mount -t nfs 10.10.20.50:/data /mnt/data
```

### Hard Mount

A hard mount means: **if the NFS server doesn't respond, keep retrying the I/O request until the server responds.**

```bash
mount -t nfs -o hard 10.10.20.50:/data /mnt/data
```

If an application does:

```bash
cat /mnt/data/file.txt
```

and the NFS server goes down:

```
Application
    |
    v
NFS client
    |
    X
Network
    |
    X
NFS server
```

The process can become stuck waiting for the NFS operation. You might see:

```bash
ps -eo pid,stat,wchan,cmd | grep ' D '
```

```
4217 D nfs_wait_on_request cat /mnt/data/file.txt
```

And:

```bash
dmesg -T | grep -i nfs
```

might show:

```
NFS: server 10.10.20.50 not responding, still trying
NFS: server 10.10.20.50 not responding, still trying
```

Even `kill -9 4217` may not immediately remove the process, because it's stuck in uninterruptible kernel I/O (**D state**).

**Why use hard mounts?** Because for important data, you generally don't want NFS to randomly return an I/O error mid-write and leave the application believing storage operations failed partially. For critical writable data, `hard` is generally preferred.

### Soft Mount

A soft mount tells the NFS client: **try the operation, but after the configured retry/timeout limit, return an error to the application.**

```
Application
     |
     v
NFS client
     |
     X
NFS server unavailable
     |
     v
Retry → Retry → Give up
     |
     v
Return I/O error
```

Example:

```bash
mount -t nfs -o soft,timeo=100,retrans=3 10.10.20.50:/data /mnt/data
```

The application may eventually receive:

```
Input/output error
```

**Why isn't soft always better?** Because now your application has to correctly handle storage failures — and many applications (e.g. databases) aren't designed to do that safely.

> ⚠️ Don't casually change a production NFS mount from hard to soft just to stop processes hanging. For critical writable data, a hard mount is generally safer.

---

## 2. What Is RPC?

This is where NFS troubleshooting gets confusing. **RPC = Remote Procedure Call.**

Instead of "client calls server function directly," think:

```
NFS client
    |
    | "Please perform this operation"
    v
RPC
    |
    v
NFS server
```

Historically, NFS uses several RPC services, especially with NFSv3.

---

## 3. What Is RPC Binding?

An important interview concept. The NFS client needs to know *which port* a service is listening on — that's where **portmapper/rpcbind** comes in. Think of `rpcbind` as a phone directory:

```
Client:  "I need the NFS service."
rpcbind: "NFS is available on port 2049."
Client:  "Thank you." → Connect to NFS service
```

Check RPC registrations:

```bash
rpcinfo -p 10.10.20.50
```

```
program vers proto   port
100000    4   tcp    111   portmapper
100003    3   tcp   2049   nfs
100005    3   tcp  20048   mountd
100021    4   tcp   4045   nlockmgr
```

| Service | Purpose |
|---|---|
| rpcbind / portmapper | Maps RPC service → port |
| nfs | Actual NFS service |
| mountd | Handles mount/export requests, especially NFSv3 |
| lockd | File locking |
| statd | Lock recovery |

---

## 4. NFSv3 vs NFSv4

A very common interview question.

**NFSv3** commonly involves several RPC services:

```
Client
  +----> rpcbind : 111
  +----> mountd
  +----> NFS : 2049
  +----> lockd
  +----> statd
```

If RPC services are broken or firewall rules block them, mounting can fail.

**NFSv4** simplified this significantly — the main NFS service just uses **TCP 2049**:

```
Client
   |
   | TCP 2049
   v
NFS server
```

That's one reason NFSv4 is easier to manage through firewalls.

---

## 5. Common Failure Scenarios

### Scenario 1 — NFS mount fails

```bash
mount -t nfs 10.10.20.50:/data /mnt/data
# mount.nfs: Connection timed out
```

**Troubleshooting steps:**

```bash
ping -c 4 10.10.20.50          # network reachable?
nc -zv 10.10.20.50 2049        # NFS port reachable?
rpcinfo -p 10.10.20.50         # RPC OK? (NFSv3 only)
```

If `rpcinfo` returns `can't contact portmapper: RPC: Remote system error`, investigate rpcbind, firewall, or network connectivity.

### Scenario 2 — `showmount` fails

```bash
showmount -e 10.10.20.50
# clnt_create: RPC: Port mapper failure - Unable to receive
```

```
Client
   |
   | TCP/UDP 111
   X
rpcbind
```

Check:

```bash
rpcinfo -p 10.10.20.50
nc -zv 10.10.20.50 111
```

Possible causes: rpcbind stopped, firewall blocking port 111, network problem, NFS server issue.

> Note: `showmount` is mainly useful for NFSv3-style export discovery; it isn't a definitive NFSv4 health test.

### Scenario 3 — Mount succeeds but accessing files hangs (classic D-state)

```bash
mount | grep /mnt/data
# 10.10.20.50:/data on /mnt/data type nfs4 (rw,hard,vers=4.1)

ls -l /mnt/data   # hangs
```

```bash
ps -eo pid,stat,wchan,cmd | grep ' D '
```

```
4217 D nfs_wait_on_request ls -l /mnt/data
4288 D nfs_wait_on_request cat /mnt/data/file
```

```bash
dmesg -T | grep -i nfs
# NFS: server 10.10.20.50 not responding, still trying
```

**Diagnosis:** the mount exists, but the server isn't responding to NFS requests.

```
Is NFS server alive?
  ├─ No  → Recover server
  └─ Yes
      └─ Is network working?
           ├─ No  → Network troubleshooting
           └─ Yes → Check NFS service / firewall / server filesystem
```

### Scenario 4 — Server up, but filesystem is read-only

```
Read-only file system
```

```bash
mount | grep nfs
df -h
mount
```

The NFS server's underlying filesystem may have gone read-only due to filesystem errors.

```
Application → NFS → Server filesystem → X Read-only
```

The fix is primarily **server-side**, not on the client.

### Scenario 5 — NFS is extremely slow

```bash
nfsstat -c
nfsstat -m
ip -s link           # RX/TX errors, dropped packets
ping -c 20 10.10.20.50
```

Potential causes: network congestion, packet loss, high NFS server load, slow backend storage, MTU problems, DNS delays, or an application generating excessive metadata operations.

### Scenario 6 — NFS mount becomes "stale"

```bash
ls /mnt/data
# ls: cannot access '/mnt/data/file': Stale file handle
```

This can happen when the server-side filesystem/export changes in a way that invalidates file handles (e.g. unmount/remount, filesystem replacement, export changes).

```bash
findmnt /mnt/data
```

A remount may be required — but first make sure no applications are actively using it:

```bash
umount /mnt/data
mount /mnt/data
```

### Scenario 7 — NFS works by IP but not by hostname

```bash
mount -t nfs 10.10.20.50:/data /mnt/data     # works
mount -t nfs nfsserver:/data /mnt/data       # fails
```

```
NFS itself = probably OK
Hostname → DNS / /etc/hosts → X Problem
```

Check:

```bash
getent hosts nfsserver
nslookup nfsserver
cat /etc/resolv.conf
```

Potential causes: DNS failure, wrong DNS record, `/etc/hosts` problem, search domain issue.

### Scenario 8 — Mount works manually but not after reboot

```bash
grep nfs /etc/fstab
```

```
nfsserver:/data /mnt/data nfs4 defaults,_netdev 0 0
```

For boot-time mounts, `_netdev` is commonly used to mark the filesystem as network-dependent. On systemd-managed systems, you may also use appropriate automount/dependency options.

### Scenario 9 — Only one client has the problem

```
Client A --> NFS --> works
Client B --> NFS --> works
Client C --> NFS --> FAILS
```

Don't immediately blame the NFS server. Compare Client C:

```bash
ip addr
ip route
cat /etc/resolv.conf
mount | grep nfs
nfsstat -m
ping NFS_SERVER
nc -zv NFS_SERVER 2049
```

Possibilities: client firewall, wrong route, wrong DNS, MTU problem, different NFS version, incorrect mount options, network interface problem.

### Scenario 10 — All clients suddenly fail

```
Client A ----\
Client B -----+----> NFS Server
Client C ----/
```

All failing simultaneously is a strong indicator of a **server-side or shared-network problem**.

```bash
ping NFS_SERVER
systemctl status nfs-server      # on the server
ss -lntp | grep 2049
journalctl -u nfs-server
journalctl -k | grep -i nfs
df -h
df -i
```

A surprisingly common culprit: the NFS server's underlying filesystem is full.

---

## 6. The Troubleshooting Decision Tree

```
NFS problem
     |
     v
Can I ping the server?
     +---- NO ----> Network / routing / server down
     YES
     v
Can I reach TCP 2049?
     +---- NO ----> NFS service / firewall / network
     YES
     v
What NFS version?
     +---- NFSv3 ----> Check RPC/rpcbind/mountd
     +---- NFSv4 ----> Focus on TCP 2049/NFSv4/service
     v
Can I mount?
     +---- NO ----> Export / permissions / RPC / network
     YES
     v
Can I access files?
     +---- NO ----> Server filesystem / permissions / stale handle
     YES
     v
Is it slow?
     +---- YES ----> Network / retransmits / server/storage performance
     NO
     v
Is application hanging?
     +---- YES ----> Check D state / hard mount / NFS server response
```

---

## 7. The 5 Things I'd Say in an Interview

If given an NFS outage scenario, structure the answer around these:

1. **Mount problem?** → `findmnt`, `mount`, NFS version/options.
2. **Network problem?** → `ping`, `ip route`, `nc -zv server 2049`, packet loss/MTU.
3. **RPC problem?** (NFSv3) → `rpcinfo -p`, `showmount -e`, rpcbind, mountd.
4. **Server problem?** → `systemctl status nfs-server`, `journalctl`, `df -h`, backend filesystem/storage.
5. **Application hanging?** → `ps`, wchan, `/proc/<pid>/stack`, D state, check for `NFS: server not responding`.

### Quick reference: Hard vs Soft

- **Hard NFS:** keeps retrying; safer for data integrity, but applications can hang in D state when the server disappears.
- **Soft NFS:** eventually returns an error; prevents indefinite waits, but can expose I/O errors to applications — risky for important writable data.

### Quick reference: RPC binding

`rpcbind` is essentially the directory that tells an RPC client which port a particular RPC service is using. For NFSv3, it's a key part of the connection process; NFSv4 relies much more directly on TCP/2049.
