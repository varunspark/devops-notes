## Day 4 — File Permissions, Users and Groups

---

### Why This is Critical for DevOps

Most common interview topic for Linux in DevOps roles. Almost every interview has 2–3 questions on this. Permissions answer three questions for every file:
- What can YOU (the owner) do?
- What can your GROUP do?
- What can EVERYONE ELSE do?

---

### What You Will Learn Today

- Reading the permission string (rwxr-xr-x)
- Users and Groups
- chmod — changing permissions (numbers and letters)
- chown — changing ownership
- sudo — running as superuser

---

### Reading the Permission String

```
d  rwx  r-x  r-x
|   |    |    |
|   |    |    └── Others (everyone else on the system)
|   |    └─────── Group (members of the file's group)
|   └──────────── Owner (the person who owns the file)
└──────────────── Type: d=directory, -=file, l=symlink
```

| Letter | On a FILE means | On a DIRECTORY means |
|--------|----------------|---------------------|
| r (read) | Can open and read | Can run ls to see contents |
| w (write) | Can modify | Can create/delete files inside |
| x (execute) | Can run as a program | Can cd into it |
| - (none) | Permission not given | Permission not given |

> ⚠️ **Most confusing part:** Execute on a directory means permission to **enter** it with `cd`. Without `x` on a directory you cannot `cd` into it even if you can see it with `ls`.

---

### The Octal (Number) Permission System

Each permission has a number:

| Permission | Value |
|-----------|-------|
| r (read) | 4 |
| w (write) | 2 |
| x (execute) | 1 |
| - (none) | 0 |

Add them up for each group (owner, group, others):

| Combo | Calculation | Number |
|-------|------------|--------|
| rwx | 4+2+1 | 7 |
| rw- | 4+2+0 | 6 |
| r-x | 4+0+1 | 5 |
| r-- | 4+0+0 | 4 |
| --- | 0+0+0 | 0 |

**Common permission numbers:**

| Number | Permissions | Use case |
|--------|------------|---------|
| 755 | rwxr-xr-x | Scripts, directories |
| 644 | rw-r--r-- | Config files — others can read only |
| 600 | rw------- | SSH keys, secrets |
| 400 | r-------- | AWS .pem key files |
| 777 | rwxrwxrwx | Everyone full access — NEVER use in production |

---

### Command 1 — chmod (Change Mode)

```bash
chmod 755 script.sh              # set using numbers
chmod 644 config.yml             # config file — others read only
chmod 600 ~/.ssh/id_rsa          # private key — owner only
chmod u+x script.sh              # add execute for owner only
chmod g-w file.txt               # remove write from group
chmod o-r secret.txt             # remove read from others
chmod a+r file.txt               # add read for ALL (owner+group+others)
chmod -R 755 myfolder/           # change permissions recursively
```

**Symbolic method:**
```
u = user (owner)    + = add permission
g = group           - = remove permission
o = others          = = set exact permission
a = all (u+g+o)
```

---

### Command 2 — chown (Change Owner)

```bash
chown varun file.txt              # change owner to varun
chown varun:developers file.txt   # change owner AND group
chown -R varun myfolder/          # change owner recursively
```

> ⚠️ `chown` requires `sudo` — only root can change ownership.

---

### sudo — Superuser Do

```bash
sudo command                      # run command as root
sudo systemctl restart nginx      # restart nginx as root
whoami                            # show current username
id                                # show user ID and group memberships
sudo -i                           # start a root shell (be careful)
```

---

### Interview Questions — Day 4

**Q1. What does `chmod 755` mean?**
Sets permissions to `rwxr-xr-x` — owner has full read, write, execute. Group and others have read and execute only. Standard for scripts and directories.

**Q2. What is the difference between `chmod` and `chown`?**
`chmod` changes the permissions of a file. `chown` changes who owns the file.

**Q3. What does `sudo` do?**
Allows a permitted user to run a command with root privileges temporarily. Requires the user's own password.

**Q4. What permissions would you set on an SSH private key and why?**
`chmod 400` — read only for owner, no access for anyone else. SSH refuses to use the key if permissions are too open.

**Q5. What is execute permission on a directory?**
The ability to enter it using `cd`. Without execute permission you cannot navigate into the directory even if you can see it with `ls`.

**Q6. A file has permissions `rw-r--r--`. What is its octal value?**
644 — owner is rw (4+2=6), group is r (4), others is r (4).

---

### Homework — Before Day 5

1. Create `secret.txt` and set permissions so **only you can read it**
2. Create `shared/` and set permissions so **everyone can read and enter but only you can write inside**
3. Run `ls -l` and verify the permissions look correct
4. Run `id` and write down what groups you belong to
5. Decode this permission string: `rwxr-x---` — what is the octal number?