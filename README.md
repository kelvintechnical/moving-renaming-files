# Lab 10: Moving and Renaming Files — `mv`

**Series:** File Operations & Shell Fundamentals · **Lab 10 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (Tasks 8, 11, 16, 18, 20), RHCE EX294 (no native mv module — patterns matter), CKA (start/stop static pods via mv), RHCA building blocks (RH342, RH358)  
**Prerequisite:** Labs 05–09  
**Time Estimate:** 40–55 minutes  
**Difficulty arc:** Tasks 1–6 foundation · 7–13 practical · 14–18 advanced · 19–20 exam-realistic

---

## 🎯 Objective

Master `mv` for both renaming (same directory) and relocating (different directory) files and directories. Understand the **two different things** `mv` does internally — instant rename on same filesystem vs `cp -a` + `rm` across filesystems — and the SELinux gotcha that breaks more services than any other single mistake.

---

## 🧠 Concept: `mv` Does Two Different Things

Internally, the kernel handles two cases:

| Case | What actually happens | Speed | SELinux risk |
|---|---|---|---|
| **Same filesystem** | Rewrites the directory entry; inode unchanged | Instant, even for 100 GB files | **None** — context follows the inode |
| **Different filesystem** | Effectively `cp -a SRC DST && rm SRC` | Slow (copies every byte) | **Yes** — new inode may inherit a different default context |

> **The killer scenario:** moving `/home/user/site/` (same FS, context `user_home_t`) to `/var/www/html/`. Inode unchanged — but **SELinux context is still `user_home_t`**. Apache returns 403. Same-FS `mv` does **not** relabel.

```
Same FS mv:    rename entry → inode unchanged → context unchanged
Cross-FS mv:   cp -a + rm   → new inode       → context = target dir default (unless -Z used)
```

---

## 📚 `mv` Reference

| Flag | Long form | Purpose |
|---|---|---|
| `-i` | `--interactive` | Prompt before overwrite |
| `-n` | `--no-clobber` | Never overwrite |
| `-f` | `--force` | Overwrite without prompting (default; defeats `-i` alias) |
| `-u` | `--update` | Move only if source is newer or destination missing |
| `-v` | `--verbose` | Print each move |
| `-b` | `--backup[=METHOD]` | Backup destination before overwriting |
| `--backup=numbered` | — | Numbered backups |
| `-T` | `--no-target-directory` | Treat destination as a regular file (prevents "moved into" surprise) |
| `-t DIR` | `--target-directory=DIR` | Destination first — handy with xargs/find |
| `-Z` | — | Set destination context to target dir's default |
| `--strip-trailing-slashes` | — | Remove trailing `/` from sources |

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **RHCSA EX200** | Tasks 8, 11, 16, 18, 20 use `mv` plus restorecon |
| **RHCE EX294** | No native `mv` module — patterns are `copy + file: state=absent` or `command: mv` |
| **CKA** | Move static-pod manifests in/out of `/etc/kubernetes/manifests` to start/stop components |
| **RHCA — RH342** | Quarantine pattern: `mv suspect.bin /tmp/quarantine/` |
| **RHCA — RH358** | Service-config rollovers: `mv config.new config.old` |

---

## 🔧 The 20 Tasks

---

### Task 1 — Set up source files

**Purpose:** Build a workspace with varied content.

```bash
mkdir -p ~/mv-lab/{src,dst,archive}
cd ~/mv-lab
echo "alpha"   > src/alpha.txt
echo "beta"    > src/beta.txt
echo "gamma"   > src/gamma.log
mkdir src/sub
echo "deep"    > src/sub/deep.txt
ls -lR
```

**Expected output (excerpt):**

```
src:
total 12
-rw-r--r--. 1 ec2-user ec2-user 6 Sep 12 14:00 alpha.txt
-rw-r--r--. 1 ec2-user ec2-user 5 Sep 12 14:00 beta.txt
-rw-r--r--. 1 ec2-user ec2-user 6 Sep 12 14:00 gamma.log
drwxr-xr-x. 2 ec2-user ec2-user 22 Sep 12 14:00 sub
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p` | Create with parents |
| `{src,dst,archive}` | Brace expansion → three dirs in one command |
| `ls -lR` | Long, recursive listing |

