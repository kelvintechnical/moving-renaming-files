# Lab: Moving and Renaming Files — `mv`

**Series:** linux-ops-mastery — RHCSA Essential Tools & File Operations
**Subjects covered:** `mv` for rename within a directory, `mv` for relocation across directories, the difference between rename (atomic, inode unchanged) and cross-filesystem move (`copy + unlink`), `-i` interactive, `-n` no-clobber, `-f` force, `-u` update-only-if-newer, `-v` verbose, `-T` treat-as-file, multiple sources with `-t TARGET`, the trailing-slash gotcha, and how `mv` preserves SELinux context, owner, group, mode, and timestamps (without any extra flags)
**Career arcs covered:** RHCSA (every "rename / move" exam task), RHCE (Ansible `copy:` + cleanup or `command: mv`), SRE (renaming log files for log rotation, swap-and-replace deploys), DevOps (build artifact relocation), AI/MLOps (atomic rename for safe checkpoint writes — `cp tmp; fsync; mv tmp final`)
**Prerequisite:** Labs 05–09 (navigation, listing, copy, links)
**Time Estimate:** 25 to 35 minutes
**Difficulty arc:** Task 1 foundation (`mv` rename) · 2 move across directories · 3 atomic vs cross-FS · 4 safety flags · 5 multi-source with `-t` · 6 RHCSA exam-realistic capstone

---

## Objective

Move and rename files like you mean it — with the right safety flags, the right understanding of "atomic vs not," and the right reflex for the rename-temp-then-mv-final pattern that powers safe writes everywhere from `sed -i` to database WAL files.

The capstone is an exam-realistic prompt: *"In `/root/files/`, rename every file matching `*.txt` to have a `.txt.bak` suffix in one safe operation, and relocate the originals' renamed copies into `/root/files-archive/`. Verify that the originals are gone, the new names exist, and no data was lost."*

> **Lab safety note:** All experiments happen in `/tmp/mv-lab` and (briefly) `/root/files`. Nothing modifies system files.

---

## Concept: A "Move" Is Either a Rename or a Copy-Then-Delete

`mv` does **two completely different things** depending on whether source and destination share a filesystem:

- **Same filesystem:** `mv` calls `rename(2)`. The directory entry is rewritten in place; the inode is unchanged; the operation is **atomic** (either it happens or it does not — nothing in between). Free.
- **Different filesystems:** `mv` copies the bytes to the new filesystem, then unlinks the old name. The new file has a new inode. The operation is **not atomic** — power-loss midway can leave a partial copy on the destination.

```
   ┌──────────────────────────────────────────────────────┐
   │   mv /home/file /home/folder/file                    │
   │                                                       │
   │   Same FS → rename(2)  → atomic, inode unchanged     │
   │   No data copy. Fast. Open file descriptors keep     │
   │   working without interruption.                       │
   ├──────────────────────────────────────────────────────┤
   │   mv /home/file /mnt/usb/file                        │
   │                                                       │
   │   Different FS → copy + unlink                       │
   │   New inode. Slow on large files. Open FDs to the    │
   │   original keep working only on the source side.     │
   └──────────────────────────────────────────────────────┘
```

> **Why this matters:** Atomic rename is **the** safe-write primitive in Unix. Editors, databases, package managers, and log rotators all rely on it. Cross-filesystem `mv` does not get you that guarantee — and the failure mode (partial copy, original still present) can surprise you in scripts.

---

## 📜 Why `mv` Exists — The Story

In **Unix v1 (1971)**, every directory-entry change required two syscalls: link the file under the new name, then unlink the old. That's a non-atomic window where the file briefly has two names — fine on a single-user machine, dangerous on multi-user systems where another process might see both names at once.

By **System V (1983)** the `rename(2)` syscall existed: one atomic operation that handles both the link and the unlink in the kernel. Crashes either complete the operation or not — there is no half-state visible to userspace.

