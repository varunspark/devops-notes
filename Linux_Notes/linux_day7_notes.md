## Day 7 — Text Processing and Piping

---

### Why This is Critical for DevOps

You have a log file with 1 million lines and need to extract only the IP addresses. You have a CSV with 10,000 users and need to find duplicates. You need to replace a config value in 50 files at once. You cannot do this manually. These commands automate it in seconds.

**Interviews at 10–12 LPA often give you a log file and say "extract this data using the command line."** Today prepares you exactly for that.

---

### What You Will Learn Today

- The `|` pipe — connecting commands together
- wc — count lines, words, characters
- cut — extract specific columns
- sort — sort lines
- uniq — remove duplicates
- awk — column-based text processing
- sed — find and replace in files
- Chaining multiple commands for real tasks

---

### Setup — Create Practice Files

```bash
cd ~/linux_practice && mkdir day7 && cd day7

cat > access.log << EOF
192.168.1.10 GET /home 200 0.234
192.168.1.20 GET /about 200 0.123
192.168.1.10 POST /login 401 0.456
192.168.1.30 GET /home 200 0.321
192.168.1.20 GET /products 200 0.234
192.168.1.10 GET /home 200 0.111
192.168.1.40 GET /home 404 0.543
192.168.1.30 POST /login 200 0.678
192.168.1.20 GET /home 500 0.890
192.168.1.10 GET /products 200 0.234
192.168.1.50 GET /about 200 0.123
192.168.1.40 POST /login 200 0.456
EOF

cat > users.csv << EOF
name,department,salary
Varun,DevOps,85000
Priya,Development,90000
Ravi,DevOps,82000
Sneha,Testing,75000
Amit,Development,88000
Priya,Development,90000
Ravi,DevOps,82000
Kiran,Testing,76000
EOF
```

---

### The Pipe | — The Most Powerful Concept

A pipe `|` takes the **output of one command** and feeds it as **input to the next command.**

```bash
command1 | command2 | command3
```

Without pipe — you'd save output to a file, then read it, then process it. With pipe — it all happens in one line.

Think of it like a factory assembly line — each machine takes what the previous one produced.

```bash
cat access.log | grep "200"     # reads file → filters 200 lines
# Same as:
grep "200" access.log
```

Piping becomes powerful when you chain 3, 4, 5 commands together.

---

### Command 1 — wc (Word Count)

```bash
wc access.log           # lines, words, characters
wc -l access.log        # count lines only — most used
wc -w access.log        # count words only
wc -c access.log        # count characters/bytes only
```

**Real DevOps use with pipe:**
```bash
grep "ERROR" app.log | wc -l    # how many error lines?
ps aux | wc -l                   # how many processes are running?
```

---

### Command 2 — cut (Extract Columns)

```bash
cut -d',' -f1 users.csv         # first column of CSV
cut -d',' -f2 users.csv         # second column
cut -d',' -f1,3 users.csv       # first and third column
cut -d' ' -f1 access.log        # first column of space-separated log (IP)
```

| Flag | Meaning |
|------|---------|
| `-d','` | delimiter is comma |
| `-d' '` | delimiter is space |
| `-f1` | take field number 1 (first column) |
| `-f1,3` | take fields 1 and 3 |

**Extract all IP addresses from log:**
```bash
cut -d' ' -f1 access.log
# Output:
192.168.1.10
192.168.1.20
192.168.1.10
...
```

---

### Command 3 — sort

```bash
sort users.csv              # alphabetical sort
sort -r users.csv           # reverse alphabetical
sort -n numbers.txt         # numerical sort (important for numbers)
sort -rn numbers.txt        # reverse numerical — largest first
sort -k2 users.csv          # sort by column 2
sort -t',' -k3 -n users.csv # sort CSV by column 3 numerically
sort -u users.csv           # sort and remove duplicates
```

**Real DevOps use:** Sort IP addresses from log so same IPs are grouped together:
```bash
cut -d' ' -f1 access.log | sort
```

---

### Command 4 — uniq (Remove Duplicates)

> ⚠️ `uniq` only removes **consecutive** duplicates. **Always use after `sort`.**

```bash
sort users.csv | uniq           # unique list
uniq -c file.txt                # count how many times each line appears
uniq -d file.txt                # show only duplicate lines
uniq -u file.txt                # show only NON-duplicate lines
```

**Classic combo — find unique IPs:**
```bash
cut -d' ' -f1 access.log | sort | uniq
```

**Count how many requests each IP made:**
```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn
# Output:
      4 192.168.1.10
      3 192.168.1.20
      2 192.168.1.30
      2 192.168.1.40
      1 192.168.1.50
```

---

### Command 5 — awk (Column Processing)

awk processes text **column by column** — like a mini programming language built into Linux.

```
$1 = first column    $NF = last column (NF = Number of Fields)
$2 = second column   $0  = entire line
```

```bash
awk '{print $1}' access.log           # print first column (IP addresses)
awk '{print $1, $3}' access.log       # print columns 1 and 3
awk '{print $NF}' access.log          # print last column (response time)
awk '$4 == "200"' access.log          # show lines where column 4 = 200
awk '$4 == "404"' access.log          # show only 404 errors
awk '$4 >= "500"' access.log          # show 5xx server errors
awk -F',' '{print $1}' users.csv      # use comma as delimiter
awk -F',' '{print $1, $3}' users.csv  # print name and salary
```