**Output decoded**

| Entry | Meaning |
|---|---|
| Three `.txt`/`.log` files | Regular files |
| `sub` | Subdirectory |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mkdir: cannot create` | Path inside your home |

---

### Task 2 — Rename a file in place

**Purpose:** Most basic case — both arguments in the same directory.

```bash
cd ~/mv-lab/src
ls -li alpha.txt
mv alpha.txt alpha-renamed.txt
ls -li alpha-renamed.txt
```

**Expected output:**

```
1048601 -rw-r--r--. 1 ec2-user ec2-user 6 Sep 12 14:00 alpha.txt
1048601 -rw-r--r--. 1 ec2-user ec2-user 6 Sep 12 14:00 alpha-renamed.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `mv OLD NEW` | If both are in the same directory, this is a **rename** |

**Output decoded**

| Token | Meaning |
|---|---|
| Inode `1048601` matches | Confirms **inode unchanged** — no data copied |

> Linux has no separate `rename` in coreutils — `mv` IS rename. (Perl ships a `rename`; util-linux ships a different one.) For exams, stick with `mv`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mv: cannot stat 'alpha.txt'` | Already renamed — re-run from Task 1 setup |

---

### Task 3 — Move a file into a directory

**Purpose:** Move without renaming. Trailing `/` makes the directory intent explicit.

```bash
cd ~/mv-lab
mv src/beta.txt dst/
ls dst/ src/
```

**Expected output:**

```
beta.txt
alpha-renamed.txt  gamma.log  sub
```

**Switches**

| Token | Meaning |
|---|---|
| `mv FILE DIR/` | Trailing `/` = destination is a directory; file keeps its name |

**Output decoded**

| Path | Effect |
|---|---|
| `dst/beta.txt` | Now lives in `dst/` |
| `src/` | No longer contains `beta.txt` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Move silently created a file named `dst` | `dst/` didn't exist — `mkdir -p dst` first |

---

### Task 4 — Move AND rename in one step

**Purpose:** Specify both destination directory and new filename.

```bash
mv src/gamma.log dst/gamma-archived.log
ls dst/
```

**Expected output:**

```
beta.txt  gamma-archived.log
```

**Switches**

| Token | Meaning |
|---|---|
| `mv FILE DIR/NEWNAME` | Move and rename atomically |

**Output decoded**

| File | Meaning |
|---|---|
| `gamma-archived.log` | Same file as `src/gamma.log`, new name and location |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Destination already exists | `mv` will overwrite without prompt unless `-i` or `-n` |

---

### Task 5 — Move multiple files into a directory

**Purpose:** When you give `mv` ≥ 2 sources, the **last argument must be a directory**.

```bash
mv src/sub src/alpha-renamed.txt dst/
ls dst/ src/ 2>/dev/null
```

**Expected output:**

```
alpha-renamed.txt  beta.txt  gamma-archived.log  sub
(src/ is empty)
```

**Switches**

| Token | Meaning |
|---|---|
| `mv f1 f2 ... DIR/` | Move all sources into DIR |

**Output decoded**

| Token | Meaning |
|---|---|
| Whole `sub/` moved | `mv` handles directories natively (no `-r` needed, unlike `cp`) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mv: target ... is not a directory` | Last arg must be a directory when ≥ 2 sources |

---

### Task 6 — Interactive overwrite with `-i`

**Purpose:** Prevent accidental clobbering.

```bash
echo "old beta"  > dst/beta.txt
echo "new beta"  > src/beta.txt
mv -i src/beta.txt dst/
```

**Expected interaction:**

```
mv: overwrite 'dst/beta.txt'? y
```

**Switches**

| Flag | Meaning |
|---|---|
| `-i` | Prompt before each overwrite |

**Output decoded**

| Prompt | Action |
|---|---|
| `overwrite 'dst/beta.txt'?` | Type `y` or `n` |

**Why on RHCA RH342:** Quarantine and rollback steps — always `-i` when destinations might exist.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Unexpected prompt | `alias mv='mv -i'` is active — type `\mv` to bypass once |

---

### Task 7 — Never overwrite with `-n`

**Purpose:** Idempotent: only move if destination is missing.