That syscall is the entire reason `mv` exists as a separate command from `ln + rm`. It is also why `mv` is the only Unix file operation that can guarantee atomicity when used carefully (same FS, no overwrite issues).

> **The point of the story:** `mv` is not "move." `mv` is "atomic rename, with cross-filesystem copy as a fallback." Knowing which mode you are in tells you whether the operation is safe to interrupt.

---

## 👪 The mv Family — Who Lives There

### `mv` flags

| Flag | Meaning |
|---|---|
| (default) | Move (rename or copy+unlink) |
| `-i` | Interactive — prompt before overwrite |
| `-n` | No-clobber — skip if destination exists |
| `-f` | Force — overwrite without prompt (default) |
| `-u` | Update — overwrite only if source is newer |
| `-v` | Verbose |
| `-T` | Treat destination as a non-directory (avoid auto-renaming into directory) |
| `-t TARGET` | Specify target dir; sources follow |

### Related rename helpers

| Tool | Notes |
|---|---|
| `rename 's/OLD/NEW/' FILES` | Perl rename — regex bulk rename (RHEL: `prename`) |
| `for f in *.x; do mv "$f" "${f%.x}.y"; done` | Pure-bash bulk rename |
| `install -m MODE src DST` | Copy with explicit mode (not strictly move) |
| `cp -a SRC DST && rm -rf SRC` | Two-step "move" with metadata preserved |

> **The point of the family tree:** `mv` is one command, but its two modes (rename vs copy+unlink) have very different semantics. The flags exist mostly to control overwrite behavior, not to change which mode is used.

---

## 🔬 The Anatomy of `mv` — In One Diagram

```
$ mv /home/user/a.txt /home/user/folder/

Behavior:
  1. stat /home/user/a.txt    — find inode + filesystem
  2. stat /home/user/folder/  — find inode + filesystem
  3. Are they on the same filesystem?
       YES → rename(2)                    (atomic; inode unchanged)
        NO → cp -a SRC DST then rm SRC    (slow; new inode)

Trailing slash:
  mv src.txt dst.txt          → rename file
  mv src.txt dst              → if dst is a directory: places src AS dst/src
                                if dst does not exist: renames src to dst
  mv src.txt dst/             → trailing / forces "dst is a directory" expectation
                                (errors if dst does not exist as a dir)

Multiple sources:
  mv a b c /target/           → /target/a, /target/b, /target/c
  mv -t /target/ a b c        → same, but target specified first (script-friendly)
```

> **Reading rule:** Always test `mv` with `--no-target-directory` (`-T`) when you really mean "I want this exact name on the destination, not 'this name placed inside a directory of that name.'"

---

## 📚 mv Reference Table

| Task | Command | Notes |
|---|---|---|
| Rename in place | `mv old new` | If `new` does not exist |
| Move into a directory | `mv src dir/` | `src` ends up as `dir/src` |
| Move and rename in one step | `mv src dir/newname` | Atomic on same FS |
| Multiple sources | `mv a b c /target/` | Target must be a directory |
| Multiple sources (target-first form) | `mv -t /target/ a b c` | Useful with `xargs` / loops |
| Interactive | `mv -i a b` | Y/N on overwrite |
| No-clobber | `mv -n a b` | Skip if dst exists |
| Update-only | `mv -u a b` | Move only if src newer than dst |
| Force | `mv -f a b` | Default; rarely needed |
| Treat dst as file even if dir exists | `mv -T a b` | Avoid auto-into-directory placement |
| Verbose | `mv -v a b` | Print operations |
| Atomic-write safe pattern | `cmd > FILE.tmp && mv FILE.tmp FILE` | Avoid partial-write corruption |
| Bulk rename `.txt` → `.bak` | `for f in *.txt; do mv "$f" "${f%.txt}.bak"; done` | Bash parameter expansion |

