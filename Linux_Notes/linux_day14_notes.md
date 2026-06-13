## Day 14 — Vim Text Editor

---

### Why Vim

Every server has Vim pre-installed. When you SSH into a server at 3am to fix an issue, Vim is what is available. Nano is simpler but Vim is faster once learned. Also — "How do you exit Vim?" is a classic interview question.

---

### What You Will Learn Today

- Understanding Vim modes — the key concept
- Opening and closing files
- Navigation without arrow keys
- Inserting and editing text
- Saving files
- Searching and replacing
- Copy, cut and paste
- Vim in real DevOps config editing

---

### The Single Most Important Concept — Vim Modes

Vim has modes. When you open it you **cannot just start typing.**

| Mode | What it is | How to enter |
|------|-----------|-------------|
| Normal mode | Default — for navigation and commands | Press `Esc` |
| Insert mode | For typing and editing text | Press `i`, `a`, or `o` |
| Command mode | For saving, quitting, searching | Press `:` from Normal |
| Visual mode | For selecting text | Press `v` |

**The most common beginner mistake:** You open Vim and start typing. Text appears in random places. You panic. **Why:** You are in Normal mode. Vim thinks your keystrokes are commands.

**Golden rule:** When confused — press `Esc` multiple times to get back to Normal mode. From Normal mode you can always recover.

---

### Opening and Closing Vim

```bash
vim filename.txt          # open existing file or create new
vim /etc/nginx/nginx.conf # open a system config file
sudo vim /etc/hosts       # open system file with root permissions
```

**Closing Vim — the classic interview question:**
```bash
:q          # quit — only works if no changes were made
:q!         # quit WITHOUT saving — force quit, discard all changes
:w          # save but stay in vim
:wq         # save AND quit
:wq!        # save and quit forcefully
:x          # same as :wq — save and quit
ZZ          # save and quit (no colon needed, in Normal mode)
ZQ          # quit without saving (no colon needed)
```

---

### Navigation in Normal Mode

```bash
h j k l         # left down up right (arrow keys also work)
w / b           # jump to next / previous word
e               # jump to END of current word
0 / $           # jump to start / end of line
^               # jump to first non-blank character of line
gg / G          # go to FIRST / LAST line of file
50G             # go to line 50
:50             # go to line 50 (command mode)
Ctrl+f / b      # page forward / backward
Ctrl+d / u      # half page down / up
```

**Memory trick for hjkl:** h is leftmost, l is rightmost, j has a tail going down, k goes up.

**Real DevOps navigation:** Open huge config file and go straight to what you need:
```bash
G          # jump to bottom
gg         # jump to top
/searchterm  # search for what you need
```

---

### Entering Insert Mode

```bash
i    # insert BEFORE cursor
a    # insert AFTER cursor (append)
I    # insert at BEGINNING of line
A    # insert at END of line (very common for editing config lines)
o    # open new line BELOW and enter insert mode
O    # open new line ABOVE and enter insert mode
```

**Workflow for editing a config file:**
1. Navigate to the line (hjkl or search with `/`)
2. Press `i` to enter Insert mode
3. Make your changes
4. Press `Esc` to return to Normal mode
5. Type `:wq` to save and quit

---

### Editing — Delete, Undo, Redo

```bash
x         # delete ONE character under cursor
dd        # delete ENTIRE current line
5dd       # delete 5 lines from cursor
dw        # delete from cursor to end of word
d$        # delete from cursor to end of line
d0        # delete from cursor to beginning of line
dG        # delete from current line to end of file
u         # undo last change
Ctrl+r    # redo — undo the undo
5u        # undo last 5 changes
r         # replace ONE character (stay in normal mode)
cw        # change word (delete word + enter insert mode)
cc        # change entire line
```

---

### Copy, Cut, Paste

In Vim: Copy = Yank, Cut = Delete (deleted text goes to clipboard), Paste = Put

```bash
yy        # copy (yank) current line
5yy       # copy 5 lines
yw        # yank from cursor to end of word
y$        # yank from cursor to end of line
p         # paste AFTER cursor / BELOW current line
P         # paste BEFORE cursor / ABOVE current line
dd        # cut current line
5dd       # cut 5 lines
```