```bash
echo "stay here" > dst/keepme.txt
echo "noop" > src/keepme.txt
mv -n src/keepme.txt dst/
cat dst/keepme.txt
```

**Expected output:**

```
stay here
```

**Switches**

| Flag | Meaning |
|---|---|
| `-n` | No-clobber — skip if destination exists |

**Output decoded**

| Line | Meaning |
|---|---|
| `stay here` | `dst/keepme.txt` survived; `mv -n` did nothing |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Combined `-n -i` | `-n` wins — silent skip, no prompt |

---

### Task 8 — Force overwrite with `-f`

**Purpose:** Bypass alias-induced prompts. `-f` is the default behavior of unmangled `mv`.

```bash
echo "clobber me" > dst/keepme.txt
echo "winner" > src/keepme.txt
mv -f src/keepme.txt dst/
cat dst/keepme.txt
```

**Expected output:**

```
winner
```

**Switches**

| Flag | Meaning |
|---|---|
| `-f` | Force overwrite without prompting |

**Output decoded**

| Line | Meaning |
|---|---|
| `winner` | Destination was replaced |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Permission denied | Need write on the parent directory |

---

### Task 9 — Backup before overwrite with `-b`

**Purpose:** Auto-keep the previous version when overwriting.

```bash
echo "v1" > dst/configme.txt
echo "v2" > src/configme.txt
mv -b src/configme.txt dst/
ls dst/configme*
cat dst/configme.txt
cat dst/configme.txt~
```

**Expected output:**

```
dst/configme.txt  dst/configme.txt~
v2
v1
```

**Switches**

| Flag | Meaning |
|---|---|
| `-b` | Backup destination before overwriting |
| `--backup=numbered` | Use `.~1~`, `.~2~` numbering |

**Output decoded**

| File | Meaning |
|---|---|
| `dst/configme.txt` | New content (`v2`) |
| `dst/configme.txt~` | Previous content (`v1`) saved automatically |

**Why a sysadmin needs this:** "Edit a config" tasks where you want a `.bak` left behind for rollback.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Backup not appearing | If destination didn't exist, no backup needed |

---

### Task 10 — Numbered backups for repeated moves

**Purpose:** Multiple iterations need distinguishable backups.

```bash
echo "v3" > src/configme.txt
mv --backup=numbered src/configme.txt dst/configme.txt
echo "v4" > src/configme.txt
mv --backup=numbered src/configme.txt dst/configme.txt
ls dst/configme*
```

**Expected output:**

```
dst/configme.txt  dst/configme.txt.~1~  dst/configme.txt.~2~  dst/configme.txt~
```

**Switches**

| Flag | Meaning |
|---|---|
| `--backup=numbered` | Numbered (`.~1~`, `.~2~`, ...) |
| `--backup=simple` | Always `~` suffix |
| `--backup=existing` | Match existing style |
| `--backup=none` | No backup (default) |

**Output decoded**

| File | Meaning |
|---|---|
| `configme.txt` | Current (latest `v4`) |
| `.~1~`, `.~2~` | Numbered prior versions |
| `configme.txt~` | The Task 9 backup that remained |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Backups pile up over time | Periodic cleanup with `find dst -name '*~*~' -mtime +30 -delete` |

---

### Task 11 — Verbose with `-v`

**Purpose:** Watch what `mv` does — essential for multi-file jobs.

```bash
mkdir -p ~/mv-lab/staging
touch ~/mv-lab/staging/{a,b,c,d,e}.tmp
mv -v ~/mv-lab/staging/*.tmp ~/mv-lab/archive/
```

**Expected output:**

```
renamed '/home/ec2-user/mv-lab/staging/a.tmp' -> '/home/ec2-user/mv-lab/archive/a.tmp'
renamed '/home/ec2-user/mv-lab/staging/b.tmp' -> '/home/ec2-user/mv-lab/archive/b.tmp'
renamed '/home/ec2-user/mv-lab/staging/c.tmp' -> '/home/ec2-user/mv-lab/archive/c.tmp'
renamed '/home/ec2-user/mv-lab/staging/d.tmp' -> '/home/ec2-user/mv-lab/archive/d.tmp'
renamed '/home/ec2-user/mv-lab/staging/e.tmp' -> '/home/ec2-user/mv-lab/archive/e.tmp'
```