> **Rule one of mv:** Same-FS `mv` is atomic and cheap. Cross-FS `mv` is `cp -a; rm -rf` under the hood — slow and not atomic. Always verify which mode you're in before relying on atomicity guarantees.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | "Rename `/etc/foo.conf` to `/etc/foo.conf.bak` and create a new `/etc/foo.conf`" is a canonical question. |
| **RHCE candidate** | Ansible `command: mv ...` or `copy: + file: state=absent` — both follow `mv` semantics. |
| **SRE / Platform** | Atomic-write-then-rename is **the** standard for safe config rewrites: `cmd > /etc/conf.new && mv /etc/conf.new /etc/conf`. |
| **DevOps** | Build outputs typically use `mv` to publish artifacts only when checksums pass. |
| **AI / MLOps** | Checkpoint writes follow `torch.save(state, "ckpt.tmp"); os.replace("ckpt.tmp", "ckpt.pt")` — the Python equivalent of atomic `mv`. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **rename → move → verify → preserve metadata** habit.

---

### Task 1 — Rename in place

**Purpose:** Set up the sandbox and use `mv` to rename a file within the same directory; observe that the inode does not change.

```bash
mkdir -p /tmp/mv-lab && cd /tmp/mv-lab

echo "rename test" > old-name.txt
ls -li old-name.txt

mv old-name.txt new-name.txt
ls -li new-name.txt
```

**Human-Readable Breakdown:** Build the sandbox, create `old-name.txt`, capture its inode, rename to `new-name.txt`, capture the inode again. Same inode — bytes never moved.

**Reading it left to right:** `mv old new` calls `rename(2)`. The kernel rewrites the directory entry from "old → inode N" to "new → inode N." Atomic, fast, zero data movement.

**The story:** Watch the inode column once and you stop worrying about "is mv expensive on a 100 GB file?" If it's the same filesystem, mv is microseconds regardless of size.

**Expected output:**

```text
12345 -rw-r--r--. 1 ec2-user ec2-user 12 May 26 14:10 old-name.txt
12345 -rw-r--r--. 1 ec2-user ec2-user 12 May 26 14:10 new-name.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `mv old new` | Rename (or move into dir) |
| `ls -li FILE` | Long listing with inode number |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mv: cannot stat 'old-name.txt'` | File does not exist — `ls` to confirm |
| Inode changed | You crossed filesystems — `mv` did copy+unlink |
| `mv: 'new-name.txt' and 'old-name.txt' are the same file` | Tried to rename to itself |

---

### Task 2 — Move across directories on the same filesystem

**Purpose:** Use `mv` to relocate files between directories on the same filesystem. Observe atomicity and that the inode is still preserved.

```bash
cd /tmp/mv-lab
mkdir -p src dst

echo "alpha" > src/a.txt
echo "bravo" > src/b.txt

ls -li src/a.txt
mv src/a.txt dst/
ls -li dst/a.txt
mv src/b.txt dst/b-renamed.txt
ls -li dst/b-renamed.txt

ls -R
```

**Human-Readable Breakdown:** Make `src/` and `dst/`, populate `src/`, move one file as-is into `dst/`, move another file with a rename. Confirm inodes stay constant.

**Reading it left to right:** `mv src/a.txt dst/` is `rename("src/a.txt", "dst/a.txt")` — single syscall. `mv src/b.txt dst/b-renamed.txt` is the same syscall with a different second arg. Trailing slash on destination tells `mv` to keep the source filename.

**The story:** This is what `mv` is for 95% of the time — relocate or rename within one filesystem. Always atomic. Always cheap.

**Expected output:**

```text
12346 -rw-r--r--. 1 user user 6 May 26 14:11 src/a.txt
12346 -rw-r--r--. 1 user user 6 May 26 14:11 dst/a.txt
12347 -rw-r--r--. 1 user user 6 May 26 14:11 dst/b-renamed.txt
.:
dst  src

./dst:
a.txt  b-renamed.txt

./src:
```

**Switches**

