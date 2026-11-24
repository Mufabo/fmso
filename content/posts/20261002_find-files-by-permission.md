---
title: 'Finding files by permission with find'
date: 2026-10-02
tags: [linux, shell]
description: 'How find -perm works with exact, "-" (all bits) and "/" (any bit) modes, worked through a permission-audit example.'
draft: false
---

Permission audits are a common sysadmin task: "which files can the group write?", "which files are hidden from others?". `find` answers these with its `-perm` test, which has three forms. A bare mode matches exactly, a `-` prefix requires **all** the given bits, and a `/` prefix requires **any** of them.[^1]

## Scenario: group-writable, hidden from others

**Find all files and directories under `/var/log/` where the group can write, but others have neither read nor write access, and save the full paths to a text file.**

```bash
find /var/log -perm -020 ! -perm /006 > /home/bob/data.txt 2>/dev/null
```

*Or using symbolic notation:*

```bash
find /var/log -perm -g=w ! -perm /o=rw > /home/bob/data.txt 2>/dev/null
```

### Command breakdown

1. **`find /var/log`**
   * Starts searching at `/var/log`. Because the starting point is an absolute path, `find` prints the full path of every result (e.g., `/var/log/nginx/access.log`).
2. **`-perm -020`** *(or `-perm -g=w`)*
   * **`020`** is the octal group-write bit.
   * The **`-`** prefix requires **at least** this bit to be set (group can write).
3. **`! -perm /006`** *(or `! -perm /o=rw`)*
   * **`006`** is the sum of **4** (other read) and **2** (other write).
   * The **`/`** prefix matches if **any** of the given bits is set (others can read OR write).
   * The **`!`** negates the test, so a file matches only if **neither** the other-read nor the other-write bit is set.
4. **`> /home/bob/data.txt`**
   * Redirects the list of matching paths into `/home/bob/data.txt`.
5. **`2>/dev/null`** *(optional but recommended)*
   * Sends standard error to `/dev/null`, so "Permission denied" warnings for subdirectories of `/var/log/` you can't read don't clutter the output.

The command matches files and directories alike. Add `-type f` (only files) or `-type d` (only directories) to narrow it down.

### Alternative: the built-in `-fprint` action

Instead of shell redirection (`>`), you can let `find` write the output file itself:

```bash
find /var/log -perm -020 ! -perm /006 -fprint /home/bob/data.txt 2>/dev/null
```

## Matching "at least read and write" for the group

To match files or directories where the group has **at least read and write** permissions, use the `-` prefix with octal **`060`** or symbolic **`g=rw`**:

```bash
# Octal notation
find /path/to/search -perm -060

# Symbolic notation
find /path/to/search -perm -g=rw
```

### How it works

* **`060` in octal:**
  * Group read (4) + group write (2) = **6**.
  * Placed in the group slot: `060` (owner `0`, group `6`, others `0`).
* **The `-` prefix ("all of these bits"):**
  * **Both** the group-read and the group-write bit must be set.
  * The group *may* also have execute (e.g., `760` or `770`), and owner and others can have any permissions. Anything where the group lacks read or write is filtered out.

## Cheat sheet: `-perm`, `-perm -`, `-perm /`

| Command | Requirement | Matches modes like... |
| --- | --- | --- |
| `find . -perm -060` | Group has **both** read AND write (at least); other bits are ignored | `660`, `770`, `664` |
| `find . -perm /060` | Group has **either** read OR write; other bits are ignored | `640`, `620`, `664` |
| `find . -perm 060` | The **entire** mode is exactly `060` | `060` (`----rw----`) only |

Note the last row: without a prefix, `-perm` compares the whole permission set, so owner and others must have *no* permissions at all. A typical `664` file does **not** match `-perm 060`. That's why you'll want the `-` or `/` form almost every time.

[^1]: [find(1) — Linux manual page](https://man7.org/linux/man-pages/man1/find.1.html), GNU findutils via man7.org.