**Switches**

| Flag | Meaning |
|---|---|
| `-v` | Verbose — `renamed 'src' -> 'dst'` per move |

**Output decoded**

| Line | Meaning |
|---|---|
| Each `renamed` line | One file's source → destination |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No output | Source glob didn't match — verify with `ls` first |

---

### Task 12 — Same-FS move keeps the inode

**Purpose:** Prove that same-filesystem `mv` is metadata-only (no data movement).

```bash
echo "stay same" > ~/mv-lab/same-fs.txt
INODE_BEFORE=$(stat -c '%i' ~/mv-lab/same-fs.txt)
mv ~/mv-lab/same-fs.txt ~/mv-lab/archive/
INODE_AFTER=$(stat -c '%i' ~/mv-lab/archive/same-fs.txt)
echo "Before: $INODE_BEFORE"
echo "After : $INODE_AFTER"
```

**Expected output:**

```
Before: 1048622
After : 1048622
```

**Switches**

| Token | Meaning |
|---|---|
| `stat -c '%i'` | Print inode number |

**Output decoded**

| Token | Meaning |
|---|---|
| Identical inode | Same-FS `mv` is a directory-entry rewrite — instant, no data copy |

**Why a sysadmin needs this:** Moving a 200 GB file across the same FS is **instant**. Try it across mounts and watch hours pass.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Inodes differ | You crossed a filesystem boundary — check with `df` |

---

### Task 13 — Cross-FS move changes the inode

**Purpose:** Demonstrate that across mounts, `mv` performs a copy + delete.

```bash
df ~/mv-lab /tmp | column -t
echo "cross fs" > ~/mv-lab/cross.txt
INODE_BEFORE=$(stat -c '%i' ~/mv-lab/cross.txt)
mv ~/mv-lab/cross.txt /tmp/
INODE_AFTER=$(stat -c '%i' /tmp/cross.txt)
echo "Before: $INODE_BEFORE"
echo "After : $INODE_AFTER"
```

**Expected output (if `/tmp` is tmpfs):**

```
Filesystem  1K-blocks  Used    Available  Use%  Mounted on
/dev/root   8000000    3500000 4500000    44%   /home/ec2-user/mv-lab
tmpfs       500000     400     499600     1%    /tmp
Before: 1048623
After : 9123201
```

**Switches**

| Token | Meaning |
|---|---|
| `df FILE` | Show which filesystem the file lives on |

**Output decoded**

| Field | Meaning |
|---|---|
| Different mount points | Different filesystems |
| Different inodes | Data was copied; old inode released |

**Why a sysadmin needs this on RHCA RH358:** Performance — never use `mv` to relocate huge files across mounts in production; use `rsync --remove-source-files` so you can resume on interruption.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Inodes match (no change) | Both paths are on the same FS — pick truly different mount |

---

### Task 14 — The SELinux trap (same-FS) — context survives

**Purpose:** Same-FS move does **not** relabel. This breaks more service tasks than any other single mistake.

```bash
mkdir -p ~/mv-lab/source_dir
echo "served file" > ~/mv-lab/source_dir/index.html
ls -lZ ~/mv-lab/source_dir/index.html

sudo mkdir -p /webroot
sudo mv ~/mv-lab/source_dir /webroot/site
sudo ls -lZ /webroot/site/index.html
```

**Expected output:**

```
unconfined_u:object_r:user_home_t:s0 /home/ec2-user/mv-lab/source_dir/index.html
unconfined_u:object_r:user_home_t:s0 /webroot/site/index.html
```

**Switches**

| Token | Meaning |
|---|---|
| `ls -lZ` | Show SELinux context |

**Output decoded**

| Observation | Meaning |
|---|---|
| Context **unchanged** after move | Same-FS `mv` keeps the inode; SELinux type rides along |
| `user_home_t` under `/webroot` | Wrong for a web root — Apache would 403 |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Stuck with wrong type | Fix it in Task 15 |

---

### Task 15 — Fix the SELinux trap with `restorecon`

**Purpose:** Recompute contexts from policy.

