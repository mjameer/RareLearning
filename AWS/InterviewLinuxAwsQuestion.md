# Linux Fundamentals — Full Q&A (AWS SysDE Prep)

## ⚡ Command Cheat Sheet (memorize this page first — everything else is context)

If you only have time to drill one thing before the 28th, drill this table.
Every command that came up anywhere in this prep, grouped by what problem it solves.

### The big-picture triage trio (run these first, always)
```
top                              # live view: CPU, memory, load average, process list
free -h                          # RAM + swap usage, human-readable
df -h                            # disk space per filesystem
```

### "Which process is the problem?"
```
ps aux --sort=-%cpu | head -10   # top 10 CPU users, one-time snapshot
ps aux --sort=-%mem | head -10   # top 10 memory users
top -H -p <pid>                  # per-thread CPU view of one process
nproc                            # how many cores — needed to interpret %CPU correctly
```

### "What is that specific process actually doing?"
```
strace -p <pid>                  # live syscall trace — spinning? blocked? working?
cat /proc/<pid>/status           # memory detail (VmRSS = real RAM used)
cat /proc/<pid>/stack            # kernel-level stack, if process is in D state
lsof -p <pid>                    # files/sockets a process has open
perf top -p <pid>                # function-level CPU profiling (if available)
```

### Hung / deadlocked process
```
ps aux | grep <pid>              # check STAT column: D = stuck on I/O, S = normal idle
jstack <pid> > dump.txt          # JVM thread dump (bundled with JDK)
kill -3 <pid>                    # JVM thread dump alternative if jstack unavailable
py-spy dump --pid <pid>          # Python thread dump (pip install py-spy first)
```

### Memory-specific
```
free -h                          # used vs. available vs. swap
cat /proc/<pid>/status | grep -E "VmRSS|VmSize"   # is one process leaking?
dmesg | grep -i "out of memory"  # did the OOM killer already fire?
journalctl -k | grep -i oom      # same, via systemd's journal
vmstat 1 5                       # watch si/so columns for active swapping (thrashing check)
cat /proc/sys/vm/swappiness      # check current swappiness value
sysctl vm.swappiness=10          # change it (temporary; edit /etc/sysctl.conf to persist)
```

### Disk-specific
```
df -h                            # space per filesystem
df -i                            # inode usage (separate from byte usage!)
du -sh /* 2>/dev/null | sort -rh | head -10   # find largest top-level directories
ls -lhS /var/log | head -5       # find largest individual files, biggest first
truncate -s 0 file.log           # empty a log file live, without restarting the process
```

### Network-specific
```
ss -tulpn                        # listening ports + established connections + owning process
ss -tn state established '( dport = :5432 )' | wc -l   # count connections to a specific port
iostat -x 1 5                    # disk I/O stats, 1-second interval, 5 samples
ping -c 4 <host>                 # basic reachability check
traceroute <host>                # hop-by-hop path to a host
```

### Process / permission basics
```
chmod 755 file                   # rwxr-xr-x — set permissions (owner/group/other)
kill <pid>                       # graceful SIGTERM
kill -9 <pid>                    # forceful, no cleanup — last resort
systemctl status <service>       # check a daemon's status
journalctl -u <service>          # read that service's logs
```

### Text/log processing (your bread-and-butter for the coding round)
```
grep -i "pattern" file           # case-insensitive search
awk '{print $1}' file            # extract a field
sed 's/old/new/' file            # find-and-replace in a stream
find /path -size +200M           # find files over a given size
tail -f file.log                 # live-follow a growing file
```

### One-off / trivia commands that showed up
```
lscpu                            # detailed CPU info (cores, sockets, cache)
cat /proc/cpuinfo                # raw CPU info (what nproc/lscpu read from)
cat /proc/meminfo                # raw memory info
cat /proc/loadavg                # raw load average numbers
uptime                           # quick load average + uptime
apt install --reinstall coreutils   # fixes a broken chmod/ls/etc. binary
```

