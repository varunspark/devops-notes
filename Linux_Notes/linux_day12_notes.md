## Day 12 — File Compression & Archives

---

### Archiving vs Compression

| Concept | What it does | Example |
|---------|-------------|---------|
| Archiving | Bundles multiple files into ONE | tar — combines 100 files into 1 |
| Compression | Reduces file size | gzip — makes a file smaller |
| Both together | Bundle AND shrink | tar.gz — combine and compress |

Think of archiving like putting files into a box. Compression is vacuum-sealing that box to make it smaller.

---

### tar — The Most Important Command

**Three commands you will use 90% of the time:**
```bash
tar -czf archive.tar.gz foldername/   # CREATE
tar -xzf archive.tar.gz              # EXTRACT
tar -tzf archive.tar.gz              # LIST contents without extracting
```

**Memory tricks:**
- Create: **c**reate g**z**ip **f**ile = `czf`
- Extract: e**x**tract g**z**ip **f**ile = `xzf`

**All tar flags:**

| Flag | Meaning |
|------|---------|
| c | Create new archive |
| x | Extract files |
| t | List contents |
| f | Specify filename (always needed) |
| v | Verbose — show files being processed |
| z | Compress with gzip (.tar.gz) |
| j | Compress with bzip2 (.tar.bz2) |
| J | Compress with xz (.tar.xz) |
| C | Extract to specific directory |

**More tar examples:**
```bash
tar -czf backup_$(date +%Y%m%d).tar.gz /home/varun/   # date in filename
tar -czvf backup.tar.gz folder/                        # verbose — see progress
tar -czf backup.tar.gz /home/ --exclude="*.log"        # exclude log files
tar -czf backup.tar.gz /home/ --exclude=".git"         # exclude .git
tar -xzf backup.tar.gz -C /tmp/restore/               # extract to specific folder
tar -xzf backup.tar.gz home/varun/file.txt             # extract ONE file only
tar -tzvf backup.tar.gz                                # list with details
```

> ⚠️ **Common tar mistakes:**
> ```bash
> tar -czv backup.tar.gz folder/    # WRONG — forgot -f flag
> tar -czvf backup.tar.gz folder/   # CORRECT
>
> tar -xzf backup.tar.gz /tmp/restore/   # WRONG — /tmp/ interpreted as file
> tar -xzf backup.tar.gz -C /tmp/restore/  # CORRECT — -C specifies destination
> ```

---

### Different Compression Formats

```bash
# .tar.gz — most common, good speed and compression
tar -czf archive.tar.gz folder/   # create
tar -xzf archive.tar.gz           # extract

# .tar.bz2 — better compression, slower speed
tar -cjf archive.tar.bz2 folder/  # create
tar -xjf archive.tar.bz2          # extract

# .tar.xz — best compression, slowest speed (used for software releases)
tar -cJf archive.tar.xz folder/   # create
tar -xJf archive.tar.xz           # extract
```

**Which format to use:**

| Format | Use when |
|--------|---------|
| .tar.gz | Default choice — good balance of speed and size |
| .tar.bz2 | Need smaller file, have time to wait |
| .tar.xz | Maximum compression — software releases |
| .zip | Sharing with Windows users |

---

### gzip — Compress Single Files

```bash
gzip file.txt          # compress — ORIGINAL IS DELETED
gzip -k file.txt       # compress but KEEP original (-k = keep)
gzip -d file.txt.gz    # decompress — same as gunzip
gunzip file.txt.gz     # decompress
gzip -l file.txt.gz    # show compression ratio and sizes
gzip -9 file.txt       # maximum compression (slower)
gzip -1 file.txt       # fastest compression (larger file)
```

> ⚠️ `gzip` deletes the original by default. Use `-k` to keep both original and compressed.

**Read gzipped files without extracting:**
```bash
zcat file.txt.gz       # like cat but for gzip files
zless file.txt.gz      # scroll through without extracting
zgrep "ERROR" file.gz  # search inside without extracting
```

Very useful — you can read compressed log files without extracting them first. Saves time and disk space.

---

### zip — Windows Compatible

```bash
zip -r archive.zip folder/     # create zip
unzip archive.zip              # extract zip
unzip -l archive.zip           # list contents
unzip -d /tmp/ archive.zip     # extract to specific directory
```

---

### Real DevOps Scenarios

**Transfer files between servers:**
```bash
# On source server — pack everything
tar -czf /tmp/transfer.tar.gz /var/www/myapp/

# Transfer to another server
scp /tmp/transfer.tar.gz user@192.168.1.20:/tmp/

# On destination server — extract
tar -xzf /tmp/transfer.tar.gz -C /var/www/
```

**Compress old logs to save disk space:**
```bash
find /var/log/nginx/ -name "*.log" -mtime +7 -exec gzip {} \;

# Or as a cron job (combine with Day 9):
0 3 * * 0 find /var/log/nginx/ -name "*.log" -mtime +7 -exec gzip {} \;
```

---

### Interview Questions — Day 12

**Q1. What is the difference between archiving and compression?**
Archiving combines multiple files into one without reducing size. Compression reduces file size. `tar.gz` does both — tar archives and gzip compresses.

**Q2. What does `tar -czf` mean?**
`c` = create new archive, `z` = compress with gzip, `f` = the next argument is the filename.

**Q3. How do you extract a tar.gz to a specific directory?**
`tar -xzf archive.tar.gz -C /path/to/directory/` — the `-C` flag specifies destination.

**Q4. What is the difference between `.tar.gz` and `.zip`?**
Both archive and compress. `tar.gz` is the Linux standard, preserves file permissions fully and has slightly better compression. `zip` is Windows compatible and more universal for cross-platform sharing. On Linux servers always use `tar.gz`.

**Q5. How do you read a gzipped log file without extracting it?**
Using `zcat file.gz`, `zless file.gz`, or `zgrep "pattern" file.gz` — all work directly on gzip files without extracting.

**Q6. What does `gzip -k` do?**
The `-k` flag keeps the original file after compression. Without `-k`, gzip deletes the original.

**Q7. How do you list the contents of a tar.gz without extracting?**
`tar -tzf archive.tar.gz` — `t` = list contents, `z` = gzip, `f` = filename.

---