```bash
sudo restorecon -Rv /webroot
sudo ls -lZ /webroot/site/index.html
matchpathcon /webroot/site/index.html
```

**Expected output:**

```
Relabeled /webroot/site/index.html from unconfined_u:object_r:user_home_t:s0 to system_u:object_r:default_t:s0
unconfined_u:object_r:default_t:s0 /webroot/site/index.html
/webroot/site/index.html	system_u:object_r:default_t:s0
```

**Switches**

| Token | Meaning |
|---|---|
| `restorecon -R` | Recursive |
| `restorecon -v` | Verbose |
| `matchpathcon` | Show policy-defined context |

**Output decoded**

| Line | Meaning |
|---|---|
| `Relabeled ... from ... to ...` | The fix happened |
| Context now `default_t` | Matches `/webroot`'s policy expectation |

> If you want `httpd_sys_content_t` under `/webroot`, define it first: `sudo semanage fcontext -a -t httpd_sys_content_t '/webroot(/.*)?'` then `restorecon -Rv /webroot`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `restorecon` says no change | Source and target dir share type, or no policy for this path — define one with `semanage fcontext` |

---

### Task 16 — Inline context with `mv -Z`

**Purpose:** Set destination context to target dir's default — equivalent of inline `restorecon`.

```bash
echo "labeled at move" > ~/mv-lab/labeled.html
sudo mv -Z ~/mv-lab/labeled.html /webroot/
sudo ls -lZ /webroot/labeled.html
```

**Expected output:**

```
unconfined_u:object_r:default_t:s0 /webroot/labeled.html
```

**Switches**

| Flag | Meaning |
|---|---|
| `-Z` | Set context to target directory's default |

**Output decoded**

| Token | Meaning |
|---|---|
| `default_t` | Already correct — no separate `restorecon` needed |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mv: unrecognized option -Z` | Old coreutils — use plain `mv` + `restorecon` |

---

### Task 17 — Build commands with `-t DIR` for find/xargs

**Purpose:** "Target first" mode lets you pipe many sources from `find` or `xargs`.

```bash
mkdir -p ~/mv-lab/bulk_src ~/mv-lab/bulk_dst
touch ~/mv-lab/bulk_src/file_{01..10}.tmp
find ~/mv-lab/bulk_src -name '*.tmp' -exec mv -t ~/mv-lab/bulk_dst/ {} +
ls ~/mv-lab/bulk_dst/
```

**Expected output:**

```
file_01.tmp  file_02.tmp  file_03.tmp  file_04.tmp  file_05.tmp
file_06.tmp  file_07.tmp  file_08.tmp  file_09.tmp  file_10.tmp
```

**Switches**

| Token | Meaning |
|---|---|
| `-t DIR` | Target directory **first**; source names follow |
| `find -exec ... {} +` | Batch arguments together (efficient) |

**Output decoded**

| Files | Meaning |
|---|---|
| All 10 moved | `find` batched them; `mv -t DIR ... files...` handled them in groups |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mv: missing destination file operand` | Without `-t`, last arg must be the destination |

---

### Task 18 — Prevent "moved into" surprise with `-T`

**Purpose:** Tell `mv` "the destination is a regular file, not a directory" so it never descends.

```bash
mkdir -p ~/mv-lab/example_dest
echo "v1" > ~/mv-lab/source.txt
mv ~/mv-lab/source.txt ~/mv-lab/example_dest
ls ~/mv-lab/example_dest/

echo "v2" > ~/mv-lab/source.txt
mv -T ~/mv-lab/source.txt ~/mv-lab/example_dest 2>&1 || true
```

**Expected output:**

```
source.txt
mv: cannot overwrite directory '/home/ec2-user/mv-lab/example_dest' with non-directory
```

**Switches**

| Flag | Meaning |
|---|---|
| `-T` | Treat destination strictly as a regular file path |

**Output decoded**

| Phase | What happened |
|---|---|
| Without `-T` | File moved INTO `example_dest/` |
| With `-T` | `mv` refused (good — saves accidental nesting) |