| Token | Meaning |
|---|---|
| `mv FILE DIR/` | Move into directory (keep filename) |
| `mv FILE DIR/NEWNAME` | Move and rename in one step |
| `ls -R` | Recursive listing |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mv: target 'dst/' is not a directory` | `dst/` does not exist — `mkdir -p` first |
| File replaced silently | Default `mv` is `-f` — use `-i` or `-n` for safety |
| Inode changed | You crossed filesystems — check `df -T`/`stat` |

---

### Task 3 — Cross-filesystem move = copy + delete

**Purpose:** Demonstrate that moving across filesystems is not atomic — bytes get copied, original is deleted, and the inode number changes.

```bash
cd /tmp/mv-lab

echo "data" > original.txt
ls -li original.txt
df -T /tmp /dev/shm 2>/dev/null | head -n 3

# /dev/shm is tmpfs (separate FS from / on most systems)
mv original.txt /dev/shm/original.txt 2>/dev/null
ls -li /dev/shm/original.txt
ls original.txt 2>&1 | head -n 1

# Bring it back
mv /dev/shm/original.txt /tmp/mv-lab/
ls -li /tmp/mv-lab/original.txt
```

**Human-Readable Breakdown:** Create a file under `/tmp` (the regular root filesystem on most setups), confirm `/dev/shm` is a different filesystem (`tmpfs`), and move the file there. Notice the **new inode number** at the destination — the bytes really did travel.

**Reading it left to right:** `df -T` shows the filesystem type per mount. `/tmp` is usually on `/`; `/dev/shm` is `tmpfs`. `mv` detects different filesystems, falls back to copy+unlink, and the destination inode is new.

**The story:** Knowing when `mv` is atomic and when it isn't is the difference between safe and unsafe scripts. "Always use temp-and-rename for atomic writes" only works if the temp lives on the same filesystem as the final.

**Expected output:**

```text
12348 -rw-r--r--. 1 user user 5 May 26 14:12 original.txt
Filesystem     Type  1K-blocks  Used  Available  Use%  Mounted on
/dev/nvme0n1p1 xfs   ...
tmpfs          tmpfs ...                                /dev/shm
98765 -rw-r--r--. 1 user user 5 May 26 14:12 /dev/shm/original.txt
ls: cannot access 'original.txt': No such file or directory
12349 -rw-r--r--. 1 user user 5 May 26 14:12 /tmp/mv-lab/original.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `df -T PATH` | Show filesystem type for path |
| `/dev/shm` | A tmpfs filesystem usually distinct from `/` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Inode did not change | `/dev/shm` is on the same FS — try another mount point |
| `Permission denied` on `/dev/shm` | Owner restrictions; `sudo` or pick a different mount |
| `mv` left source file behind | Indicates a partial-copy failure mid-write |

---

### Task 4 — Safety flags: `-i`, `-n`, `-u`, `-v`

**Purpose:** Practice the four safety flags so destructive mistakes during interactive `mv` are caught.

```bash
cd /tmp/mv-lab
rm -rf src dst
mkdir src dst
echo "src content" > src/a.txt
echo "dst content" > dst/a.txt

# Interactive
mv -i src/a.txt dst/    # answer 'n' to keep existing dst
cat dst/a.txt

# No-clobber
echo "new src" > src/a.txt
mv -n src/a.txt dst/
cat dst/a.txt           # unchanged
ls src/                 # src/a.txt is STILL THERE (no-clobber refuses)

# Update — only move if src newer
touch -d "1 hour ago" src/a.txt
mv -u src/a.txt dst/
cat dst/a.txt           # unchanged
touch src/a.txt
mv -u src/a.txt dst/
cat dst/a.txt           # changed

# Verbose
echo "moveme" > src/b.txt
mv -v src/b.txt dst/
```

**Human-Readable Breakdown:** Walk through `-i` (asks), `-n` (refuses), `-u` (only-if-newer), `-v` (loud). Each flag has a different defensive purpose.

