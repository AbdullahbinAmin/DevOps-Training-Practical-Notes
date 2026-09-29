# Chapter 10: Pro Linux Commands – AWK, SED, GREP

To become a "pro" you should know a little programming. In Linux, you get this through **shell scripting**. But often you do not need a whole script. You can do a lot of work directly from the command line with three powerful tools: **awk**, **sed** and **grep**. They are mainly used to work with **log files**.

(The video title also lists `find`. In the video only awk, sed and grep were demonstrated, so a short note on `find` is added at the end.)

## The practice setup

1. Create a folder and enter it: `mkdir awk-sed && cd awk-sed`
2. Get a sample **log file** from the internet (a small piece of an application log).
3. Save it as `application.log` using `vi application.log` and paste the content.

Each line in the log has columns such as: date, time, a level word (TRACE, INFO, EVENT), and a message. The log level tells you the type of message.

Sample idea of a log line:
```
2023-12-15 08:53:22 INFO Some message here
```

---

## 1. `head` – the simple way (quick recap)

To see line 2 or the first few lines you can do:
```bash
head -n 2 application.log
```
But when you want only some **columns** (like date and time) you need awk.

---

## 2. AWK

`awk` is a small programming language inside a command. It works on **columns** (fields) of each line. It works best on **structured data** such as CSV (comma separated values) or TSV (tab separated values). Words separated by spaces are treated as columns by default.

### Basic syntax

```bash
awk '{ commands }' filename
```
- Single quotes wrap the whole program.
- The code lives inside curly braces `{ }`.

### Print the whole file
```bash
awk '{print}' application.log
```

### Print columns
`$1` is the first column, `$2` the second, and so on. `$0` is the whole line.

```bash
awk '{print $1}' application.log          # first column
awk '{print $1, $2}' application.log      # first and second column
awk '{print $1, $2, $4}' application.log  # skip the third
awk '{print $1, $2, $3, $5}' application.log
```
This is how you can pull out only the **date and time** from each line.

### Filter lines with a pattern

Only lines that contain `INFO`:
```bash
awk '/INFO/ {print $1, $2, $3, $4, $5}' application.log
```
Save the result to a new file:
```bash
awk '/INFO/ {print $1, $2, $3}' application.log > only-info.log
cat only-info.log
```

Show only `EVENT` lines:
```bash
awk '/EVENT/ {print}' application.log
```

### Count how many times a word appears

```bash
awk '/INFO/ {count++} END {print count}' application.log
```

How it works:
- `/INFO/ {count++}`: every time a line contains INFO, increase `count` by 1.
- `END { ... }`: this block runs after the last line.
- `print count`: prints the final number.

You can print a message too:
```bash
awk '/INFO/ {count++} END {print "The count of INFO is", count}' application.log
```
Similarly you can count a particular IP address by putting that IP in the pattern.

### Filter by a time range

Suppose all logs are from 08:53 to 08:54 and you only want those in 08:53. If column 2 is the time:

```bash
awk '$2 >= "08:53:00" && $2 <= "08:53:59" {print $1, $2, $3, $4}' application.log
```
- `&&` means AND.
- Both conditions must be true. Lines from 08:54 are not printed.

### Filter by line number

`NR` is the built-in variable for **Number of Row** (current line number).

```bash
awk 'NR >= 2 && NR <= 10 {print}' application.log
awk 'NR >= 2 && NR <= 10 {print NR, $0}' application.log    # also show line numbers
```
This prints lines 2 to 10.

### Summary about awk
awk gives you a mini programming language: print columns, conditions, counters, loops. It needs **formatted (structured) data**, so it is used on CSV/TSV-like files.

---

## 3. SED (Stream Editor)

`sed` is like awk, but it works **line by line** on any text: structured, semi-structured or unstructured. The name means **s**tream **ed**itor: it edits the data stream (from a file or from the output of another command) as it flows.

### Print matching lines

