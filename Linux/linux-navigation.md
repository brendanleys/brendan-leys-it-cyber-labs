# Linux Filesystem Navigation — Lab Notes

Date: May 14, 2026
Environment: WSL2 (Ubuntu) on Windows
Time spent: 2 hours

---

## Commands Practiced and What Each Does

### pwd — Print Working Directory

  $ pwd
  /home/brendan

What it does: Shows the full path of the directory you are currently in. Always run this first if you are confused about where you are in the filesystem. The output is your current location — every command you run operates relative to this location unless you give a full path.

---

### ls — List Directory Contents

  $ ls
  file1.txt  file2.txt  file3.sh  archive/

  $ ls -l
  total 12
  -rw-r--r-- 1 brendan brendan  12 May 14 09:01 file1.txt
  -rw-r--r-- 1 brendan brendan   0 May 14 09:01 file2.txt
  -rwxr-xr-x 1 brendan brendan  32 May 14 09:01 file3.sh
  drwxr-xr-x 2 brendan brendan  40 May 14 09:01 archive/

  $ ls -la
  (same as ls -l but also shows hidden files starting with a dot)

ls         shows filenames only.
ls -l      long format: shows permissions, owner, size, modification date.
ls -la     same as ls -l but includes hidden files (names starting with .).

---

### mkdir — Make Directory

  $ mkdir -p ~/practice/week1/linux_nav
  $ cd ~/practice/week1/linux_nav
  $ pwd
  /home/brendan/practice/week1/linux_nav

What it does: Creates a new directory. The -p flag creates all parent directories automatically. Without it, the command fails if the parent does not already exist.

---

### touch — Create an Empty File

  $ touch file1.txt file2.txt file3.sh
  $ ls
  file1.txt  file2.txt  file3.sh

What it does: Creates an empty file. If the file already exists, it updates its timestamp without changing its contents. You can create multiple files at once by listing them separated by spaces.

---

### cp — Copy a File

  $ cp file1.txt file1_backup.txt
  $ ls
  file1.txt  file1_backup.txt  file2.txt  file3.sh

What it does: Makes a copy of a file with a new name. The original is unchanged. Format: cp source destination

---

### mv — Move or Rename a File

  $ mkdir archive
  $ mv file2.txt archive/file2.txt
  $ ls
  archive/  file1.txt  file1_backup.txt  file3.sh
  $ ls archive/
  file2.txt

What it does: Moves a file to a new location. If the destination is in the same directory with a different name, it renames the file. Format: mv source destination

---

### cat, head, tail — View File Contents

  $ echo 'Hello Linux' > file1.txt
  $ cat file1.txt
  Hello Linux

  $ head -5 /etc/passwd
  root:x:0:0:root:/root:/bin/bash
  daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
  (first 5 lines shown)

  $ tail -5 /etc/passwd
  (last 5 lines shown)

cat          prints the entire file. Use for short files only.
less         opens the file in a scrollable viewer. Press q to quit, spacebar to scroll.
head -5      prints the first 5 lines (default 10 without a number).
tail -5      prints the last 5 lines. tail -f follows a live file as it grows.

---

### rm — Remove a File

  $ rm file1_backup.txt
  $ ls
  archive/  file1.txt  file3.sh

What it does: Permanently deletes a file. There is no recycle bin on Linux. Always double-check before running rm. Use ls first to confirm you are targeting the right file.

---

## Key Takeaways

- pwd tells you where you are. Run it whenever you are confused.
- ls -l shows permissions and metadata. ls -la includes hidden files.
- mkdir -p creates parent directories automatically.
- cat for short files, less for long ones, tail -f for live logs.
- rm is permanent — verify before running.