**Reading it left to right:** `-i` produces a prompt. `-n` silently skips the move when destination exists. `-u` compares mtimes. `-v` prints `'src' -> 'dst'`.

**The story:** Production scripts that loop `mv` over thousands of files need `-n` and `-v` — `-n` prevents disasters; `-v` produces an audit trail.

**Expected output:**

```text
mv: overwrite 'dst/a.txt'? n
dst content
dst content
a.txt
dst content
src content
'src/b.txt' -> 'dst/b.txt'
```

**Switches**

| Token | Meaning |
|---|---|
| `-i` | Prompt before overwrite |
| `-n` | Refuse if destination exists |
| `-u` | Move only if source mtime > dst mtime |
| `-v` | Print operations |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-i` did not prompt | Script context — stdin not a tty |
| `-n` skipped silently | That is the point |
| `-u` did the wrong thing | Compare mtimes manually with `stat -c '%y'` |

---

### Task 5 — Multiple sources and the `-t` target-first form

**Purpose:** Move many files into one directory in one command; use `-t` so the target is named **before** the sources (essential when generating mv calls from `xargs`).

```bash
cd /tmp/mv-lab
rm -rf src dst
mkdir src dst
touch src/{1..5}.log

# Target-last form (default)
mv src/1.log src/2.log src/3.log dst/
ls dst/

# Target-first form with -t
mv -t dst/ src/4.log src/5.log
ls dst/