```bash
sed -n '/INFO/p' application.log
```
- `-n` means "do not print everything by default".
- `/INFO/` is the pattern.
- `p` means print. So only lines that match INFO are printed.

(Without `-n`, every line would print, and matching lines would print twice.)

### Substitute (find and replace)

```bash
sed 's/INFO/LOG/g' application.log
```
- `s` = substitute.
- `INFO` = word to find, `LOG` = replacement.
- `g` = global (replace all matches in each line, not just the first).

This shows the output on the screen; the original file is unchanged. To change the file itself you can use `-i` (in-place):
```bash
sed -i 's/INFO/LOG/g' application.log
```
Real use: if sensitive data appeared in a log by mistake, quickly replace it before sharing. Be careful, since `-i` changes the original file.

### Show the line numbers of a match

```bash
sed -n '/INFO/=' application.log
```
`=` prints the line number.

To print the line numbers **and** the lines, run two expressions with `-e`:
```bash
sed -n -e '/INFO/=' -e '/INFO/p' application.log
```
Rule: when you use expressions you need `-e`, and `-n` limits the output.

### Replace only in a line range

Replace INFO with LOG only in lines 1 to 10:

```bash
sed '1,10s/INFO/LOG/g' application.log
```
- `1,10` = from line 1 to line 10.
- `s/INFO/LOG/g` = the replacement.

If you still see INFO after line 10 that is normal because the range stopped at 10. Increase the range (for example `1,15`) to change more lines.

### Print only a small part and quit

```bash
sed '1,15s/INFO/LOG/g;15q' application.log
```
- The `;` separates two commands.
- `15q` means **quit** after line 15, so you only get the first 15 lines.

---

## AWK vs SED (interview question)

| Point | awk | sed |
|-------|-----|-----|
| Works on | Records/columns | Lines |
| Data type | Best for **structured** data (CSV, TSV) | Structured, semi-structured or unstructured |
| Main strength | Column extraction, calculations, counters | Search, replace, print or delete lines |
| Syntax | `awk '{print $1}' file` | `sed 's/old/new/g' file` |
| Feel | Small programming language | Stream editor |

---

## 4. GREP (Global Regular Expression Print)

`grep` searches a file (or a whole system) for a **pattern** and prints the matching lines.

```bash
grep INFO application.log
```

### Case-insensitive search
```bash
grep -i info application.log
```
`-i` ignores upper/lower case. So `info`, `INFO` and `Info` all match.

### Count matches
```bash
grep -ic info application.log
```
Or `grep -c INFO application.log`. `-c` prints the **number of matching lines** (for example 16).

To get this same result with awk you would need to write `count++`, `END`, `print count` – much longer. That is why grep is loved for quick searches.

### Using grep with a pipe

You can filter the output of another command:

```bash
ps -ax | grep ubuntu       # only processes run by ubuntu
```
The `|` (pipe) sends the output of `ps -ax` into `grep`, which keeps only lines with "ubuntu".

You can also combine awk with pipes:
```bash
ps -ax | awk '{print $2}'   # print only the second column
```

---

## When to use which?

| Need | Tool |
|------|------|
| Simply find lines with a word | `grep` |
| Count occurrences of a word | `grep -c` |
| Take specific columns, do conditions/counters | `awk` |
| Replace text, print certain lines, edit a stream | `sed` |

---

## Extra note: `find` (not covered in the video)

`find` is used to **search for files and folders by name, type, size or time**:

```bash
find /home -name "application.log"     # find by name
find . -type f -name "*.log"           # all .log files in the current folder
find /var/log -size +10M               # files bigger than 10 MB
```

## Quick summary

- **grep**: search for text patterns; `-i` ignore case, `-c` count.
- **awk**: column-based, mini programming (`$1`, `NR`, `END`, conditions).
- **sed**: line-based stream editor (`-n`, `p`, `s/old/new/g`, `=`, `q`).
- These three help a lot when you handle big log files.