**Practical copy-paste:** Duplicate a config block in nginx:
1. Navigate to first line of block
2. Count lines — say it is 5 lines
3. Press `5yy` — copies 5 lines
4. Navigate to where you want to paste
5. Press `p` — pastes below current line
6. Edit the pasted block as needed

---

### Searching and Replacing

```bash
/searchterm          # search FORWARD — press Enter
?searchterm          # search BACKWARD — press Enter
n                    # jump to NEXT match
N                    # jump to PREVIOUS match

# Replace in current line:
:s/old/new/          # replace FIRST occurrence on current line
:s/old/new/g         # replace ALL on current line

# Replace in entire file:
:%s/old/new/         # replace first occurrence on each line
:%s/old/new/g        # replace ALL in entire file
:%s/old/new/gc       # replace all but ASK confirmation each time
:%s/old/new/gi       # replace all, case insensitive

# Replace in a range of lines:
:5,20s/old/new/g     # replace between lines 5 and 20
```

---

### Full Vim Quick Reference

```bash
# Modes
Esc           # Normal mode — press when confused
i             # Insert mode before cursor
a             # Insert mode after cursor
o             # Insert mode, new line below
:             # Command mode

# Save/Quit
:w            # save
:q!           # quit without saving
:wq or :x     # save and quit
ZZ            # save and quit (no colon)

# Navigation
h j k l       # left down up right
gg / G        # top / bottom of file
0 / $         # line start / end
:50           # go to line 50

# Edit
dd            # delete line
yy            # copy line
p / P         # paste below / above
u / Ctrl+r    # undo / redo
r             # replace one character
cw            # change word
.             # repeat last change

# Search/Replace
/term         # search forward
n / N         # next / previous
:%s/old/new/g # replace all in file
:%s/old/new/gc # replace all with confirmation
```

---

### Real Scenario

Your manager says: "Change nginx port from 8080 to 9090 everywhere in the config."

```bash
sudo vim /etc/nginx/nginx.conf
# Inside Vim:
/8080                       # verify it exists — press Enter
n                           # check all occurrences
:%s/8080/9090/gc            # replace all with confirmation
:wq                         # save and quit
sudo systemctl reload nginx  # apply changes — no downtime
```

---

### Interview Questions — Day 14

**Q1. How do you exit Vim without saving?**
Press `Esc` to ensure Normal mode, then `:q!` and Enter.

**Q2. What are the main modes in Vim?**
Normal (navigation/commands), Insert (typing), Command (save/quit), Visual (selecting text).

**Q3. How do you search and replace all occurrences in a file?**
`:%s/oldword/newword/g` — `%` = entire file, `s` = substitute, `g` = global (all occurrences).

**Q4. How do you go to a specific line number?**
Type `:50` in command mode, or `50G` in Normal mode.

**Q5. What is the difference between `dd` and `yy`?**
`dd` deletes the current line and puts it in the clipboard (cut). `yy` copies the current line without deleting it. Both can be pasted with `p`.

**Q6. How do you save a file you opened without sudo but need root to save?**
`:w !sudo tee %` — writes the file using sudo through tee. Common trick when you forget to open with sudo.

**Q7. What does the `.` command do in Vim?**
Repeats the last change. Extremely useful for applying the same edit to multiple locations — navigate and press `.` each time.

**Q8. How do you comment out multiple lines at once?**
Press `Ctrl+v`, select lines with `j`, press `I` (capital i), type `#`, press `Esc`. The `#` appears on all selected lines simultaneously.

---

### Homework — Before Day 15

1. Open `vim day14_practice.txt` in `~/linux_practice`
2. Enter Insert mode and type 10 lines of any text
3. Press Esc and navigate using hjkl only — no arrow keys
4. Go to line 5 using `:5`
5. Delete line 3 with `dd`
6. Copy line 2 with `yy` and paste below line 7 with `p`
7. Search for a word using `/word`
8. Replace all occurrences of a word using `:%s/old/new/g`
9. Save and quit with `:wq`
10. Open again and quit WITHOUT saving using `:q!`

---
