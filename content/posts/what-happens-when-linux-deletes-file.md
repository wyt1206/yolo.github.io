---
title: "What Really Happens When Linux Deletes a File?"
date: 2026-08-05
draft: false
tags:
  - Linux
  - File System
  - inode
  - dentry
  - unlink
  - File Descriptor
  - VFS
categories:
  - Many WHYs
---
# What Really Happens When Linux Deletes a File?

When we type:

```bash
rm test.txt
```

Most people think:

> Linux deletes the file immediately.

But this is not exactly true.

Linux does not directly erase data.

The real question:

> What does deleting a file actually mean?

---

# 1. What Happens When We Execute rm?

First:

```bash
rm test.txt
```

is not a kernel operation.

`rm` is a user-space program.

Check:

```bash
which rm
```

Example:

```text
/bin/rm
```

---

Execution chain:

```text
User

 |

 v

Shell

 |

 v

rm Process

 |

 v

unlink() syscall

 |

 v

Kernel VFS

 |

 v

Filesystem
```

---

Important:

`rm` does not delete file content itself.

It asks the kernel:

```c
unlink("test.txt");
```

---

# 2. What Does unlink() Actually Do?

The name is confusing.

`unlink` does not mean:

> erase file

It means:

> remove the link between filename and inode.

---

Linux filesystem structure:

```text
Filename

    |

    v

Dentry

    |

    v

Inode

    |

    v

Data Blocks
```

---

Example:

Before:

```text
/home/user/test.txt

        |

        v

     inode 1234

        |

        v

    data blocks
```

---

After:

```bash
unlink("test.txt")
```

becomes:

```text
filename

        X

inode 1234

        |

        v

data blocks
```

---

The directory entry is removed.

The inode may still exist.

---

# 3. Why Doesn't Linux Directly Delete The inode?

Because Linux supports:

```text
Multiple Hard Links
```

---

Example:

Create:

```bash
echo hello > a.txt
```

inode:

```text
inode 100
```

---

Create hard link:

```bash
ln a.txt b.txt
```

Now:

```text
a.txt
   |
   |
 inode 100
   |
   |
b.txt
```

---

If you delete:

```bash
rm a.txt
```

Should Linux delete inode 100?

No.

Because:

```text
b.txt
```

still points to it.

---

Therefore inode contains:

```text
link count
```

Example:

```text
inode

links = 2
```

---

After:

```bash
rm a.txt
```

becomes:

```text
links = 1
```

---

Only when:

```text
links == 0
```

can inode be released.

---

# 4. When Is inode Really Freed?

Linux checks two conditions:

## Condition 1

No directory entry points to inode:

```text
link count = 0
```

---

## Condition 2

No process is using the file:

```text
open file count = 0
```

---

Only then:

```text
inode freed

+

data blocks returned
```

---

Full lifecycle:

```text
Filename removed

        |

        v

Dentry removed

        |

        v

inode link count decreases

        |

        v

Check open references

        |

        v

Free inode

        |

        v

Free data blocks
```

---

# 5. What Does "File Still Has Open FD" Mean?

Example:

Terminal 1:

```bash
vim test.txt
```

File opened.

Kernel creates:

```text
Process

 |

 v

File Descriptor Table

 |

 v

struct file

 |

 v

inode
```

---

Now Terminal 2:

```bash
rm test.txt
```

The filename disappears.

But vim still has:

```text
open file reference
```

---

The file becomes:

```text
(deleted)
```

but data still exists.

---

You can see:

```bash
lsof | grep deleted
```

Example:

```text
vim

/tmp/test.txt (deleted)
```

---

This is why:

> Linux allows deleting files that are still open.

---

# 6. Why Does df Show No Space Released After rm?

Classic Linux interview question.

Example:

```bash
rm big.log
```

but:

```bash
df -h
```

space unchanged.

Why?

Because:

```text
inode still referenced
```

---

Example:

Process:

```text
Application

 |

 v

File Descriptor

 |

 v

inode

 |

 v

data blocks
```

---

Filename is gone:

```text
rm
```

but:

```text
open fd
```

still exists.

---

The disk space is released only when:

```text
last fd closed
```

---

Example:

Restart application:

```bash
systemctl restart app
```

Then:

```text
fd closed

↓

inode released

↓

space returned
```

---

# 7. How To Completely Delete A File?

Normal case:

```bash
rm file
```

is enough.

---

If space is not released:

Find deleted open files:

```bash
lsof | grep deleted
```

---

Example:

```text
java

/tmp/log (deleted)
```

---

Solution:

Restart process:

```bash
systemctl restart service
```

or:

close the file descriptor.

---

Important:

Removing filename ≠ freeing storage.

---

# 8. Why Is rm -rf Slow For 100 Million Files?

Suppose:

```text
directory

 |
 |
 + file1
 + file2
 + ...
 + file100000000
```

---

`rm -rf` needs:

For every file:

1. lookup dentry
2. unlink filename
3. update inode
4. update metadata
5. release blocks
6. update journal

---

One file:

```text
O(1)
```

---

One hundred million files:

```text
100,000,000 operations
```

---

The bottleneck is not deleting data.

It is:

```text
metadata operations
```

---

Filesystem spends time on:

* directory lookup
* inode updates
* journal writes
* cache misses

---

# 9. How To Delete 100 Million Files Faster?

## Method 1: Delete Entire Directory

Instead of:

```bash
rm -rf bigdir/*
```

remove filesystem tree:

```bash
rm -rf bigdir
mkdir bigdir
```

---

Why faster?

Because:

Deleting directory metadata:

```text
remove one namespace
```

instead of:

```text
unlink millions of files
```

---

## Method 2: Use Filesystem Snapshot

For large systems:

Examples:

* ZFS snapshot
* LVM snapshot

Delete snapshot:

```text
metadata operation
```

---

## Method 3: Use Application Design

Avoid:

```text
millions of small files
```

Use:

* database
* object storage
* log rotation

---

# 10. The Complete Linux Delete Flow

Putting everything together:

```text
rm

 |

 v

unlink()

 |

 v

VFS

 |

 v

Remove dentry

 |

 v

Decrease inode link count

 |

 v

Check open references

 |

 v

Free inode

 |

 v

Free data blocks
```

---

# Final Mental Model

Deleting a file is not:

```text
delete filename

↓

erase disk
```

It is:

```text
Remove name

↓

Remove directory entry

↓

Decrease inode reference

↓

Wait for open references

↓

Release storage
```

---

# Connection With Previous ManyWhys Articles

This article connects:

```text
Everything is File

        |

        v

File Descriptor

        |

        v

struct file

        |

        v

inode

        |

        v

Filesystem
```

And:

```text
lsof

 |

 v

open fd

 |

 v

deleted file still exists
```

---

# Conclusion

The key idea:

> Linux does not delete files. Linux removes references to files.

A file disappears when nobody can reach it anymore:

```text
No filename

+

No hard link

+

No open FD

=

File finally deleted
```