**How to actually memorize this in a few days:** don't try to memorize the
table itself — instead, for each of the 5 troubleshooting scenarios in
Section 5 below, say the *sequence* out loud from memory (e.g., "CPU spike →
nproc, then ps aux sorted by cpu, then strace"). The commands stick much
better attached to a story than as a flat list.

---

Everything covered in this session, organized by the original question list, with
every follow-up question and command explanation folded in under the relevant topic.

---

## 1. "Stuff about Linux commands, such as what `chmod` does, etc."

**The permission model:** every file has three permission sets — **owner, group,
other** — each with **read (r), write (w), execute (x)**.

- Symbolic form: `-rwxr-xr--`
- Octal form: each `rwx` triplet sums to a digit — r=4, w=2, x=1. So `rwx`=7,
  `r-x`=5, `r--`=4 → `-rwxr-xr--` = **754**.

`chmod 755 file` → owner=rwx(7), group=r-x(5), other=r-x(5) — the standard
"executable, readable by everyone" permission.

**One-liner for the interview:** *"chmod changes a file's read/write/execute
permissions for owner, group, and other — either symbolically (`chmod u+x
file`) or with a 3-digit octal number where each digit sums r=4, w=2, x=1."*

**`coreutils`, since it came up:** `chmod` isn't a shell built-in — it's one of
~100 separate binaries (`ls`, `cp`, `mv`, `rm`, `cat`, `du`, `df`, etc.) bundled
in the GNU **coreutils** package. That's exactly why `apt install --reinstall
coreutils` can fix a broken `chmod` — reinstalling the package overwrites the
`/bin/chmod` file with a correct-permission copy, without ever needing to run
the broken binary itself.

---

## 2. The trick question: `cd /bin; chmod 644 chmod` — how do you fix it?

`644` = `rw-r--r--` — **no execute bit anywhere**. You just made `chmod`
non-executable, so running `chmod 755 chmod` to fix it fails with "Permission
denied" — you can't execute `chmod` to repair `chmod`.

**Three valid fixes, in order of practicality:**

**Option 1 — reinstall the package (the pragmatic answer):**
```
apt install --reinstall coreutils     # Debian/Ubuntu
yum reinstall coreutils               # RHEL/CentOS
```

**Option 2 — use a different program that does the same job internally:**
```
python3 -c "import os; os.chmod('/bin/chmod', 0o755)"
```
- `python3` has its own execute permission — untouched by what you did to `chmod`.
- `-c "..."` tells Python to run this string as code directly, not from a `.py` file.
- `os.chmod(...)` calls the same underlying kernel **system call** that the
  command-line `chmod` tool normally calls — Python just invokes it directly.
- `0o755` is Python's octal literal syntax — the `0o` prefix means "read as
  base-8," giving the same 755 permission value.
- **Key insight:** the command-line `chmod` tool and `os.chmod()` are two
  different *programs* that both call the same kernel-level `chmod()` syscall.
  Breaking the tool doesn't touch the syscall itself.

**Option 3 — the classic sysadmin trick, invoking the dynamic linker directly:**
```
/lib64/ld-linux-x86-64.so.2 /bin/chmod 755 /bin/chmod
```
- The kernel only refuses to *execute* a non-executable file. Most binaries
  (including `chmod`) rely on a dynamic linker/loader (`ld.so`) to load their
  shared libraries before running — this normally happens invisibly.
- Here, you invoke `ld.so` **yourself**, directly, and hand it `chmod` as an
  argument. The kernel's permission check now only applies to `ld.so` (which
  you never touched) — not to `chmod`.
- `ld.so`, once running, doesn't ask the kernel to "execute" `/bin/chmod" as a
  new process — it just **opens it as a data file** (using ordinary *read*
  access, still intact at `644`), loads its machine code into memory itself,
  and jumps execution into that code. From the kernel's point of view, only
  one process (`ld.so`) ever ran.
- **Java analogy:** `java -jar mytool.jar` — the JVM is what the OS actually
  executes; the jar doesn't need to be independently executable, because the
  JVM just reads it as data and interprets it. `ld.so` does the same thing to
  `/bin/chmod`.
- Path varies by system — find yours with `ls /lib64/ld-linux*` or
  `readelf -l /bin/ls | grep interpreter`.

**The plain-English story version, if the technical explanation didn't fully land:**

Imagine a library with a rule: *"a book can only be picked up off the shelf
by its own reader-robot, and only if that book has a 'pick-me-up' sticker on
it."* You accidentally peeled the sticker off one book (`chmod`) — so its own
robot can no longer grab it and start reading.

But you also have a **translator** (`ld.so`) whose entire job, for every book
in the library, is to pick up whatever book you hand them and read it out
loud. The translator has their *own* sticker, glued to themselves — they
never needed the book's sticker in the first place. So instead of asking
`chmod`'s own robot to fetch itself (which fails, no sticker), you personally
carry the `chmod` book over to the translator and say "read this one for
me." The translator doesn't check the book's sticker — only their own — so
this works fine.

Mapped onto the real commands: `ld.so` still has its own execute permission
intact (you never touched that file). Running `ld.so /bin/chmod 755
/bin/chmod` tells `ld.so` — not the kernel — to open `/bin/chmod` and read
its contents; **reading** a file only needs *read* permission, which
`chmod` still has at `644`. Since `chmod`'s job, once running, is "go change
some file's permission bits," and you told it to fix its own bits back to
`755`, running it this one time via the translator is what restores its
execute permission for good.

**The one thing to remember:** the kernel's rule only cares about *how a
file gets launched* — not what's inside it. `chmod` without its execute bit
can't launch *itself*, but it can absolutely be *carried and read* by
something else (`ld.so`) that already has a working execute bit of its own.
"Reading a file" and "launching a file" are two separate permissions, and
only one of them was actually broken.

---

## 3. What's in `/proc`?

A **virtual filesystem** — not real files on disk. The kernel generates its
content live, in memory, purely so you can read kernel/process state with
ordinary tools (`cat`, `ls`) instead of special APIs.

**Two categories:**
- **Per-process** — `/proc/<pid>/status`, `/proc/<pid>/cmdline`,
  `/proc/<pid>/fd/`, `/proc/<pid>/environ`, `/proc/<pid>/maps`,
  `/proc/self` (symlink to your own process)
- **System-wide** — `/proc/cpuinfo`, `/proc/meminfo`, `/proc/loadavg`,
  `/proc/uptime`, `/proc/mounts`, `/proc/sys/vm/swappiness`

**Interview hook:** *"where does `top` get its data from?"* → `/proc`. Tools
like `top`, `ps`, `free`, and `nproc` are all just reading and formatting files
from `/proc`.

**"So is it always empty?" — No.** Two different ideas got conflated:
- **Content when you read it:** full of real, current data (`cat
  /proc/meminfo` shows real numbers).
- **File size on disk (`ls -l /proc/meminfo` shows `0`):** there's no
  persistent storage backing it — the kernel generates content only at the
  exact moment you open and read the file, so the size field is meaningless.
- **Analogy:** it's like a live API endpoint exposed through the filesystem
  instead of HTTP — nobody calls a `GET` endpoint "empty" just because there's
  no static file behind it.

---

## 4. Linux distros, processes, memory management, and disk management

**Distros:** the kernel + package manager + userland tools, bundled
differently by vendor. Debian/Ubuntu family (`apt`/`dpkg`, `.deb`) vs.
RHEL/CentOS/Fedora family (`yum`/`dnf`, `.rpm`) — **Amazon Linux** is
RHEL-derived and AWS's own distro, worth knowing by name for this interview.

**Processes:** a running program instance, with its own memory space, PID, and
entry in the process table. Every process except PID 1 (`init`/`systemd`) has
a parent — this is why `ps` output is tree-shaped, and why zombies get
reparented to PID 1 when their original parent dies. States: `R` (running),
`S` (sleeping), `T` (stopped), `Z` (zombie).

**Memory management:** the kernel gives each process a *virtual* address
space and translates it to physical RAM (or swap) transparently — this
translation layer is what makes swapping, thrashing, and OOM-killing possible.

**Disk management:** partition (`fdisk`/`parted`) → format with a filesystem
(`mkfs`, e.g. `ext4`) → mount it into the directory tree (`mount`, persisted
via `fstab`) → monitor (`df -h` for space, `du -sh` for per-directory usage,
`lsblk` for the block devices themselves). A disk can be "full" in either
bytes (`df -h`) or inodes (`df -i`) — check both.

---

## 5. In-depth Linux process & resource troubleshooting

This category is scenario-based — they hand you a symptom and watch your
**process**, not just your final answer. General framework: get the big
picture (`top`) → narrow to the constrained resource → identify the specific
process → dig into that process → check logs → form and verify a hypothesis
→ act.

### 5a. "A process is using 100% CPU — how do you investigate?"

**Step 1 — scope it:** `nproc` tells you how many cores the box has, because
`top`'s `%CPU` is **per-core**, not capped at 100 for the whole machine. A
process can show 200%, 3200%, etc., if it has enough busy threads spread
across enough cores.
- 1-core box, process at 100% → the *entire machine* is saturated — a real emergency.
- 32-core box, process at 100% → only ~3% of total capacity (100 ÷ 3200) —
  barely worth investigating; the other 31 cores are idle.
- Formula: `% of total capacity = process %CPU ÷ (nproc × 100)`.
- A **single thread** caps at 100% (one thread = one core at a time), but a
  **process** can exceed 100% because multiple threads can genuinely run on
  different cores simultaneously — confirm with `top -H -p <pid>` (per-thread view).

**Step 2 — identify the process, sorted:**
```
ps aux --sort=-%cpu | head -10
```
- `ps aux` = all processes, detailed format (`a`=all users, `u`=user-oriented
  format, `x`=include processes with no terminal, e.g. daemons)
- `--sort=-%cpu` = sort by CPU column; the `-` means **descending** (biggest
  first) — this dash means "reverse direction," a totally different use of `-`
  than in `head -10` below
- `| head -10` = pipe output into `head`, keep only the first 10 lines;
  `-10` is shorthand for `-n 10` ("give me 10 lines") — this dash is just
  option syntax, unrelated to sorting
- Columns worth knowing: `%CPU`, `%MEM`, `VSZ` (virtual memory reserved),
  `RSS` (actual physical RAM used), `STAT` (process state), `TIME` (total
  CPU time accumulated over the process's life, not a percentage)

**Step 3 — check *why*, with `strace -p <pid>`:**
`strace` attaches to a running process (`-p <pid>` targets that PID) and
prints every **system call** it makes live — every time a program needs
something from the kernel (read a file, write a socket, allocate memory,
sleep), that's a syscall.
- **Busy-spinning (bug):** same syscall repeating instantly, thousands of
  times, getting nothing back (e.g. `read()` returning 0 over and over) —
  the process is polling instead of blocking efficiently.
- **Blocked/waiting (I/O-bound, not CPU-bound):** long silent gaps between
  calls (e.g. `epoll_wait` with multi-second pauses) — genuinely idle, not
  actually burning CPU despite what `top` suggested.
- **Genuine work:** varied syscalls, changing data, logical sequence (read
  request → open file → read → close → write response) — probably healthy,
  just busy.

**Step 4 — per-thread view if multi-threaded:** `top -H -p <pid>`.

**Step 5 — function-level profiling if available:** `perf top -p <pid>`.

**Step 6 — sanity-check against expected load** (traffic spike? scheduled
batch job?) via logs (`journalctl -u <service>`) before assuming it's a bug.

**Step 7 — act:** `kill <pid>` (graceful `SIGTERM`) before `kill -9`
(forceful); for recurring issues, mention rate limiting, circuit breakers, or
cgroup CPU limits so one process can't starve the host.

### 5b. "The server is out of memory — walk me through your steps."

**Step 1 — confirm scope:**
```
free -h
```
Read `available` (kernel's estimate of memory it could hand a new process
without swapping) as the real signal — not raw `used`, since high `used` is
often just healthy page cache (`buff/cache`) that the kernel would evict on
demand. Swap actively filling up alongside low `available` = real trouble.

**Step 2 — find the responsible process:**
```
ps aux --sort=-%mem | head -10
```
Same tool as CPU triage, sorted on `%MEM` instead. `RSS` = actual physical
RAM used.

**Step 3 — leak vs. legitimately large:**
```
cat /proc/<pid>/status | grep -E "VmRSS|VmSize"
```
Run this more than once, minutes apart. Steadily climbing `VmRSS` with no
plateau = memory leak. Climbs once, then flat = probably just a
memory-hungry-but-healthy workload (e.g. a JVM with a large configured heap).

**Step 4 — check if the OOM killer already fired:**
```
dmesg | grep -i "out of memory"
```
- `dmesg` ("display message") prints the **kernel ring buffer** — low-level
  kernel/driver/hardware events, including OOM-killer activity, since that's
  a kernel-level decision, not something any individual app logs.
- `grep -i "out of memory"` filters that (potentially thousands of lines)
  down to just the matching line; `-i` = case-insensitive.
- `command | grep -i "thing"` (piping noisy output into a filter) is a
  general technique used constantly — same pattern as `ps aux | grep`.
- Checking this explains outages that otherwise look mysterious — a process
  "just disappearing" with no app-level crash log.

**Step 5 — decide:** leak → profile (heap dump for JVM, `tracemalloc`/
`py-spy` for Python). Capacity problem (several processes, each reasonable,
together exceeding RAM) → instance is undersized, consider right-sizing.
Legitimately large and stable → not a bug; check the app's own configured
limits (`-Xmx`, container memory limits) before assuming otherwise.

**Step 6 — mitigate now, fix root cause after:** restart the confirmed
leaking process to restore service; separately, add memory alerts/limits
(cgroup limits, CloudWatch alarms) so you're paged before the next critical point.

### 5c. "A process is unresponsive/hung — what do you check?"

**Step 1 — process state:**
```
ps aux | grep <pid>
```
Read the `STAT` column: `D` = **uninterruptible sleep**, waiting on I/O
(disk, NFS, a device) — can't even be killed while in this state, and is
almost always the sign of genuine "stuck." `S` = normal idle waiting, not hung.

**Step 2 — if `D`, find what it's stuck on:**
```
cat /proc/<pid>/stack
```
Shows the kernel-level call stack — pattern-match for disk vs. network
filesystem vs. lock-related function names, not full technical understanding needed.

**Step 3 — if `S` but still seems hung, attach `strace`:** total, indefinite
silence with no syscalls at all suggests a deadlock or infinite loop entirely
inside the application's own code, rather than a kernel-level I/O block.

**Step 4 — get an application-level stack trace:**
```
jstack <pid> > threaddump.txt      # JVM — bundled with the JDK, not a separate install
py-spy dump --pid <pid>            # Python — needs `pip install py-spy` (not bundled)
```
Look for threads `BLOCKED` on a lock that's never released, or `WAITING`
with nothing that will ever notify them.
- `jstack` needs the JDK, not just the JRE — if a minimal production image
  only ships the JRE, `jstack` won't be there; `kill -3 <pid>` makes a
  running JVM dump its own thread stacks to its log output instead, no
  extra tool needed.

**Step 5 — check for deadlock specifically:** one thread `WAITING` on lock A
while holding lock B, another thread holding A and waiting on B — classic
deadlock, unrecoverable without a kill/restart.

**Step 6 — decide:** `D` state waiting on disk/NFS → may self-resolve if
storage recovers; check storage health in parallel. Confirmed deadlock →
kill (`kill -9` if unresponsive to `SIGTERM`) and restart, then fix lock
ordering in code. Just slow, not hung → compare against Step 3's expectation.

### 5d. "Disk is full but you don't know why — how do you find the cause?"

**Step 1 — which filesystem:**
```
df -h
```

**Step 2 — also check inodes (a different exhaustion mode):**
```
df -i
```
`IUse% = 100%` while bytes still show free space = "too many small files,"
needing a different fix (delete files) than a byte-space problem (delete big files).

**Step 3 — find what's large:**
```
du -sh /* 2>/dev/null | sort -rh | head -10
```
- `du -sh /*` = summarize size per top-level directory (`-s`), human-readable (`-h`)
- `2>/dev/null` = discard permission-denied error messages
- `sort -rh` = sort human-readable sizes correctly (biggest first) — plain
  `sort` would wrongly treat "15G" as less than "3G" as plain text
- `head -10` = top 10 offenders

**Step 4 — drill deeper, repeat one level down** (`du -sh /var/* ...`) until
you find the actual large file(s) — in practice, almost always `/var/log`.

**Step 5 — confirm and fix:**
```
ls -lhS /var/log | head -5
```
`-S` = sort by size. Fix: truncate the offending log live
(`> /var/log/app-debug.log`), then find why log rotation isn't cleaning it up
(`logrotate` config) and whether the app is logging too verbosely.

### 5e. "An application is slow but CPU/memory look fine — now what?"

The trap: `top` says everything's fine, so you have to know what to check *next*.

**Step 1 — rule out disk I/O:**
```
iostat -x 1 5
```
`1 5` = interval and count: refresh every **1 second**, **5 times total**,
then stop. The very first block `iostat` prints (no args at all) is an
average since boot — nearly useless for "right now" — so you always want 2+
samples and discard the first. `%util` near 100 = disk saturated even though
nothing shows up in CPU/memory. `await` high = requests queuing.

**Step 2 — check network connections:**
```
ss -tulpn
```
- `-t`/`-u` = TCP/UDP, `-l` = listening sockets, `-p` = show owning process, `-n` = numeric (skip DNS resolution)
- `LISTEN` line = process waiting to accept new connections (`0.0.0.0` = any
  interface on this machine; `0.0.0.0:*` on the remote side = nobody's
  connected yet)
- `ESTAB` line = an actual open connection right now, with real local/remote
  IP:port pairs on both sides
- Confirms the process is accepting traffic on the right port, and roughly
  how many clients are connected.

**Step 3 — check if it's waiting on a downstream dependency:**
```
ss -tn state established '( dport = :5432 )' | wc -l
```
- `state established` = only fully-open connections, skip `LISTEN` etc.
- `'( dport = :5432 )'` = `ss`'s own filter syntax, `dport` = destination
  port — here, Postgres's default port
- `| wc -l` = "word count, lines" — with `-l`, just counts matching lines,
  giving one number (e.g. `248`)
- If that number is far above your connection pool's configured max, or
  growing over time, that points to a connection leak or misconfigured pooling.

**Step 4 — check for internal lock contention:** same `jstack`/`py-spy`
tools as the hung-process case — look for many `RUNNABLE` threads queued or
`BLOCKED` on the same lock, a code-level bottleneck rather than an OS-level one.

**Step 5 — check the network path, if distributed:** `ping` / `traceroute`
to a downstream service — the slowness might be a hop away, not in this
process at all.

---

## 6. "What basic Linux tools have you used?"

Recognize these by name and purpose — flags don't need to be memorized:

| Command | Purpose |
|---|---|
| `ps aux` / `top` / `htop` | list processes, CPU/mem usage (live for top/htop, one-shot for ps) |
| `df -h` / `du -sh` | disk space: per-filesystem / per-directory |
| `free -h` | RAM + swap usage |
| `ss -tulpn` (modern) / `netstat -tulpn` (older) | listening ports, active connections, owning process |
| `grep`, `awk`, `sed` | text search / field extraction / stream editing — the log-parsing toolkit |
| `find /path -size +200M` | find files by size |
| `kill`, `kill -9` | terminate a process, graceful vs. forced |
| `systemctl status <service>` | check/manage a systemd service |
| `journalctl -u <service>` / `journalctl -k` | read a service's logs / kernel logs |
| `lsof -p <pid>` | files/sockets a process has open |
| `strace -p <pid>` | trace a running process's syscalls |
| `dmesg` | kernel ring buffer — hardware, driver, OOM-killer messages |
| `iostat -x 1 5` | disk I/O stats, sampled over time |
| `nproc` | number of CPU cores |
| `tail -f file.log` | live-follow a growing file |

---

## 7. "Linux complex commands" — the pipe pattern

The recurring idiom worth internalizing: **`noisy_command | filter_command`**
— run something that produces a lot of output, then pipe (`|`) it into a
second command that narrows it down. Examples covered: `ps aux --sort=-%cpu |
head -10`, `dmesg | grep -i "out of memory"`, `ss ... | wc -l`. The pipe
takes the left command's output and feeds it as the right command's input,
instead of printing it to the screen directly.

---

## 8. "Basic Linux internals, algorithmic and TCP/IP-related questions — e.g., how does traceroute work?"

**Traceroute:** sends packets with an increasing TTL (time-to-live), starting
at 1. Each router that forwards the packet decrements TTL; when a router
decrements it to 0, it drops the packet and sends back an ICMP "time
exceeded" message, revealing itself. Repeating with TTL=1, 2, 3... maps the
path to the destination one hop at a time.

**TCP/IP, one line:** IP handles addressing/routing between hosts; TCP adds
reliability on top — ordering, retransmission, flow control — via the
three-way handshake (`SYN`, `SYN-ACK`, `ACK`).

---

## 9. Difference between a daemon process and a foreground process

A **daemon** runs in the background, detached from any terminal, usually
started at boot and running continuously (`sshd`, `cron`, `systemd`
services). A **foreground process** is attached to your terminal session and
blocks it until finished, or until you background it (`Ctrl+Z`, `&`).

---

## 10. How do you kill a zombie process?

You can't — it's already dead; there's nothing left to kill. A zombie is a
finished process whose exit status hasn't yet been read by its **parent**
(via `wait()`) — it's just a leftover process-table entry (`Z` in `ps`), not
consuming CPU or memory. Fix: get the parent to reap it (often just requires
the parent's own code path to run `wait()`), or, if the parent is stuck/
unresponsive, kill the parent — the zombie then gets reparented to `init`/
`systemd` (PID 1), which reaps it automatically.

**What does "reap" actually mean?** It's the term for a parent **reading a
finished child's exit status**, which is the action that lets the kernel
fully remove that process's table entry. Full lifecycle:

1. A parent creates a child (`fork()`).
2. The child runs, then exits — but the kernel keeps a small leftover record
   (PID + exit status), it doesn't erase the process immediately.
3. That leftover record is the **zombie** — the process itself has stopped
   doing anything, but its "report card" is still sitting in the process
   table, waiting to be picked up.
4. The parent is expected to call `wait()`/`waitpid()` at some point — the
   syscall that reads that exit status. The moment it does, the kernel
   considers the child fully reaped and deletes the entry.
5. If the parent never calls `wait()` (buggy, stuck, badly written), the
   zombie sits there forever — nobody's claimed its exit status.

Think of it like harvesting a crop that's already grown and just sitting in
the field — the child already did all its "growing" (running); reaping is
just the quick final act of collecting the result, done by the parent, not
the child.

**Why PID 1 "reaps it automatically":** if the parent dies before reaping its
child, the child (now an "orphan") gets reparented to `init`/`systemd`
(PID 1). PID 1 has a special job baked in: it periodically calls `wait()` on
anything reparented to it this way, cleaning up zombies that would otherwise
have no legitimate parent left to claim them.

---

## 11. What is thrashing in an operating system?

When RAM is so oversubscribed that the system spends more time swapping
pages in/out of disk than doing real work — throughput collapses even though
the CPU looks "busy." Caused by too many processes, or too little RAM, for
the combined working set currently in use.

**The full mechanism:** every process has a "working set" — the pages of
data/code it's actively using right now. The kernel doesn't guarantee all of
it sits in physical RAM — it can page some out to swap when RAM is tight, and
page it back in on demand.

- **Enough RAM:** working sets fit comfortably, page-ins/outs are rare,
  everything runs at normal speed.
- **Not enough RAM — the spiral:** too many processes (or too-large working
  sets) compete for scarce RAM → kernel evicts pages to swap, guessing which
  are least likely needed soon → a process then touches a page that just got
  evicted → **page fault** → kernel fetches it back from disk (orders of
  magnitude slower than RAM) → to make room, it evicts a *different* page,
  possibly needed by someone else a moment later → repeat, constantly, across
  every competing process. The system spends nearly all its time shuffling
  pages via disk I/O, almost none running actual application code.

**Why "the CPU looks busy" is misleading:** `top` might show high CPU or a
high load average, suggesting the CPU is the bottleneck — but much of that
"busy" time is the kernel handling page faults and I/O wait, not useful work.
The system is active, but barely any of that activity is your application
making progress — the dashboard can lie to you if you don't also check
`free -h` and swap activity.

**Spotting it in practice:**
```
free -h        # swap usage climbing/actively churning — not just "some swap used," which is normal
vmstat 1 5     # watch si/so columns (swap-in/swap-out, pages per sec) — consistently high = thrashing
```

**The fix is never "add more swap"** — that just gives the system a bigger
area to thrash within, without addressing "not enough RAM for the actual
working set." Real fixes: add RAM, reduce concurrent processes/containers on
the box, or tune the app to use less memory (smaller caches, smaller JVM heap).

**One-sentence version:** *"Thrashing is when there's so little free RAM
relative to what's actively being used that the system is constantly
swapping pages in and out of disk just to keep processes running — most of
the system's time goes to page faults and disk I/O instead of real
computation, and more swap doesn't fix it, only more RAM or less memory
pressure does."*

---

## 12. Discussion on operating systems and computer network concepts

(General/open-ended prompt — covered via the specific sub-topics throughout
this document: processes, memory, disk, and the networking/TCP-IP section above.)

---

## 13. Implement a priority scheduler for an OS

Core idea: processes have a priority (like Linux's `nice` value, -20 to 19,
lower = higher priority); the scheduler always picks the highest-priority
ready process to run next. A minimal version is a priority queue:

```python
import heapq

class Scheduler:
    def __init__(self):
        self.queue = []
        self.counter = 0  # tie-breaker so equal priorities stay FIFO

    def add_process(self, name, priority):
        heapq.heappush(self.queue, (priority, self.counter, name))
        self.counter += 1

    def run_next(self):
        if not self.queue:
            return None
        priority, _, name = heapq.heappop(self.queue)
        return name
```
Lower `priority` number runs first (mirrors `nice`). If asked about
preemption or time-slicing, mention round-robin as the next step up —
describe it, don't over-engineer live.

**What is `priority` in this code, exactly?** It's a plain integer **you pass
in yourself** when calling `add_process` — the class doesn't compute it, the
caller decides how important each process is, the same way you'd set a
`nice` value on a real process. Lower number = higher importance = runs
first, since `heapq` is a min-heap (always pops the smallest value first).

Traced with actual calls:
```python
s = Scheduler()
s.add_process("backup_job", 10)     # low importance
s.add_process("web_request", 1)     # high importance
s.add_process("log_flush", 5)       # medium importance

print(s.run_next())   # "web_request"  (priority 1 — smallest — runs first)
print(s.run_next())   # "log_flush"    (priority 5 — next smallest)
print(s.run_next())   # "backup_job"   (priority 10 — last)
```

**Why `self.counter` rides along in the tuple:** `heapq` compares tuples
element by element — `(priority, counter, name)` compares by `priority`
first, and only falls back to `counter` if two processes tie on priority.
Since `counter` increments by 1 on every add, it's really just recording
insertion order — so two processes at the same priority still come out
first-in-first-out, instead of `heapq` trying to compare the `name` strings
directly (which could error, or give an arbitrary alphabetical tie-break).

---

## 14. Linux internals: inode, OOM killer, disk space cleanup, `fstab`, swap space

**Inode:** metadata about a file — permissions, owner, size, timestamps,
pointers to data blocks — **not the filename** (filenames live in directory
entries pointing to an inode). You can run out of inodes (too many small
files) while `df` still shows free space — a separate limit from raw bytes.

**Analogy:** the inode is a library card (owner, size, timestamps, and which
shelf/blocks the actual data sits at). The filename is just a label in a
directory pointing *at* a card — it isn't part of the card itself.

**Why hard links prove the filename/inode split — a real walkthrough:**
```
$ echo "hello world" > report.txt
$ ls -i report.txt
524288 report.txt              # inode number for the data

$ ln report.txt backup.txt     # creates a 2nd NAME pointing at the SAME inode — no data copied
$ ls -i report.txt backup.txt
524288 report.txt
524288 backup.txt              # same inode number — same underlying data, two labels

$ rm report.txt
$ cat backup.txt
hello world                    # data survives — report.txt was just one of two names pointing at it
```
Every inode keeps a **link count** — how many filenames currently point at
it. Creating the hard link bumped it from 1→2; removing `report.txt` dropped
it back to 2→1. Only when the *last* filename pointing at an inode is
removed (link count hits 0) does the kernel actually reclaim the disk space
— `rm` only ever removes a name, never data directly. This is also why tools
like `rsync --link-dest` can create what look like multiple full backups
while actually storing unchanged files only once — hard links are instant
and free, no data copied.

**OOM killer:** when the system genuinely runs out of memory and can't
reclaim more, the kernel picks a process to kill (scored via `oom_score`) to
prevent total system lockup. Check via `dmesg | grep -i "out of memory"`.

**Analogy:** an office building where every floor is drawing power and the
supply is maxed out — if nothing changes, the whole building's grid trips at
once. Facilities cuts power to one floor to save the rest. The OOM killer is
that forced sacrifice: once RAM+swap are truly exhausted and nothing more
can be freed by normal means (cache already squeezed dry), the kernel kills
whichever process scores worst on `oom_score` — usually the single biggest
memory consumer, since killing it frees the most room fastest. This is why a
service that "just vanished" with no error in its own logs is a signature of
an OOM kill — the process was terminated from outside, instantly, with no
chance to log its own death; the only record lives in the kernel's log
(`dmesg`), not the app's.

**Disk space cleanup — a real walkthrough, drilling one level deeper each round:**
```
$ du -sh /* 2>/dev/null | sort -rh | head
15G   /var
12G   /home
8.2G  /usr

$ du -sh /var/* 2>/dev/null | sort -rh | head    # step INSIDE the biggest offender, ask again
12G   /var/log
2.1G  /var/lib

$ du -sh /var/log/* 2>/dev/null | sort -rh | head   # one level deeper again
11G   /var/log/myapp

$ ls -lhS /var/log/myapp | head -5    # small enough now to list individual files, sorted by size
-rw-r--r-- 1 root root 10G Aug 22 09:00 debug.log
```
Same command re-run each round, just one directory further in — stop the
moment the numbers stop being surprising (spread evenly across several
folders = no single culprit; one file explaining almost everything = found it).
Common culprits: log files, old kernels/package caches, Docker images/layers, `/tmp`.

**Why `2>/dev/null` is tacked onto the `du` command:** every command has two
separate output streams — **stdout** (`1`, normal output) and **stderr**
(`2`, error messages) — normally both print to your screen mixed together.
`du -sh /*` tries to walk into every folder under `/`, including ones you
lack permission for, and prints a `Permission denied` line for each one,
interleaved with your actual size results. `2>/dev/null` redirects just the
error stream into `/dev/null` — a special file that discards anything
written to it — leaving only the clean, successful measurements on screen.
The errors still happen (those folders just aren't counted), but you're not
forced to read the noise. Using this without being asked is a small but real
signal that you've actually run this command for real, not just read about it.

**`fstab` (`/etc/fstab`):** config file defining how partitions/devices are
mounted at boot — device, mount point, filesystem type, options — so mounts
happen automatically without manual `mount` commands.

**Why it matters practically:** `mount` run by hand is temporary — it only
lasts until the next reboot, at which point the kernel forgets the
attachment entirely (the data isn't gone, just no longer attached to a
folder). `fstab` is what the boot process reads automatically every startup,
re-running the equivalent `mount` for every line in the file with no human
involved. Attach a new disk to an EC2 instance manually without adding it to
`fstab`, and after a reboot the data looks "missing" — it's not, it's just unattached.

**Swap space:** disk-backed overflow for RAM. Inactive pages get written to
swap when RAM fills, so RAM can be reused. Slow (it's disk), so heavy
swapping is itself a symptom of a memory problem, not a fix.

**Analogy:** RAM is a desk (fast, small); swap is a filing cabinet down the
hall (much bigger, slow to reach). Moving pages to swap isn't data loss —
just relocation somewhere slower. Some light, occasional swap use is normal
and healthy (a safety valve for rare spikes). *Heavy, continuous* swapping
means the workload genuinely needs more RAM than exists — adding more swap
just delays hitting that wall longer, it doesn't remove it.

---

## 15. What is swappiness?

A kernel tunable (0-100, `/proc/sys/vm/swappiness`) controlling how eagerly
the kernel swaps memory pages to disk vs. reclaiming page cache instead. Low
value = keep things in RAM as long as possible (good for latency-sensitive
workloads); high value = swap more readily.

**The desk analogy (the version that actually makes it click):**

RAM is a small desk. Two kinds of things pile up on it:

- **Pile 1 — process memory (anonymous memory):** a program's actual
  in-progress work (e.g. a Java heap). If evicted, the only place it can go
  is the filing cabinet down the hall (swap, on disk) — getting it back is a
  slow walk there and back.
- **Pile 2 — page cache:** photocopies of file data the kernel cached in RAM
  for speed. The real original still exists on disk (the "library"), so
  throwing a photocopy away loses nothing — worst case, it's slowly
  re-copied from disk next time it's needed.

**When the desk is full and something new needs space, the kernel picks one:**
- **Option A:** send some Pile 1 (real work) to the filing cabinet — disruptive, slow to retrieve.
- **Option B:** just throw away some Pile 2 (photocopies) — nothing real lost.

**Swappiness is the dial for "which pile do you clear from first?"**
- **Low swappiness** = *"clear Pile 2 first — protect my real work, sacrifice cache."*
- **High swappiness** = *"I don't mind sending some real work to the filing cabinet — keep more cache around instead."*

**Worked example:** desk is full, kernel needs to free 100MB. Currently: 300MB
of Pile 1 (a Java heap) + 200MB of Pile 2 (cached log file reads).
- **Low swappiness:** kernel frees the 100MB entirely from Pile 2 — never
  touches the Java process's real memory.
- **High swappiness:** kernel might take, say, 60MB from Pile 1 (swapped to
  disk) and only 40MB from Pile 2 — meaning part of that Java process's
  actual heap is now on slow disk, and touching it again soon means a stall.

**Why it matters for something like a database:** its in-progress working
data (query buffers, connection state) is Pile 1 — you don't want that
sent to the slow filing cabinet mid-query. So databases are typically tuned
to low swappiness (`1`–`10`): protect real work, sacrifice cache shortcuts first.

**Default is usually 60** — general-purpose middle ground; commonly tuned
lower explicitly on database/latency-sensitive servers.

**Checking/changing it:**
```
cat /proc/sys/vm/swappiness      # check current value
sysctl vm.swappiness=10          # change temporarily (resets on reboot)
```
Persist across reboots by adding `vm.swappiness=10` to `/etc/sysctl.conf`.

**Nuance:** `swappiness=0` doesn't mean "never swap" — the kernel can still
swap as a last resort if there's no cache left to evict and RAM is truly
exhausted. It means "avoid swapping as strongly as possible, prefer cache
eviction first," not an absolute guarantee.

**Ties back to thrashing:** swappiness doesn't prevent thrashing — it only
changes *which* memory (cache vs. process data) gets sacrificed first when
memory pressure hits. If there's genuinely not enough RAM for the workload,
thrashing happens regardless of this setting.

**One-sentence version:** *"Swappiness controls whether the kernel prefers
to evict disposable file cache or push a process's actual memory out to swap
when RAM gets tight — low values protect process memory at the cost of
cache, high values do the reverse — but it doesn't prevent memory pressure
itself, it just decides which kind of memory gets sacrificed first."*

---

## 16. What are system load averages?

Three numbers (1-min, 5-min, 15-min) from `uptime`/`top`/`/proc/loadavg` —
roughly, the average number of processes that were running or waiting for
CPU/I/O. Compare against core count: load of 4 on 4 cores ≈ fully utilized;
load of 8 on 4 cores ≈ overloaded, things queuing.

---

## 17. CPU affinity?

Pinning a process to run only on specific CPU core(s) (`taskset`).
Improves cache locality and avoids the cost of a process bouncing between
cores — useful for latency-sensitive or high-throughput workloads.

---

## 18. Memory management? Process management?

Covered under items 4, 11, 13, 14, 15 above (memory management = virtual
address space translation, thrashing, swap, OOM killer, swappiness; process
management = process states, daemons vs. foreground, zombies, scheduling/priority).

---

## 19. "What is the most complex problem you've solved?" (Linux/systems context)

Not a knowledge question — it's behavioral, dressed in systems language. Prep
a real story from Polaris/OME.Next orchestrator work: pick a genuinely gnarly
cross-service debugging incident (Kafka/Valkey/Postgres interaction, a k3s
scheduling issue, a non-obvious root cause), structure it STAR-style
(situation → what didn't work → what isolated the cause → the fix).
Interviewers weight *how you debugged* over *what the bug was*.

---

## Appendix — every command in this document, alphabetical, with why it's used

| Command | Why you'd use it |
|---|---|
| `apt install --reinstall coreutils` | Restore a broken core binary (e.g. a `chmod` that lost its execute bit) by overwriting it with a fresh copy from the package |
| `cat /proc/<pid>/stack` | See the kernel-level call stack of a process stuck in `D` state, to tell disk vs. network vs. lock |
| `cat /proc/<pid>/status` | Per-process memory/state detail — `VmRSS` is the real physical RAM in use |
| `cat /proc/cpuinfo` | Raw CPU info — what `nproc`/`lscpu` read from |
| `cat /proc/loadavg` | Raw load-average numbers, straight from the kernel |
| `cat /proc/meminfo` | Raw memory info — what `free` reads from |
| `cat /proc/sys/vm/swappiness` | Check the current swappiness value |
| `chmod 755 file` | Set permissions: owner=rwx, group=r-x, other=r-x |
| `df -h` | Disk space used/free, per filesystem, human-readable |
| `df -i` | Inode usage per filesystem — a *separate* exhaustion mode from raw bytes |
| `dmesg` | Print the kernel ring buffer — hardware/driver/OOM-killer events |
| `dmesg \| grep -i "out of memory"` | Check whether the OOM killer already killed something |
| `du -sh /* 2>/dev/null \| sort -rh \| head -10` | Find which top-level directories are eating disk space |
| `find /path -size +200M` | Find files over a given size |
| `free -h` | RAM + swap usage at a glance — `available` matters more than raw `used` |
| `grep -i "pattern" file` | Case-insensitive text search |
| `iostat -x 1 5` | Disk I/O stats, sampled every 1s for 5 samples — `%util`/`await` reveal disk-bound slowness |
| `journalctl -k \| grep -i oom` | Same OOM check as `dmesg`, via systemd's journal |
| `journalctl -u <service>` | Read a specific systemd service's logs |
| `jstack <pid> > dump.txt` | Full thread dump of a JVM process — bundled with the JDK |
| `kill <pid>` | Graceful shutdown request (`SIGTERM`) — lets the process clean up |
| `kill -3 <pid>` | Ask a JVM to dump its own thread stacks to its log, if `jstack` isn't available |
| `kill -9 <pid>` | Forceful, immediate kill — no cleanup, last resort |
| `ls -lhS /var/log \| head -5` | Find the largest individual files in a directory, biggest first |
| `ls /lib64/ld-linux*` | Find the exact path of the dynamic linker on this system |
| `lscpu` | Detailed CPU info: sockets, cores, threads, cache |
| `lsof -p <pid>` | List every file/socket a process currently has open |
| `nproc` | Number of CPU cores — needed to correctly interpret any `%CPU` reading above 100 |
| `perf top -p <pid>` | Function-level CPU profiling of a running process |
| `ping -c 4 <host>` | Basic reachability/latency check to a host |
| `ps aux \| grep <pid>` | Check one process's `STAT` column — is it `D` (I/O-blocked) or just `S` (idle)? |
| `ps aux --sort=-%cpu \| head -10` | One-time snapshot of the top 10 CPU-hungriest processes |
| `ps aux --sort=-%mem \| head -10` | Same, sorted by memory instead |
| `py-spy dump --pid <pid>` | Thread dump for a Python process (needs `pip install py-spy` first) |
| `readelf -l /bin/ls \| grep interpreter` | Find which dynamic linker a given binary expects |
| `sed 's/old/new/' file` | Find-and-replace text in a stream |
| `ss -tulpn` | Listening ports + established connections + the process that owns each |
| `ss -tn state established '( dport = :PORT )' \| wc -l` | Count active connections to one specific port (e.g. a database) |
| `strace -p <pid>` | Live trace of every syscall a running process makes — the core tool for "what is it actually doing" |
| `sysctl vm.swappiness=10` | Change swappiness temporarily (add to `/etc/sysctl.conf` to persist) |
| `systemctl status <service>` | Check whether a daemon/service is running and healthy |
| `tail -f file.log` | Live-follow a growing log file |
| `taskset` | Pin a process to specific CPU core(s) — sets CPU affinity |
| `top` | Live, auto-refreshing dashboard: CPU, memory, load average, process list |
| `top -H -p <pid>` | Same, but broken down by individual thread within one process |
| `traceroute <host>` | Map the hop-by-hop network path to a host, via increasing TTL |
| `truncate -s 0 file.log` | Empty a log file instantly, without restarting the process writing to it |
| `uptime` | Quick load-average + how-long-the-box-has-been-up check |
| `vmstat 1 5` | Watch `si`/`so` (swap in/out) columns over time — confirms active thrashing |
| `/lib64/ld-linux-x86-64.so.2 /bin/chmod 755 /bin/chmod` | The dynamic-linker trick — restore `chmod`'s own execute bit by having `ld.so` read and run it as data, bypassing the need for `chmod` to execute itself |
| `python3 -c "import os; os.chmod('/bin/chmod', 0o755)"` | Same goal, via Python calling the underlying `chmod()` syscall directly |