**Why a sysadmin needs this:** Scripts that should rename, not relocate.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-T` not supported | Use absolute file path for destination instead of a directory |

---

### Task 19 — RHCSA-style scenario: rename + relocate + restorecon

**Task statement:** *"Rename `/etc/myapp.conf` to `/etc/myapp.conf.old`, then move `/tmp/myapp.conf` into `/etc/`. Verify SELinux contexts."*

```bash
sudo touch /etc/myapp.conf
echo "new config" | sudo tee /tmp/myapp.conf > /dev/null

sudo ls -lZ /etc/myapp.conf
sudo mv -v /etc/myapp.conf /etc/myapp.conf.old
sudo mv -v /tmp/myapp.conf /etc/myapp.conf

sudo ls -lZ /etc/myapp.conf /etc/myapp.conf.old
sudo restorecon -Rv /etc/myapp.conf /etc/myapp.conf.old
sudo ls -lZ /etc/myapp.conf /etc/myapp.conf.old
```

**Expected output (excerpts):**

```
unconfined_u:object_r:etc_t:s0 /etc/myapp.conf
renamed '/etc/myapp.conf' -> '/etc/myapp.conf.old'
renamed '/tmp/myapp.conf' -> '/etc/myapp.conf'
... shows context (likely user_tmp_t for the newly arrived file)
Relabeled /etc/myapp.conf from unconfined_u:object_r:user_tmp_t:s0 to system_u:object_r:etc_t:s0
... shows etc_t for both
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| First `mv` | Same-FS rename — instant |
| Second `mv` | Cross-FS (tmpfs `/tmp` → root `/etc`) — copies data, new inode, brings `user_tmp_t` |
| `restorecon -Rv` | Recompute contexts — fixes both files |
| `ls -lZ` before/after | Auditable proof |

**Output decoded**

| Phase | Context for `/etc/myapp.conf` |
|---|---|
| After mv | `user_tmp_t` (wrong) |
| After restorecon | `etc_t` (correct) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Context unchanged after restorecon | `/etc/myapp.conf` may be a new path — verify with `matchpathcon` |

---

### Task 20 — CKA-style scenario: toggle a static pod with mv

**Task statement (CKA-style):** *"Stop `kube-scheduler` by moving its manifest aside, then restart it."*

> ⚠️ Run only on a kubeadm cluster you control. On the lab VM, simulate it by creating a placeholder.

```bash
sudo mkdir -p /etc/kubernetes/manifests /tmp/manifest-backup
echo "apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system" | sudo tee /etc/kubernetes/manifests/kube-scheduler.yaml > /dev/null

ls -l /etc/kubernetes/manifests/

sudo mv -v /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/manifest-backup/
ls -l /etc/kubernetes/manifests/
ls -l /tmp/manifest-backup/

sudo mv -v /tmp/manifest-backup/kube-scheduler.yaml /etc/kubernetes/manifests/
ls -l /etc/kubernetes/manifests/
```

**Expected output (excerpts):**

```
-rw-r--r--. 1 root root ... /etc/kubernetes/manifests/kube-scheduler.yaml
renamed '/etc/kubernetes/manifests/kube-scheduler.yaml' -> '/tmp/manifest-backup/kube-scheduler.yaml'
(empty manifests dir)
-rw-r--r--. 1 root root ... /tmp/manifest-backup/kube-scheduler.yaml
renamed '/tmp/manifest-backup/kube-scheduler.yaml' -> '/etc/kubernetes/manifests/kube-scheduler.yaml'
-rw-r--r--. 1 root root ... /etc/kubernetes/manifests/kube-scheduler.yaml
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| First `mv` to `/tmp/manifest-backup/` | kubelet stops the static pod |
| Second `mv` back into manifests | kubelet restarts the static pod |

**Output decoded**

| Phase | Cluster state |
|---|---|
| File in manifests | Pod running |
| File missing | Pod terminated by kubelet |
| File back | Pod running again |

**Cleanup**

```bash
cd ~
rm -rf ~/mv-lab
sudo rm -rf /webroot /etc/myapp.conf /etc/myapp.conf.old /tmp/manifest-backup
sudo rm -f /tmp/cross.txt
sudo rm -rf /etc/kubernetes/manifests/kube-scheduler.yaml   # only in lab env
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Pod doesn't toggle | You're not on a kubelet-managed node — this is purely a demonstration |

---

## 🔍 `mv` Decision Guide

