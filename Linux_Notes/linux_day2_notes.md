## Day 2 — Creating, Copying, Moving and Deleting Files

---

### What You Will Learn Today

- mkdir — create folders
- touch — create files
- cp — copy files
- mv — move and rename files
- rm — delete files (carefully!)

---

### Setup — Create a Practice Folder First

Always create a safe practice area. Never experiment on real system files.

```bash
cd ~
mkdir linux_practice
cd linux_practice
```

---

### Command 1 — mkdir (Make Directory)

```bash
mkdir myfolder                      # create one folder
mkdir folder1 folder2 folder3       # create multiple at once
mkdir -p projects/devops/scripts    # create nested folders in one shot
```

**What is `-p`?**
- Without `-p`: `mkdir projects/devops/scripts` → ERROR (parent doesn't exist)
- With `-p`: creates all three levels automatically — no error

> ⚠️ If folder already exists Linux throws an error — will NOT silently overwrite.

---

### Command 2 — touch (Create Empty File)

```bash
touch myfile.txt                          # create empty file
touch file1.txt file2.txt file3.txt       # create multiple at once
touch deploy.sh config.yml .env           # create DevOps files
```

**Why called "touch"?** Originally designed to update a file's timestamp. If the file doesn't exist, it creates it. That side effect became its most common use.

---

### Command 3 — cp (Copy)

```bash
cp file1.txt file2.txt                    # copy with new name
cp file1.txt /home/varun/Desktop/         # copy to different folder
cp -r myfolder/ myfolder_backup/          # copy entire folder (MUST use -r)
cp -i file1.txt file2.txt                 # ask before overwriting
```

> ⚠️ If destination file already exists — Linux **silently overwrites**. No warning. No recycle bin. Use `-i` to be safe.

---

### Command 4 — mv (Move / Rename)

Linux has no separate rename command. `mv` does both.

```bash
mv file1.txt /home/varun/Desktop/         # MOVE to different folder
mv file1.txt newname.txt                  # RENAME — same folder, new name
mv file1.txt /Desktop/newname.txt         # MOVE and RENAME at same time
mv -i file1.txt file2.txt                 # ask before overwriting
```

**Real DevOps situation:**
```bash
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup   # backup old config
```
Rename old config as backup before deploying the new one. Done constantly on real servers.

---

### Command 5 — rm (Remove / Delete)

> ⚠️ **MOST DANGEROUS COMMAND IN LINUX** — deletes permanently. No recycle bin. No undo.

```bash
rm myfile.txt               # delete one file
rm file1.txt file2.txt      # delete multiple files
rm -r myfolder/             # delete folder and everything inside
rm -i file.txt              # ask confirmation before deleting
rm -v file.txt              # verbose — tells you what was deleted
```

**Never type this:**
```bash
rm -rf /      # DESTROYS ENTIRE OPERATING SYSTEM — no recovery
```

**Safe habits:** Always use `-i` when not 100% sure. Double check path before pressing Enter.

---

### Interview Questions — Day 2

**Q1. What does `touch` do?**
Updates the last modified timestamp of a file. If the file doesn't exist, it creates an empty file.

**Q2. What is the difference between `cp` and `cp -r`?**
`cp` copies individual files. `cp -r` copies directories recursively — the folder and all its contents including subfolders.

**Q3. How do you delete a folder with all its contents?**
`rm -r foldername` — the `-r` flag makes it recursive.

**Q4. What happens if you `cp` or `mv` to a file that already exists?**
Linux silently overwrites it with no warning. Use `-i` flag to prompt for confirmation.

**Q5. What does `mkdir -p` do?**
Creates the entire directory path including all parent directories that don't exist yet. Without `-p`, mkdir fails if the parent directory is missing.

---

### Homework — Before Day 3

Inside `linux_practice`, create this structure:
```
project/
├── src/
├── logs/
└── backup/
```
1. Create 2 files inside `src/`
2. Copy one file from `src/` to `backup/`
3. Rename the other file in `src/`
4. Delete the original from `src/` after copying
5. Run `ls -ltr project/` to verify everything