# Same with brace expansion (within bash)
mkdir src2
touch src2/{a,b,c,d}.log
mv -v src2/*.log dst/
ls dst/

# Pipe-friendly with xargs
mkdir src3
touch src3/extra1.log src3/extra2.log
ls src3/*.log | xargs mv -t dst/
ls dst/
```

**Human-Readable Breakdown:** Move a batch of files in three different styles: standard `mv A B C DIR/`, the `-t TARGET` form (target named first), and a generator (`ls | xargs mv -t DST`).

**Reading it left to right:** `mv -t TARGET sources...` exists because some shells / pipelines find it easier to vary the source list than the target. With `xargs`, this lets you stream filenames into a single `mv` invocation efficiently.

**The story:** Once you write your first batch-move shell loop, you appreciate `-t`. It makes `xargs` and `find -exec mv -t TARGET {} +` clean.

**Expected output:**

```text
1.log  2.log  3.log
1.log  2.log  3.log  4.log  5.log
'src2/a.log' -> 'dst/a.log'
'src2/b.log' -> 'dst/b.log'
'src2/c.log' -> 'dst/c.log'
'src2/d.log' -> 'dst/d.log'
1.log 2.log 3.log 4.log 5.log a.log b.log c.log d.log
1.log 2.log 3.log 4.log 5.log a.log b.log c.log d.log extra1.log extra2.log
```

**Switches**

| Token | Meaning |
|---|---|
| `mv -t TARGET ...` | Target-first form |
| `xargs mv -t DST` | Pipe filenames into a single `mv` |
| `touch {1..5}.log` | Brace expansion — create five files |
| `find ... -exec mv -t DIR {} +` | Bulk `find` rename pattern |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mv: missing destination file operand` | Forgot the target or supplied none — add `-t DIR` |
| `mv: target 'X' is not a directory` | The target is a regular file — pre-create the directory |
| `xargs: mv: argument list too long` | Use `find ... -exec mv -t DIR {} +` instead of `xargs` for very long lists |

---

### Task 6 — Capstone: RHCSA-realistic bulk rename + relocation

**Task statement:** *"In `/root/files/`, rename every `*.txt` file to have a `.txt.bak` suffix and relocate every `.bak` into `/root/files-archive/`. Verify no `.txt` files remain in `/root/files/`, that every `.bak` exists in `/root/files-archive/`, and that no data was lost."*

**Purpose:** Execute a real exam-style bulk rename + relocation, with verification.

```bash
sudo -i
rm -rf /root/files /root/files-archive
mkdir -p /root/files /root/files-archive

for i in 1 2 3 4 5; do
  echo "content $i" > /root/files/report$i.txt
done
ls /root/files/

# Bulk rename to .txt.bak using bash parameter expansion
cd /root/files
for f in *.txt; do
  mv -v -- "$f" "${f}.bak"
done

# Relocate all .bak files to the archive directory
mv -v -t /root/files-archive/ /root/files/*.bak

ls /root/files/
ls /root/files-archive/

# Verify
test ! -f /root/files/report1.txt && echo "VERIFY: original .txt gone"
test    -f /root/files-archive/report1.txt.bak && echo "VERIFY: renamed file relocated"

# Byte-level integrity
for f in /root/files-archive/*.bak; do
  echo "$f: $(wc -c < "$f") bytes"
done
```

**Human-Readable Breakdown:** Become root, create five test files in `/root/files`. Rename each `*.txt` to `*.txt.bak` using a loop with bash parameter expansion (`${f}.bak`). Then move every `.bak` into `/root/files-archive/` with `mv -t`. Verify originals are gone, destinations exist, and byte counts are sane.

**Layer stack you built:**

```text
/root/files/                  (started with 5 *.txt files; now empty)
   │
   │  for f in *.txt; do mv "$f" "${f}.bak"; done   ← in-place rename
   │
   ▼
/root/files/*.txt.bak         (intermediate state — both rename steps used `mv`)
   │
   │  mv -t /root/files-archive/ /root/files/*.bak  ← relocate
   │
   ▼
/root/files-archive/          (now contains 5 *.txt.bak files)
   ├── report1.txt.bak
   ├── report2.txt.bak
   ├── report3.txt.bak
   ├── report4.txt.bak
   └── report5.txt.bak
```

**The story:** This is the **canonical 90-second exam answer** for bulk rename. Memorize the spine: `for f in *.EXT; do mv "$f" "${f%.EXT}.NEW"; done` for in-place renames, `mv -t DST SRC1 SRC2 ...` for relocation. Variants are infinite; the pattern is fixed.

**Expected verification output:**

```text
report1.txt  report2.txt  report3.txt  report4.txt  report5.txt
renamed 'report1.txt' -> 'report1.txt.bak'
renamed 'report2.txt' -> 'report2.txt.bak'
...
renamed '/root/files/report1.txt.bak' -> '/root/files-archive/report1.txt.bak'
...

report1.txt.bak  report2.txt.bak  report3.txt.bak  report4.txt.bak  report5.txt.bak
VERIFY: original .txt gone
VERIFY: renamed file relocated
/root/files-archive/report1.txt.bak: 10 bytes
/root/files-archive/report2.txt.bak: 10 bytes
/root/files-archive/report3.txt.bak: 10 bytes
/root/files-archive/report4.txt.bak: 10 bytes
/root/files-archive/report5.txt.bak: 10 bytes
```

**Cleanup**

```bash
rm -rf /tmp/mv-lab /root/files /root/files-archive
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Filenames with spaces broke | Always quote `"$f"` |
| `*.bak` glob did not match | No `.bak` files in the dir — confirm rename step worked |
| Wanted to use `rename` (Perl) | `rename 's/\.txt$/.txt.bak/' *.txt` is the one-liner equivalent |
| `Permission denied` writing to `/root/files-archive` | Not root — `sudo -i` |

---

## 🔍 mv Decision Guide

```
Need to move or rename a file?
  │
  ├── "Same name, different directory"
  │       └── ✅ mv src dst/
  │
  ├── "Same directory, different name"
  │       └── ✅ mv old new
  │
  ├── "Both at once"
  │       └── ✅ mv src dst/newname
  │
  ├── "Many sources, one destination"
  │       └── ✅ mv a b c /target/
  │       └── ✅ mv -t /target/ a b c
  │
  ├── "Be paranoid about overwriting"
  │       └── ✅ mv -i src dst       (prompt)
  │       └── ✅ mv -n src dst       (skip if dst exists)
  │       └── ✅ mv -u src dst       (only if newer)
  │
  ├── "Atomic safe-write of a config file"
  │       └── ✅ cmd > /etc/conf.tmp && mv /etc/conf.tmp /etc/conf
  │
  ├── "Bulk regex rename"
  │       └── ✅ rename 's/\.old$/.new/' *.old      (RHEL: prename)
  │
  └── "Cross-filesystem move of huge file"
          └── ⚠️ Be aware: mv falls back to cp -a + rm; not atomic.
                 Consider rsync -aP for resumability.
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Set up `/tmp/mv-lab` and rename a file in place; inode unchanged
- [ ] 02 Move across directories on the same filesystem; inode unchanged
- [ ] 03 Move to a different filesystem (`/dev/shm`); observe new inode
- [ ] 04 Practice `-i`, `-n`, `-u`, `-v` safety flags
- [ ] 05 Move multiple sources with `-t TARGET` and `xargs mv -t`
- [ ] 06 Execute the RHCSA capstone — bulk rename `*.txt → *.txt.bak` and relocate to archive

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Trailing `/` on destination that does not exist | `mv: ... is not a directory` | Drop `/` or `mkdir -p` first |
| Renamed onto an existing file by accident | Original silently overwritten | Use `-i` interactively or `-n` in scripts |
| Cross-FS `mv` of huge file mid-power-loss | Partial copy lingers | Use `rsync -aP --remove-source-files` |
| Forgot to quote filename with spaces | `mv: cannot stat 'a'` (when name was `a b`) | Always quote `"$f"` |
| Used `mv` expecting atomic write across filesystems | Not atomic | Keep temp file on same FS, then `mv` |
| `mv DIR1 DIR2/` while `DIR2/DIR1` existed | Got "directory not empty" | Pre-clean target, or use `rsync` |
| Bulk rename via `mv *.txt prefix*` | Got "target 'prefix*' is not a directory" | Use a `for` loop with parameter expansion |
| Used `mv` then needed to undo | No undo | Always `cp -a` to staging first if irreversible |
| Forgot `mv` updates ctime | Audit trail tampering | Use `cp -a` + `rm` if you must preserve ctime |
| `mv` of open file | Renames the directory entry; open FDs still work | This is by design (Unix semantics) |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- The two patterns to memorize: in-place rename loop `for f in *.X; do mv "$f" "${f%.X}.Y"; done` and bulk relocate `mv -t /target/ *.bak`.

**RHCE candidate**
- Ansible has no first-class `mv` module; use `command: mv ...` with `creates:` / `removes:` guards for idempotence, or `copy: + file: state=absent`.

**SRE / Platform interview**
- "Walk me through safely rewriting `/etc/sshd_config`." → `cat > /etc/sshd_config.tmp; mv /etc/sshd_config.tmp /etc/sshd_config; sshd -t; systemctl reload sshd`. Atomic rename + config check + reload.

**DevOps**
- Build pipelines write to `dist.tmp/`, run integrity checks, then `mv dist.tmp dist` — atomic publish.

**AI / MLOps**
- PyTorch's `torch.save(state, "ckpt.tmp"); os.replace("ckpt.tmp", "ckpt.pt")` is the Python equivalent of atomic `mv` — never leaves a partially-written checkpoint visible.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Directory Navigation | The path-discipline foundation |
| Lab 08 — Copying Files | `mv` is `cp + rm` when crossing filesystems |
| Lab 09 — Hard and Soft Links | `mv` of a hard link keeps the link; `mv` of a symlink moves the link, not the target |
| Lab 11 — Safe Deletion | The cleanup tool — `mv` and `rm` are companions |
| Lab 12 — Creating Nested Directories | Often paired with `mv` to relocate trees |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