```
Renaming in the same directory?         → mv old new
Moving into a directory?                → mv file dir/
Moving and renaming together?           → mv file dir/newname
Multiple files into a directory?        → mv f1 f2 f3 dir/
Don't overwrite anything?               → mv -n
Prompt before overwrite?                → mv -i
Backup overwritten destination?         → mv -b   (or --backup=numbered)
Verbose tracking of multi-file moves?   → mv -v
Need correct SELinux context at target? → mv -Z   or   restorecon -Rv afterwards
Cross-FS move (slow, copies bytes)?     → mv works, but inode changes — verify
Build commands with find/xargs?         → mv -t DIR ...
Prevent "moved INTO" surprise?          → mv -T
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Set up `~/mv-lab/{src,dst,archive}`
- [ ] 02 Rename a file in place
- [ ] 03 Move a file into a directory
- [ ] 04 Move and rename in one step
- [ ] 05 Move multiple files into a directory
- [ ] 06 Interactive overwrite with `-i`
- [ ] 07 Skip overwrite with `-n`
- [ ] 08 Force overwrite with `-f`
- [ ] 09 Backup with `-b`
- [ ] 10 Numbered backups with `--backup=numbered`
- [ ] 11 Verbose with `-v`
- [ ] 12 Confirm same-FS move keeps the inode
- [ ] 13 Confirm cross-FS move changes the inode
- [ ] 14 Observe SELinux context surviving a same-FS move
- [ ] 15 Fix with `restorecon -Rv`
- [ ] 16 Apply target context inline with `mv -Z`
- [ ] 17 Use `mv -t DIR` with `find -exec`
- [ ] 18 Prevent "moved INTO" surprise with `-T`
- [ ] 19 Exam: rename + relocate + restorecon
- [ ] 20 CKA: toggle static-pod with `mv`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Renaming into a directory by mistake | File "disappears" into a folder | Use `-T` or check with `ls -ld dest` first |
| Plain `mv` into `/var/www/html` | Apache 403 (wrong SELinux type) | Add `restorecon -Rv` after the move |
| Trusting `mv` to carry contexts across FS | Owners/perms preserved; context may not | Run `restorecon -Rv` or pre-create `semanage fcontext` |
| Overwriting without a backup | No undo | `mv -b` or `--backup=numbered` |
| Using `*` glob — misses dotfiles | `.htaccess`, `.git` left behind | `mv src/. dst/` for contents |
| Moving in-use file across FS | Partial copy if interrupted | Prefer `rsync --remove-source-files` for big jobs |
| Forgetting `-v` on multi-file moves | Hard to confirm | Always `-v` for batch |
| Moving a symlink expecting target to follow | Symlink moves; target stays | Dereference first (`readlink -f`) or `cp -aL` + `rm` |

---

## 📌 Exam Strategy

**RHCSA EX200**
- Know which filesystem source and destination live on — `df SRC DST`.
- After any `mv` into an SELinux-aware path (`/var/www`, `/var/lib/*`, `/srv`, `/etc/*`), run `restorecon -Rv`.
- For "rename" tasks, the answer is `mv` — don't search for a `rename` command.

**RHCE EX294 (Ansible)**
- No native `mv` module. Idiomatic patterns:
  - `copy:` + `file: state=absent` (clearest, idempotent)
  - `command: mv ...` with `creates:` / `removes:` for idempotence

**CKA**
- Toggle control-plane: `mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/` then back.
- `mv` is the simplest way to stop a static pod cleanly.

**RHCA**
- RH342: quarantine pattern: `sudo mv suspect.bin /tmp/quarantine-$(date +%s)/` then investigate.
- RH358: rolling deploys — `mv config.new config.live && systemctl reload`.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Navigation | Knowing source/destination paths cold |
| Lab 06 — `ls -lZ` | Verification tool after every move |
| Lab 08 — `cp` | Cross-FS `mv` IS `cp -a` + `rm` |
| Lab 09 — Links | Moving a symlink ≠ moving its target |
| Lab 11 — `rm` | Same-FS `mv` = rename + unlink; cross-FS `mv` ends with `rm` |
| Task 16 — Apache document root | Move + restorecon |
| Task 20 — Config edits | `mv original original.bak` before editing |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
