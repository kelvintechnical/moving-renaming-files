# Lab 10: Moving and Renaming Files — `mv`, `mv -i`, `mv -n`, atomic renames

- **Series:** linux-ops-mastery — File Operations & Shell Fundamentals
- **Career arcs covered:** RHCSA EX200 (log rotation, fstab safety, config staging), RHCE EX294 (`ansible.builtin.command` with `creates:` for moves; `ansible.builtin.copy` for atomic config replace), CKA (ConfigMap atomic swaps), RHCA — RH358 (atomic file update patterns)
- **Prerequisite:** Lab 00 (Ansible control node) + Lab 09 (`ln`, `ln -s`, `readlink`)
- **Time Estimate:** 35–50 minutes
- **Tasks:** 5 (ADHD 3-1-1 spec — 3 RHCSA + 1 partial-Ansible-Boundary + 1 Verification capstone)
- **Practice Directory (lab-wide rotation #10):** `/var` (real `mv` happens during log rotation, package upgrades, etc.)
- **Sandbox:** `/srv/mv-lab`
- **Traps rehearsed this lab:** **T10-A** (`mv` ACROSS filesystems is NOT atomic — it's `cp` + `rm`; partial state on power loss) · **T10-B** (`mv` silently overwrites without `-i`/`-n` — gone is gone) · **T10-C** (Ansible has no `mv` module — `command: mv` needs `creates:` for idempotence)

> **This lab's practice directory is: `/var`** — we read log rotation patterns there. The sandbox is `/srv/mv-lab` where we actually move and rename.

---

## 🖥️ LAB HEADER BLOCK — run this FIRST

```bash
echo "🖥️  ENV:   ${ENV:-DECLARE_ME}"
echo "💿  DISK:  $(lsblk 2>/dev/null | awk '$NF=="disk"{print "/dev/"$1}' | paste -sd, -)"
echo "🔐  SE:    $(getenforce 2>/dev/null || echo n/a)"
echo "📦  OS:    $(cat /etc/redhat-release 2>/dev/null || grep PRETTY_NAME /etc/os-release)"
echo "🕒  TIME:  $(date -Is)"
echo "👤  USER:  $(whoami)@$(hostname)"
echo "⚠️  TRAP REMINDERS THIS LAB: T10-A T10-B T10-C"
echo "📁  PRACTICE DIR: /var"
echo ""
echo "💡 Log rotation example (read-only):"
ls -lt /var/log/messages* 2>/dev/null | head -3 || ls -lt /var/log/ 2>/dev/null | head -3
```

> **STOP — paste header output before running setup.**

---

## 🎯 Objective

Rename and move files safely. By the end you will:

- Rename a file in place with `mv old new`
- Move files between directories on the **same filesystem** (atomic — the inode just gets a new parent)
- Move files between **different filesystems** (NOT atomic — `cp` + `rm` under the hood)
- Use `-i` (interactive), `-n` (no-clobber), `-u` (update if newer), `-b` (backup) safely
- Understand why `ansible.builtin.copy` is the RHCE-canonical "atomic config replace" — NOT `command: mv`
- Replicate `mv` semantics with `ansible.builtin.command: mv` + `creates:` (the Ansible-Boundary pattern)

---

## 🛠️ Setup — run once before Task 1

```bash
sudo mkdir -p /srv/mv-lab/{src,archive}
sudo mkdir -p /root/rhcsa_journal/lab10
echo "config v1"  | sudo tee /srv/mv-lab/src/config.txt
echo "logfile A"  | sudo tee /srv/mv-lab/src/app.log
echo "logfile B"  | sudo tee /srv/mv-lab/src/access.log
ls -liR /srv/mv-lab/
```

---

## Task 1 — Rename in Place + Move Between Directories (Same Filesystem)

**Practice directory this task:** `/var` (logs), `/srv/mv-lab` (write)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab10/task1
date -Is | sudo tee /root/rhcsa_journal/lab10/task1/start.txt
ls -li /srv/mv-lab/src/ | sudo tee -a /root/rhcsa_journal/lab10/task1/start.txt
echo "exit was: $?"
```

### Purpose

Rename `config.txt` to `config.txt.old`, move `app.log` into `/srv/mv-lab/archive/`, observe that the inode does NOT change (same filesystem → atomic rename), and capture the before/after listing.

### Main Command Block

```bash
# Before
ls -li /srv/mv-lab/src/
inode_before=$(stat -c '%i' /srv/mv-lab/src/config.txt)

# Rename in place
sudo mv /srv/mv-lab/src/config.txt /srv/mv-lab/src/config.txt.old
ls -li /srv/mv-lab/src/

# Inode should be unchanged (rename within same fs)
inode_after=$(stat -c '%i' /srv/mv-lab/src/config.txt.old)
echo "before=$inode_before after=$inode_after"
[ "$inode_before" = "$inode_after" ] && echo "INODE_PRESERVED"

# Move into archive dir
sudo mv /srv/mv-lab/src/app.log /srv/mv-lab/archive/
ls -liR /srv/mv-lab/

# Move + rename in one go
sudo mv /srv/mv-lab/src/access.log /srv/mv-lab/archive/access-20260527.log
ls -li /srv/mv-lab/archive/

# Inspect: mtime did NOT change (mv preserves all metadata — it's just a rename)
stat -c 'mtime=%y' /srv/mv-lab/archive/access-20260527.log
stat -c 'ctime=%z' /srv/mv-lab/archive/access-20260527.log
# Note: ctime DID change (inode metadata field 'parent directory' changed)

# Capture
{
  echo "=== rename inode preserved ==="
  echo "before=$inode_before after=$inode_after"
  [ "$inode_before" = "$inode_after" ] && echo "INODE_PRESERVED" || echo "INODE_CHANGED"
  echo "=== final layout ==="; ls -liR /srv/mv-lab/
} 2>&1 | sudo tee /root/rhcsa_journal/lab10/task1/transcript.txt
```

### Human-Readable Breakdown

Within a **single filesystem**, `mv` is a directory-entry rename. The inode (the actual file data) does not move — only its name and/or parent-directory entry change. That's why:

- mtime is preserved (no data write)
- ctime is updated (inode metadata changed — the "parent directory" field)
- The operation is atomic (the rename either happens completely or not at all)

`mv src dst`:
- If `dst` is an **existing directory**, `mv` moves `src` INTO it as `dst/$(basename src)`
- If `dst` is a **file path that exists**, `mv` overwrites it (T10-B — silent overwrite without `-i`/`-n`)
- If `dst` does not exist, `mv` renames `src` to `dst`

### Reading It Left to Right

`mv SRC DST`

- `mv` — move/rename
- `SRC` — source path
- `DST` — destination (rules above)

`stat -c '%i' FILE`

- `stat` — file metadata
- `-c '%i'` — inode number

### The Story

A grader: "rotate `/var/log/myapp.log` to `/var/log/myapp.log.1` and start a fresh log file." Step one: `mv /var/log/myapp.log /var/log/myapp.log.1`. Step two: `touch /var/log/myapp.log`. The mv preserves the old log's mtime exactly because `mv` doesn't touch data. That's RHCSA log-rotation 101.

### Expected Output

```
$ ls -li /srv/mv-lab/src/
total 12
12340 -rw-r--r--. 1 root root 11 May 27 15:01 access.log
12341 -rw-r--r--. 1 root root 11 May 27 15:01 app.log
12342 -rw-r--r--. 1 root root 11 May 27 15:01 config.txt

$ sudo mv /srv/mv-lab/src/config.txt /srv/mv-lab/src/config.txt.old
$ ls -li /srv/mv-lab/src/
12340 -rw-r--r--. 1 root root 11 May 27 15:01 access.log
12341 -rw-r--r--. 1 root root 11 May 27 15:01 app.log
12342 -rw-r--r--. 1 root root 11 May 27 15:01 config.txt.old
^^^^^                                                          ^^^^^^^^^^^
same inode                                                     renamed only

$ echo "before=12342 after=12342"
INODE_PRESERVED
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `mv SRC DST` | Move/rename | Base case |
| `-i` | Interactive (prompt before overwrite) | T10-B antidote |
| `-n` | No-clobber (don't overwrite) | Idempotent without prompting |
| `-u` | Update only (move if SRC newer than DST) | Sync-like move |
| `-b` | Backup overwritten dest | Safety net |
| `-v` | Verbose | Useful in scripts |
| `-t DIR` | Target-directory-first form: `mv -t DIR SRC1 SRC2...` | Useful with `xargs` |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| Same-fs `mv` | Directory-entry rename; inode preserved; atomic |
| mtime preservation | `mv` does NOT update mtime (no data write) |
| ctime update | `mv` DOES update ctime (inode metadata changed) |
| `dst` is dir | `mv` moves src INTO dst |
| `dst` is file | `mv` overwrites dst silently (use `-i`/`-n`) |
| `dst` does not exist | `mv` renames |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **T10-B** | `mv src dst` silently overwrites an existing dst | Use `-i` interactive or `-n` no-clobber |
| Filename collision | Moving multiple sources into a dir, two have same basename | One overwrites the other; use unique basenames or `-b` backup |
| Overlooking `-t` | `find ... -exec mv {} /dst \;` is verbose | Use `mv -t /dst SRC1 SRC2...` for many sources |

### 🔁 Persistence Check

```bash
test -f /srv/mv-lab/src/config.txt.old && echo "rename ok"
test -f /srv/mv-lab/archive/app.log && echo "moved ok"
test -f /srv/mv-lab/archive/access-20260527.log && echo "move+rename ok"
[ "$inode_before" = "$(stat -c '%i' /srv/mv-lab/src/config.txt.old)" ] && echo "inode preserved"
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab10/task1/done.txt > /dev/null <<EOF
lab=10 task=1
when=$(date -Is)
practice_dir=/var
rename_inode_match=$([ "$inode_before" = "$(stat -c '%i' /srv/mv-lab/src/config.txt.old)" ] && echo yes || echo no)
moves_completed=3
EOF
cat /root/rhcsa_journal/lab10/task1/done.txt
```

### 🧹 Cleanup

Leave the lab; Task 2 demonstrates overwrites.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `mv: cannot stat 'X': No such file or directory` | Source path wrong; check `ls -l` |
| Inode changed after rename | You crossed a filesystem boundary — that's Task 2 |

> **STOP — confirm `rename_inode_match=yes` in done.txt before Task 2.**

---

## Task 2 — Overwrite Safety: `mv -i`, `mv -n`, `mv -b`

**Practice directory this task:** `/srv/mv-lab` (write)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab10/task2
date -Is | sudo tee /root/rhcsa_journal/lab10/task2/start.txt
echo "exit was: $?"
```

### Purpose

Build two files with conflicting destinations, attempt a `mv` that would overwrite, then show how `-i`, `-n`, and `-b` each change the outcome.

### Main Command Block

```bash
echo "VERSION 1" | sudo tee /srv/mv-lab/src/v1.txt
echo "VERSION 2" | sudo tee /srv/mv-lab/src/v2.txt
echo "EXISTING TARGET" | sudo tee /srv/mv-lab/archive/target.txt
ls -l /srv/mv-lab/archive/target.txt
cat /srv/mv-lab/archive/target.txt

# 1) Default mv — silently overwrites (T10-B)
sudo mv /srv/mv-lab/src/v1.txt /srv/mv-lab/archive/target.txt
cat /srv/mv-lab/archive/target.txt          # now: "VERSION 1" — original "EXISTING TARGET" gone

# Restore for next test
echo "EXISTING TARGET" | sudo tee /srv/mv-lab/archive/target.txt

# 2) mv -n — skip if target exists (silent)
sudo mv -n /srv/mv-lab/src/v2.txt /srv/mv-lab/archive/target.txt
cat /srv/mv-lab/archive/target.txt          # still "EXISTING TARGET" — mv was no-op
ls /srv/mv-lab/src/v2.txt                   # v2.txt still in src/ (it wasn't moved)

# 3) mv -b — backup the existing target before overwrite
sudo mv -b /srv/mv-lab/src/v2.txt /srv/mv-lab/archive/target.txt
ls /srv/mv-lab/archive/
cat /srv/mv-lab/archive/target.txt          # now "VERSION 2"
cat /srv/mv-lab/archive/target.txt~         # backup file with ~ suffix: "EXISTING TARGET"

# 4) mv -i — interactive (prompt before overwrite); use yes/no
echo "EXISTING TARGET" | sudo tee /srv/mv-lab/archive/target.txt
echo "VERSION 3" | sudo tee /srv/mv-lab/src/v3.txt
echo "n" | sudo mv -i /srv/mv-lab/src/v3.txt /srv/mv-lab/archive/target.txt
cat /srv/mv-lab/archive/target.txt          # still "EXISTING TARGET" — user said n

# Capture
{
  echo "=== default mv overwrites silently (T10-B) ==="
  echo "OLD"   | sudo tee /srv/mv-lab/archive/demo.txt
  echo "NEW1"  | sudo tee /srv/mv-lab/src/d1.txt
  sudo mv /srv/mv-lab/src/d1.txt /srv/mv-lab/archive/demo.txt
  echo "after default mv: $(cat /srv/mv-lab/archive/demo.txt)"

  echo "=== mv -n preserves target ==="
  echo "PRESERVE" | sudo tee /srv/mv-lab/archive/demo.txt
  echo "NEW2"     | sudo tee /srv/mv-lab/src/d2.txt
  sudo mv -n /srv/mv-lab/src/d2.txt /srv/mv-lab/archive/demo.txt
  echo "after mv -n: $(cat /srv/mv-lab/archive/demo.txt)"
  ls /srv/mv-lab/src/d2.txt && echo "src preserved"

  echo "=== mv -b creates backup ==="
  echo "ORIG"  | sudo tee /srv/mv-lab/archive/demo.txt
  echo "REPL"  | sudo tee /srv/mv-lab/src/d3.txt
  sudo mv -b /srv/mv-lab/src/d3.txt /srv/mv-lab/archive/demo.txt
  cat /srv/mv-lab/archive/demo.txt
  cat /srv/mv-lab/archive/demo.txt~
} 2>&1 | sudo tee /root/rhcsa_journal/lab10/task2/transcript.txt
```

### Human-Readable Breakdown

`mv` is **destructive by default**. The four guard switches:

| Switch | Behavior |
|---|---|
| (default) | Overwrite silently |
| `-i` | Prompt: `mv: overwrite 'dst'?` — answer `y` or `n` |
| `-n` | No-clobber: skip silently if dst exists |
| `-b` | Backup existing dst as `dst~` before overwriting |

`-i` and `-n` are mutually exclusive — the LAST one on the command line wins. So `mv -in` behaves as `-n` (no-clobber); `mv -ni` behaves as `-i` (prompt).

`-b` pairs well with `-S SUFFIX` to choose the backup suffix (default `~`).

### Reading It Left to Right

`mv -b SRC DST`

- `mv` — move
- `-b` — backup dst if it exists
- backup gets suffix `~` (or whatever `--backup-suffix` says)

`mv -n SRC DST`

- `-n` — no-clobber; if DST exists, do nothing (SRC stays in place)

### The Story

A grader: "deploy `new-config.yml` to `/etc/myservice/config.yml` but keep the old one as a backup." Answer: `mv -b /tmp/new-config.yml /etc/myservice/config.yml`. The old config is saved as `config.yml~`. RHCSA expects you to reach for `-b` or `-i` when overwriting a config.

### Expected Output

```
=== default mv overwrites silently (T10-B) ===
after default mv: NEW1

=== mv -n preserves target ===
after mv -n: PRESERVE
/srv/mv-lab/src/d2.txt
src preserved

=== mv -b creates backup ===
REPL
ORIG
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `-i` | Interactive prompt | Safe ad-hoc |
| `-n` | No-clobber | Idempotent without prompts |
| `-b` | Backup overwritten dst | Safety net for configs |
| `-S SUFFIX` | Backup suffix override | Match company convention |
| `-u` | Move only if SRC newer | rsync-like behaviour |
| `-v` | Verbose | Script debugging |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| Default `mv` | Destructive — silent overwrite |
| `-i` vs `-n` | Prompt vs silent skip |
| `-b` | Safety net — `dst~` backup |
| Last switch wins | `mv -in` ≡ `-n`; `mv -ni` ≡ `-i` |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **T10-B** | Default `mv` overwrites silently — production config gone | Use `-i`, `-n`, or `-b` on config moves |
| Alias surprise | Distros sometimes alias `mv='mv -i'` — script behaves differently in shell vs cron | Use `/usr/bin/mv` or `\mv` in scripts |

### 🔁 Persistence Check

```bash
test -f /srv/mv-lab/archive/demo.txt   && echo "demo.txt ok"
test -f /srv/mv-lab/archive/demo.txt~  && echo "backup exists"
grep -c 'PRESERVE' /root/rhcsa_journal/lab10/task2/transcript.txt
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab10/task2/done.txt > /dev/null <<EOF
lab=10 task=2
when=$(date -Is)
backup_present=$(test -f /srv/mv-lab/archive/demo.txt~ && echo yes || echo no)
no_clobber_worked=$(grep -c 'PRESERVE' /root/rhcsa_journal/lab10/task2/transcript.txt)
EOF
cat /root/rhcsa_journal/lab10/task2/done.txt
```

### 🧹 Cleanup

Leave the archive; Task 3 demonstrates cross-fs behaviour.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `-i` doesn't prompt | Alias `mv=mv -i` interaction with script input — use `/usr/bin/mv -i` |
| No backup created with `-b` | dst didn't exist before — `-b` only acts on overwrite |

> **STOP — confirm `backup_present=yes` in done.txt before Task 3.**

---

## Task 3 — Cross-Filesystem `mv` Is NOT Atomic (cp + rm Under the Hood)

**Practice directory this task:** `/srv/mv-lab` and `/tmp` (often different filesystems)

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab10/task3
date -Is | sudo tee /root/rhcsa_journal/lab10/task3/start.txt
df -hT /srv /tmp | sudo tee -a /root/rhcsa_journal/lab10/task3/start.txt
echo "exit was: $?"
```

If `/srv` and `/tmp` are on the **same filesystem** (typical on a default RHEL VM where everything is on `/`), the cross-fs demonstration won't manifest. Use `df -T` to detect; if same filesystem, the lab still works — Task 3 just shows the inode-preserved path, not the cross-fs path.

### Purpose

Move a file across filesystems (or simulate it) and observe:

- Inode number CHANGES (different filesystem = different inode table)
- mtime is preserved by `mv` even across filesystems (`mv` calls `cp -p` internally)
- Operation is NOT atomic — interrupt-able

### Main Command Block

```bash
df -hT /srv /tmp

# Source file on /srv
echo "cross-fs test" | sudo tee /srv/mv-lab/src/crossfs.txt
inode_src=$(stat -c '%i' /srv/mv-lab/src/crossfs.txt)
fs_src=$(df --output=source /srv/mv-lab/src/crossfs.txt | tail -1)
echo "src: fs=$fs_src inode=$inode_src"

# Move across (or within, depending on layout)
sudo mv /srv/mv-lab/src/crossfs.txt /tmp/crossfs.txt
inode_dst=$(stat -c '%i' /tmp/crossfs.txt)
fs_dst=$(df --output=source /tmp/crossfs.txt | tail -1)
echo "dst: fs=$fs_dst inode=$inode_dst"

if [ "$fs_src" != "$fs_dst" ]; then
  echo "ACROSS FILESYSTEMS"
  [ "$inode_src" != "$inode_dst" ] && echo "INODE_CHANGED_AS_EXPECTED"
else
  echo "SAME FILESYSTEM (atomic rename — inode preserved)"
  [ "$inode_src" = "$inode_dst" ] && echo "INODE_PRESERVED"
fi

# mtime preserved either way (mv copies metadata)
stat -c 'mtime=%y' /tmp/crossfs.txt

# Demonstrate the consequence of non-atomic cross-fs mv via simulated interrupt
# (we don't actually kill mv — we show that for large files, mv has to copy bytes)
sudo dd if=/dev/zero of=/srv/mv-lab/src/big.bin bs=1M count=20 status=none
ls -lh /srv/mv-lab/src/big.bin

# Time the move — if cross-fs, it takes ~1s (copying 20MB); if same-fs, near-instant
time sudo mv /srv/mv-lab/src/big.bin /tmp/big.bin
ls -lh /tmp/big.bin

# Capture
{
  echo "=== source FS ==="; df --output=source,fstype /srv | tail -1
  echo "=== dest FS ===";   df --output=source,fstype /tmp | tail -1
  echo "=== inode src/dst ==="
  echo "src=$inode_src dst=$inode_dst"
  if [ "$fs_src" != "$fs_dst" ]; then echo "CROSS_FS_MOVE"; else echo "SAME_FS_RENAME"; fi
  echo "=== mtime preserved ==="; stat -c 'mtime=%y' /tmp/crossfs.txt
} 2>&1 | sudo tee /root/rhcsa_journal/lab10/task3/transcript.txt
```

### Human-Readable Breakdown

`mv` calls the kernel's `rename()` system call. If src and dst are on the **same filesystem**, `rename()` succeeds and `mv` is done in microseconds (atomic directory-entry change). If src and dst are on **different filesystems**, `rename()` returns `EXDEV` (cross-device link not permitted). `mv` then falls back to:

1. `cp -p src dst` (copy with metadata preservation)
2. `rm src`

This is NOT atomic. If power dies between steps 1 and 2, both files exist briefly. Worse — for large files, the copy takes wall-clock time during which the file can be observed in two places.

Cross-fs `mv` also CHANGES the inode (different inode table on the destination filesystem). Hard links to the source path are NOT preserved by cross-fs `mv`. This is the operational reason backup systems often use `cp -a` plus a separate retention step rather than `mv`.

### Reading It Left to Right

`df --output=source,fstype PATH`

- `df` — filesystem disk usage
- `--output=source,fstype` — only these columns
- `PATH` — show which filesystem PATH is on

`time mv SRC DST`

- `time` — measure wall + cpu time
- `mv SRC DST` — the move

### The Story

A grader: "move `/var/log/large.log` to `/backup/large.log`." If `/var/log` and `/backup` are on the same filesystem, instant. If `/backup` is a mounted NFS share or a separate disk, you're copying gigabytes — and the operation is interrupt-able. RHCSA expects you to know that `mv` across filesystems can be slow and non-atomic.

### Expected Output

If same filesystem (typical default install):

```
SAME FILESYSTEM (atomic rename — inode preserved)
INODE_PRESERVED
```

If cross filesystem (e.g., `/tmp` on tmpfs):

```
ACROSS FILESYSTEMS
INODE_CHANGED_AS_EXPECTED
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `df --output=source,fstype PATH` | Show which fs PATH is on | Cross-fs detection |
| `time CMD` | Measure CMD duration | See cross-fs mv take wall time |
| `dd if=/dev/zero of=FILE bs=1M count=N` | Generate N-MB test file | Reliably create large enough file to feel the copy |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| Same-fs `mv` | `rename()` — instant, atomic, inode preserved |
| Cross-fs `mv` | `cp -p` + `rm` — NOT atomic, inode changes |
| Hard links | Survive same-fs `mv`, BREAK on cross-fs `mv` |
| EXDEV | `rename()` errno when src/dst on different fs |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **T10-A** | Cross-fs `mv` of a huge file appears to "hang" — it's copying | Check `df -T` first; use `rsync --remove-source-files` for explicit semantics |
| Hard-link loss | `mv` across fs breaks hard-link relationships silently | Inspect with `find -inum` before AND after cross-fs moves |
| SELinux drift | Cross-fs `mv` may not preserve contexts in all configs | `restorecon` on the destination after a cross-fs move |

### 🔁 Persistence Check

```bash
test -f /tmp/crossfs.txt && echo "file moved"
test -f /tmp/big.bin && echo "big file moved"
df --output=source /tmp /srv/mv-lab/src/ 2>/dev/null
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab10/task3/done.txt > /dev/null <<EOF
lab=10 task=3
when=$(date -Is)
src_fs=$(df --output=source /srv/mv-lab 2>/dev/null | tail -1)
dst_fs=$(df --output=source /tmp 2>/dev/null | tail -1)
cross_fs=$([ "$src_fs" != "$dst_fs" ] && echo yes || echo no)
EOF
cat /root/rhcsa_journal/lab10/task3/done.txt
```

### 🧹 Cleanup

```bash
sudo rm -f /tmp/crossfs.txt /tmp/big.bin
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Same-fs detected even though `/tmp` looks separate | RHEL 9 default is `/tmp` on `/` — not tmpfs unless you enabled `tmp.mount` |
| `dd` slow | Add `oflag=direct` (skip page cache) or reduce count |

> **STOP — confirm `cross_fs` line in done.txt (yes or no — both valid) before Task 4.**

---

## Task 4 — Ansible: `command: mv` with `creates:` Idempotence (Boundary-Aware)

**Practice directory this task:** `/srv/mv-lab`

> **This is a partial Ansible Boundary task.** There is no real `ansible.builtin.mv` module. For renames of existing data we wrap `mv` in `ansible.builtin.command` with `creates:` to keep it idempotent. For atomic config replacement, the RHCE-canonical answer is `ansible.builtin.copy` (NOT `mv`) — we demonstrate both below.

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab10/task4/playbooks
date -Is | sudo tee /root/rhcsa_journal/lab10/task4/start.txt
ansible --version | head -1 | sudo tee -a /root/rhcsa_journal/lab10/task4/start.txt
echo "exit was: $?"
```

If `ansible --version` fails — **Lab 00**.

### Purpose

Demonstrate the two RHCE-defensible patterns:

1. **Move existing data** → `ansible.builtin.command: mv SRC DST` with `creates: DST` and `removes: SRC` for idempotence
2. **Atomic config replace** → `ansible.builtin.copy` (Ansible handles temp-file + rename atomically inside the module)

Then prove idempotence on both: second run = `changed=0`.

### Main Command Block

Setup the test fixture (existing files):

```bash
echo "playbook-content" | sudo tee /srv/mv-lab/src/play-source.txt
echo "old config v0"    | sudo tee /srv/mv-lab/archive/service.conf
```

Write the playbook:

```bash
sudo tee /root/rhcsa_journal/lab10/task4/playbooks/move.yml > /dev/null <<'EOF'
---
- name: Lab 10 Task 4 — move existing data + atomic config replace
  hosts: localhost
  become: true
  gather_facts: false

  vars:
    move_src:    /srv/mv-lab/src/play-source.txt
    move_dst:    /srv/mv-lab/archive/play-source-moved.txt
    config_dst:  /srv/mv-lab/archive/service.conf
    new_config_body: "config v1 (managed by Ansible)\n"

  tasks:
    # PATTERN 1 — move existing data (Ansible Boundary; no native module)
    - name: Move existing data with command:mv (boundary; idempotent via creates:/removes:)
      ansible.builtin.command:
        cmd: "mv {{ move_src }} {{ move_dst }}"
        creates: "{{ move_dst }}"     # skip task if dst already exists
        removes: "{{ move_src }}"     # skip task if src is already gone
      register: move_result

    # PATTERN 2 — atomic config replace using ansible.builtin.copy
    # (Ansible writes to a temp file, then renames into place — atomic)
    - name: Atomic config replace with ansible.builtin.copy
      ansible.builtin.copy:
        content: "{{ new_config_body }}"
        dest: "{{ config_dst }}"
        owner: root
        group: root
        mode: '0644'
        backup: true
      register: copy_result

    - name: Show results
      ansible.builtin.debug:
        msg:
          - "move task changed: {{ move_result.changed }}"
          - "copy task changed: {{ copy_result.changed }}"
          - "backup file: {{ copy_result.backup_file | default('n/a') }}"
EOF
```

Check-mode first:

```bash
ansible-playbook --check --diff /root/rhcsa_journal/lab10/task4/playbooks/move.yml \
  2>&1 | sudo tee /root/rhcsa_journal/lab10/task4/check.log
```

Apply:

```bash
ansible-playbook /root/rhcsa_journal/lab10/task4/playbooks/move.yml \
  2>&1 | sudo tee /root/rhcsa_journal/lab10/task4/apply.log
```

Idempotence proof:

```bash
ansible-playbook /root/rhcsa_journal/lab10/task4/playbooks/move.yml \
  2>&1 | sudo tee /root/rhcsa_journal/lab10/task4/rerun.log
grep '^localhost' /root/rhcsa_journal/lab10/task4/rerun.log
```

### Human-Readable Breakdown

**Pattern 1 — `command: mv` with `creates:` / `removes:`:**

`ansible.builtin.command` is the only way to invoke `mv` in Ansible (there is no module). To make the call idempotent, the module supports two short-circuit parameters:

- `creates: PATH` — if PATH exists, skip the task (return `changed=False` without running)
- `removes: PATH` — if PATH does NOT exist, skip the task

Together, `creates: move_dst` + `removes: move_src` means: "run `mv` only if dst doesn't exist AND src does exist." On the first run, both conditions are satisfied → mv runs → src vanishes and dst appears. On the second run, dst exists → `creates:` triggers the skip → `changed=False`. That's the RHCE-defensible idempotent move.

**Pattern 2 — `ansible.builtin.copy` for atomic config replace:**

When the operational goal is "atomically replace a config file," Ansible's `copy` module is the right answer, NOT `mv`. Internally `ansible.builtin.copy` writes the new content to a temp file in the same directory as `dest:`, sets its permissions, then calls `rename()` — which is atomic on a single filesystem. With `backup: true`, the old config is saved as `dest.<timestamp>~` before the rename. This is exactly what a careful admin would do by hand, but as a single idempotent module call.

### Reading It Left to Right

```yaml
ansible.builtin.command:
  cmd: "mv /srv/mv-lab/src/play-source.txt /srv/mv-lab/archive/play-source-moved.txt"
  creates: "/srv/mv-lab/archive/play-source-moved.txt"
  removes: "/srv/mv-lab/src/play-source.txt"
```

- `ansible.builtin.command:` — exec a binary (no shell)
- `cmd:` — the `mv` invocation
- `creates: PATH` — skip if PATH exists
- `removes: PATH` — skip if PATH doesn't exist

```yaml
ansible.builtin.copy:
  content: "config v1\n"
  dest: /srv/mv-lab/archive/service.conf
  backup: true
```

- `content:` — body to write
- `dest:` — destination (atomic rename target)
- `backup: true` — keep timestamped backup of overwritten dest

### The Story

A grader's RHCE question: "rotate `/var/log/app.log` to `/var/log/app.log.1` only if `app.log.1` does not already exist." The Ansible answer is exactly the `command: mv` pattern above with `creates:` set to the destination. A grader's separate question: "deploy `service.conf` to `/etc/myservice/`, preserving the previous version." The answer is `ansible.builtin.copy: backup: true` — NOT `mv`, because `copy` handles atomicity and backup in one call.

### Expected Output

First apply:

```
TASK [Move existing data with command:mv (boundary; idempotent via creates:/removes:)] ***
changed: [localhost]

TASK [Atomic config replace with ansible.builtin.copy] ***
changed: [localhost]

TASK [Show results] ***
ok: [localhost] => msg: ["move task changed: True", "copy task changed: True", "backup file: /srv/mv-lab/archive/service.conf.12345.2026-05-27@15:04:01~"]

PLAY RECAP ***
localhost : ok=3 changed=2 unreachable=0 failed=0
```

Idempotence rerun:

```
TASK [Move existing data ...] ***
ok: [localhost]                    <-- creates: matched; skipped

TASK [Atomic config replace ...] ***
ok: [localhost]                    <-- content already matches; no change

PLAY RECAP ***
localhost : ok=3 changed=0 unreachable=0 failed=0
```

### Switches Table

| Switch / Key | Meaning | Why it matters |
|---|---|---|
| `ansible.builtin.command:` | Exec binary (no shell) | Used when no real module exists |
| `creates: PATH` | Skip task if PATH exists | The RHCE idempotence trick for commands |
| `removes: PATH` | Skip task if PATH doesn't exist | Symmetric guard |
| `ansible.builtin.copy:` | Atomic content write to dest | RHCE-canonical for config replace |
| `backup: true` | Keep timestamped backup of dest before overwrite | Equivalent of `mv -b` |
| `content:` | Inline literal content | Alternative to `src:` |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| `command: mv` + `creates:` | Ansible Boundary pattern for moving existing data |
| `ansible.builtin.copy` | The RHCE answer for atomic config replace (not `mv`) |
| `backup: true` | Like `mv -b` — timestamped backup of overwritten dest |
| Idempotence | First run = changed; second = `changed=0` via `creates:` or content-match |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **T10-C** | Using `command: mv` WITHOUT `creates:` — every run shows `changed=1` and fails after src is gone | Always pair `command: mv` with `creates: DST` and `removes: SRC` |
| **Wrapping `command: mv` when you actually want atomic config replace** | Wrong tool for the job | Use `ansible.builtin.copy` (or `template:`) for content-based config replacement |
| `shell: mv` with redirects | Adds shell parsing where unnecessary; harder to read | Use `command:` unless you genuinely need shell features |

### 🔁 Persistence Check

```bash
test -f /srv/mv-lab/archive/play-source-moved.txt && echo "moved ok"
test ! -f /srv/mv-lab/src/play-source.txt        && echo "src gone"
grep -c 'config v1' /srv/mv-lab/archive/service.conf
ls /srv/mv-lab/archive/service.conf*~ 2>/dev/null && echo "backup present"
grep -c 'changed=0' /root/rhcsa_journal/lab10/task4/rerun.log
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab10/task4/done.txt > /dev/null <<EOF
lab=10 task=4
when=$(date -Is)
playbook=/root/rhcsa_journal/lab10/task4/playbooks/move.yml
move_done=$(test -f /srv/mv-lab/archive/play-source-moved.txt && echo yes || echo no)
src_gone=$(test ! -f /srv/mv-lab/src/play-source.txt && echo yes || echo no)
config_replaced=$(grep -c 'config v1' /srv/mv-lab/archive/service.conf)
backup_present=$(ls /srv/mv-lab/archive/service.conf*~ 2>/dev/null | wc -l)
idempotent_rerun_changed_0=$(grep -c 'changed=0' /root/rhcsa_journal/lab10/task4/rerun.log)
EOF
cat /root/rhcsa_journal/lab10/task4/done.txt
```

### 🧹 Cleanup

Leave files; Task 5 verifies them.

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Second run shows `changed=1` on the `mv` task | `creates:` path is wrong — must equal `move_dst` exactly |
| `mv: cannot stat 'SRC'` on first run | `removes:` will catch this; check src path |
| `copy` task `changed=1` on every run | `content:` ends with a different newline than what's on disk — match exactly (mind trailing `\n`) |

> **STOP — confirm `idempotent_rerun_changed_0 >= 2` in done.txt before Task 5.**

---

## Task 5 — RHCSA Verification Capstone: Prove the Moves Worked and Are Reboot-Safe

**Practice directory this task:** `/srv/mv-lab`

### 🔁 Warm-Up — Commands from Previous Labs

```bash
sudo mkdir -p /root/rhcsa_journal/lab10/task5
date -Is | sudo tee /root/rhcsa_journal/lab10/task5/start.txt
echo "exit was: $?"
```

### Purpose

Use **only** RHCSA inspection commands to prove:

1. The moved file lives at the destination, src is gone
2. The config replacement has the new content
3. The backup file exists with the old content
4. mtime + mode + owner all match what the playbook claimed

### Main Command Block

Three+ RHCSA inspection commands:

```bash
# 1) File-presence inspection
ls -la /srv/mv-lab/src/play-source.txt 2>&1 | head -1
ls -la /srv/mv-lab/archive/play-source-moved.txt
test -e /srv/mv-lab/src/play-source.txt        && echo "SRC_STILL_THERE" || echo "SRC_GONE"
test -e /srv/mv-lab/archive/play-source-moved.txt && echo "DST_PRESENT"

# 2) Content + metadata
cat /srv/mv-lab/archive/service.conf
stat -c 'mode=%a owner=%U:%G size=%s' /srv/mv-lab/archive/service.conf

# 3) Backup present
ls -la /srv/mv-lab/archive/service.conf*~
backup=$(ls /srv/mv-lab/archive/service.conf*~ | head -1)
cat "$backup"
[ "$(cat "$backup")" = "old config v0" ] && echo "BACKUP_CONTENT_MATCHES_OLD"

# 4) SELinux + DAC sanity
ls -lZ /srv/mv-lab/archive/

# Capture combined evidence
{
  echo "=== src/dst ==="
  test -e /srv/mv-lab/src/play-source.txt && echo "SRC_STILL_THERE" || echo "SRC_GONE"
  test -e /srv/mv-lab/archive/play-source-moved.txt && echo "DST_PRESENT"
  echo "=== config content ==="
  cat /srv/mv-lab/archive/service.conf
  echo "=== backup ==="
  ls /srv/mv-lab/archive/service.conf*~
  cat $(ls /srv/mv-lab/archive/service.conf*~ | head -1)
  echo "=== DAC + SELinux ==="
  ls -lZ /srv/mv-lab/archive/service.conf
  echo "=== match expected ==="
  [ "$(cat /srv/mv-lab/archive/service.conf)" = "config v1 (managed by Ansible)" ] && echo "CONFIG_MATCH"
  test ! -e /srv/mv-lab/src/play-source.txt && test -e /srv/mv-lab/archive/play-source-moved.txt && echo "MOVE_MATCH"
} 2>&1 | sudo tee /root/rhcsa_journal/lab10/task5/evidence.txt
```

### Human-Readable Breakdown

The audit:

- `ls -la` of src and dst — proves the move actually happened (src gone, dst present)
- `cat` of the config — proves the playbook content reached the file
- `ls -la` of `*~` backup — proves the backup mechanism worked
- `cat` of the backup — proves the OLD content was preserved before overwrite
- `ls -lZ` — DAC + SELinux audit (no surprises after the move + copy)
- String equality tests at the end — programmatic "everything matches" check

### Reading It Left to Right

`test -e PATH && echo "X" || echo "Y"`

- `test -e PATH` — does PATH exist? (follows symlinks)
- `&& ... || ...` — short-circuit ternary

`[ "$(cat FILE)" = "EXPECTED" ] && echo MATCH`

- `[` — `test` builtin
- `"$(cat FILE)"` — command substitution to get file content
- `=` — string equality
- `"EXPECTED"` — literal compare string

### The Story

You hand a grader `evidence.txt` and it reads: "src is gone, dst is present, config content matches the playbook string exactly, backup file exists with the old content, and DAC + SELinux look normal." That's the complete audit of an Ansible-driven move + atomic config replace.

### Expected Output

```
=== src/dst ===
SRC_GONE
DST_PRESENT

=== config content ===
config v1 (managed by Ansible)

=== backup ===
/srv/mv-lab/archive/service.conf.12345.2026-05-27@15:04:01~
old config v0

=== DAC + SELinux ===
-rw-r--r--. 1 root root unconfined_u:object_r:var_t:s0 31 May 27 15:04 service.conf

=== match expected ===
CONFIG_MATCH
MOVE_MATCH
```

### Switches Table

| Switch | Meaning | Why it matters |
|---|---|---|
| `test -e PATH` | Path exists (follows symlinks) | Move check |
| `test ! -e PATH` | Path does NOT exist | Source-gone check |
| `cat FILE` + `=` | String equality on contents | Programmatic config match |
| `ls -lZ` | DAC + SELinux | Post-move audit |
| `ls *~` | List backup files | Backup mechanism check |

### 🧠 Concept Card

| Concept | One-Line |
|---|---|
| Move audit | src absent + dst present + content correct + backup present |
| Reboot reasoning | `/srv/` survives reboot; the moved file stays moved; the backup stays as a snapshot |
| Auditor reflex | Always verify with ≥3 RHCSA commands; check BOTH src absence and dst presence |

| 🪤 Trap Risk | What goes wrong | How to avoid |
|---|---|---|
| **Trusting `ansible-playbook`'s changed=1 without inspecting state** | Whole point of the verification capstone | Verify with `test -e`, `cat`, `ls -lZ` |
| Checking only dst | Move "succeeded" but src still exists somehow (failed mv, copy without rm) | ALWAYS check `test ! -e SRC` as well |

### 🔁 Persistence Check (Reboot Reasoning)

```bash
echo "REBOOT REASONING:"                                                                | sudo tee /root/rhcsa_journal/lab10/task5/reboot.txt
echo "1. /srv/ persists across reboot; move + content change persist."                 | sudo tee -a /root/rhcsa_journal/lab10/task5/reboot.txt
echo "2. The backup file (*~) survives — useful for rolling back."                     | sudo tee -a /root/rhcsa_journal/lab10/task5/reboot.txt
echo "3. The Ansible playbook itself persists in /root/rhcsa_journal/."                | sudo tee -a /root/rhcsa_journal/lab10/task5/reboot.txt
test -f /root/rhcsa_journal/lab10/task4/playbooks/move.yml && echo "playbook persists" | sudo tee -a /root/rhcsa_journal/lab10/task5/reboot.txt
test -f /srv/mv-lab/archive/play-source-moved.txt && echo "move persists"              | sudo tee -a /root/rhcsa_journal/lab10/task5/reboot.txt
```

### 📓 Journal Write

```bash
sudo tee /root/rhcsa_journal/lab10/task5/done.txt > /dev/null <<EOF
lab=10 task=5
when=$(date -Is)
evidence=/root/rhcsa_journal/lab10/task5/evidence.txt
reboot=/root/rhcsa_journal/lab10/task5/reboot.txt
config_match=$(grep -c '^CONFIG_MATCH$' /root/rhcsa_journal/lab10/task5/evidence.txt)
move_match=$(grep -c '^MOVE_MATCH$' /root/rhcsa_journal/lab10/task5/evidence.txt)
status=lab10-complete
EOF
cat /root/rhcsa_journal/lab10/task5/done.txt
```

### 🧹 Cleanup (No Regression)

```bash
# Remove the entire sandbox
sudo rm -rf /srv/mv-lab
ls -d /srv/mv-lab 2>&1 | grep -q "No such" && echo "sandbox cleaned"

# Journal stays
ls /root/rhcsa_journal/lab10/
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `CONFIG_MATCH` missing | The `content:` body in playbook doesn't match — mind trailing newlines |
| `MOVE_MATCH` missing | src still exists (mv didn't run) OR dst missing (mv failed) — re-run Task 4 |
| Multiple `*~` backups | Each apply creates one; clean up old backups with `find ... -delete` |

> **STOP — record `status=lab10-complete` in done.txt. Lab 10 is finished.**

---

## ✅ Lab 10 Complete When

```bash
ls /root/rhcsa_journal/lab10/task{1,2,3,4,5}/done.txt
grep -l 'lab10-complete' /root/rhcsa_journal/lab10/task5/done.txt
test -f /root/rhcsa_journal/lab10/task4/playbooks/move.yml
grep -cE '(CONFIG|MOVE)_MATCH' /root/rhcsa_journal/lab10/task5/evidence.txt
```

All four must succeed. You can rename and move files safely in shell, distinguish atomic same-fs renames from non-atomic cross-fs moves, replicate with `command: mv + creates:`, and use `ansible.builtin.copy` for atomic config replacement.