**awk doing calculations:**
```bash
awk -F',' 'NR>1 {sum += $3} END {print "Total salary:", sum}' users.csv
# NR>1 = skip the first row (header)
# sum += $3 = add column 3 (salary) to running total
# END = do this after processing all lines
# Output: Total salary: 668000
```

**awk vs cut — when to use which:**

| Situation | Use |
|-----------|-----|
| Simple column extraction | `cut` — faster to type |
| Need conditions or math | `awk` — more powerful |
| Multiple columns with processing | `awk` |
| Just splitting by delimiter | `cut` |

---

### Command 6 — sed (Stream Editor)

Finds and replaces text in files or streams. Like Ctrl+H in a text editor but from the command line and much more powerful.

```bash
sed 's/old/new/' file.txt           # replace FIRST occurrence per line
sed 's/old/new/g' file.txt          # replace ALL (global)
sed 's/old/new/gi' file.txt         # replace all, case insensitive
sed 's/GET/HTTP-GET/g' access.log   # replace all GET with HTTP-GET
```

**sed does NOT change the file by default** — it prints modified output to screen. To actually modify the file:
```bash
sed -i 's/old/new/g' file.txt        # edit file IN PLACE (modifies file directly)
sed -i.bak 's/old/new/g' file.txt    # creates file.bak backup FIRST then modifies
```

> ⚠️ **Always use `-i.bak` on important files** — it creates a backup before changing. Never run `sed -i` on a production config without a backup.

**Deleting lines:**
```bash
sed '/ERROR/d' app.log              # delete all lines containing ERROR
sed '1d' users.csv                  # delete line 1 (header row)
sed '1,5d' file.txt                 # delete lines 1 through 5
```

**Real DevOps situation:** Update database host in config across servers:
```bash
sed -i.bak 's/db-old.company.com/db-new.company.com/g' /etc/app/config.yml
```
One command updates the config. The `.bak` backup means you can restore if something goes wrong.

---

### Chaining Everything Together — Real Pipeline Challenges

**Challenge 1 — Find unique IPs that got 404 errors:**
```bash
grep "404" access.log | cut -d' ' -f1 | sort | uniq
```
Step by step:
1. `grep "404"` — filter only 404 lines
2. `cut -d' ' -f1` — extract IP column
3. `sort` — group same IPs together
4. `uniq` — remove duplicates

**Challenge 2 — Count requests per IP, highest first:**
```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn
```

**Challenge 3 — Top 5 IPs with most requests:**
```bash
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -5
```

**Challenge 4 — Count total errors vs successes:**
```bash
grep "200" access.log | wc -l              # count successes
grep -E " [45][0-9][0-9] " access.log | wc -l  # count errors
```

**Challenge 5 — Replace all 500 errors with CRITICAL in a new file:**
```bash
sed 's/ 500 / CRITICAL /g' access.log > access_flagged.log
```

---

### Full Command Summary — Day 7

| Command | What it does | Key flag |
|---------|-------------|---------|
| `wc -l file` | Count lines | `-w` words, `-c` chars |
| `cut -d' ' -f1` | Extract column 1 | `-d` delimiter, `-f` field |
| `sort file` | Sort alphabetically | `-n` numeric, `-r` reverse, `-k` by column |
| `uniq file` | Remove consecutive duplicates | `-c` count, `-d` show duplicates |
| `awk '{print $1}'` | Print column 1 | `-F` delimiter, conditions |
| `sed 's/old/new/g'` | Find and replace | `-i` in-place, `-i.bak` with backup |
| `cmd1 | cmd2` | Pipe output to next command | Chain as many as needed |

---

### Interview Questions — Day 7

**Q1. How do you count the number of lines in a file?**
`wc -l filename`

**Q2. How do you extract the third column from a space-separated file?**
`cut -d' ' -f3 filename` or `awk '{print $3}' filename`

**Q3. Why must you sort before using uniq?**
Because `uniq` only removes **consecutive** duplicate lines. If duplicates are not adjacent it won't detect them. Sorting first brings all identical lines together.

**Q4. How do you find the top 3 most frequent values in a column?**
`cut -d' ' -f1 file | sort | uniq -c | sort -rn | head -3`

**Q5. How do you replace text in a file without opening an editor and create a backup?**
`sed -i.bak 's/oldtext/newtext/g' filename` — modifies in place and creates `.bak` backup automatically.

**Q6. What is the difference between `awk` and `cut`?**
`cut` is simpler — extracts columns by a fixed delimiter. `awk` is more powerful — can apply conditions, do calculations, handle variable whitespace, and process data like a programming language.

**Q7. How do you count how many times each unique value appears in a column?**
`cut -d' ' -f1 file | sort | uniq -c | sort -rn` — extracts the column, sorts it, counts unique values, and sorts by frequency.

---

### Homework — Before Day 8

Using `access.log` from today's practice:
1. How many total requests were made? (`wc -l`)
2. How many requests returned status 200? (`grep | wc -l`)
3. Which IP made the most requests? (`cut | sort | uniq -c | sort -rn | head -1`)
4. How many unique IPs visited? (`cut | sort | uniq | wc -l`)
5. What is the most requested URL? (column 3 — `cut | sort | uniq -c | sort -rn`)

---
