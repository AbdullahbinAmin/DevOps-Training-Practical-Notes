# Chapter 4: Linux Basic Commands for DevOps Engineers

## What is a command?

A computer is smart, but it works only when you give it **instructions**. In Linux, these instructions are called **commands**. You type them in the **shell** (through the terminal). These commands work the same on an EC2 instance, VirtualBox, Vagrant or WSL.

Example: type `date` and press Enter. It shows today's date and time (often in UTC time on cloud servers).

---

## 1. Listing and navigating

### `ls` – list
Shows everything inside the current directory (folder).

```bash
ls
```

### `ls -l` – long list
Shows details: type, permissions, owner, group, size, date and name.

```bash
ls -l
```

In the first column, a `d` at the start means it is a **directory**. For example `drwxr-xr-x` is a directory, `-rw-r--r--` is a normal file.

### `ls -a` and `ls -la`
- `-a` shows **hidden** files (names that start with a dot).
- `-la` combines long list and hidden files.

### `pwd` – print working directory
Shows the folder you are in right now, for example `/home/ubuntu`.

### `clear`
Cleans the terminal screen.

### `cd` – change directory

```bash
cd foldername     # go into a folder
cd ..             # go one step back (to the parent folder)
cd /              # go to the root folder
cd /bin           # go to an absolute path
```

Use `pwd` after each `cd` to see where you are. Tip: `..` means "the folder above".

---

## 2. Creating and deleting

### `mkdir` – make directory
```bash
mkdir devops
```
Creates a folder named `devops`.

### `touch` – create an empty file
```bash
touch newfile.txt
```

### `rm` – remove a file
```bash
rm newfile.txt
```
Warning: Linux has **no recycle bin**. A removed file is gone.

### Removing a folder
`rm` alone cannot remove a folder. You get an error "is a directory". Use a **flag** (an option that starts with a dash):

```bash
rm -r foldername      # -r = recursive, deletes the folder and everything inside
rmdir foldername      # removes an empty directory
```

---

## 3. Viewing and writing file content

### `cat` – show file content
```bash
cat demofile.txt
```
(Not a cat, it is short for "concatenate". It prints what is inside a file.)

### `echo` – print text
```bash
echo "Hello friends"
```

### Redirection `>` and `>>`
`>` sends the output of a command into a file instead of the screen.

```bash
echo "Hello friends" > demofile.txt     # writes text to file (replaces old content)
echo "Second line" >> demofile.txt      # appends (adds to the end)
```

If the file does not exist, `>` creates it. So you do not always need `touch`.

### `zcat`
Shows the content of a compressed (gzip) file without unzipping it.

### `head` and `tail`
- `head myfile` shows the **first** lines (default 10).
- `tail myfile` shows the **last** lines (default 10).
- `head -n 5 myfile` shows the first 5 lines.
- `tail -n 5 myfile` shows the last 5 lines.

### `tail -f` (very important for DevOps)
```bash
tail -f logfile.log
```
It **keeps watching** the file and prints new lines as they are added. Useful when you check live application logs. Press **Ctrl + C** to stop.

### `less` and `more`
For very large files (like 200 pages), these show the content **page by page**.

```bash
less bigfile.txt
more bigfile.txt
```

---

## 4. Copy, move and rename

### `cp` – copy
```bash
cp source destination
cp new.txt devops/          # copy a file into a folder
cp -r cloud devops/         # copy a whole folder (-r = recursive)
```
After copy, the file exists in **both** places.

You can copy from outside the folder by giving the path:
```bash
cp devops/file.txt cloud/
```

### `mv` – move (and rename)
```bash
mv new.txt cloud/           # move a file into a folder (it disappears from the old place)
mv devops linux-for-devops  # rename devops to linux-for-devops
```
Rename in Linux is just "move to a new name".

**Copy vs move:** in copy, the file stays in the source and also appears in the destination. In move, the file leaves the source.

---

## 5. `wc` – word count

```bash
wc myfile.txt
```
Shows three numbers: **number of lines, number of words, number of bytes**. It can also work with many files at once.

`ls -l` also shows the size of a file in bytes. A small file like "this is my file" is 16 bytes.

---

## 6. Links: hard link vs soft link (a top interview question)

A **link** is like a **shortcut**. On Windows you put a shortcut of a game on your desktop instead of opening the folder every time.

### Command

```bash
ln  original_file  hard_link_name       # hard link
ln -s original_file soft_link_name      # soft (symbolic) link
```
Tip: use full paths when the link is in a different folder.

### Difference

| Point | Soft link (`ln -s`) | Hard link (`ln`) |
|-------|---------------------|------------------|
| If the original file is deleted | The link **breaks** (`ls` shows it in red) | The link **still works** and keeps the content |
| Changes to the original | Show through the link | Show through the link (same data) |
| Shown in `ls -l` | With an arrow `->` pointing to the original | As a normal file |

### Practical demonstration from the course

1. Create a file, write "Hello friends, this is soft link" in it.
2. Create a soft link. `cat softlink` shows the same text.
3. Change the original file. The soft link shows the new text too.
4. Delete the original. The soft link turns red and stops working.
5. Recreate the original and make a **hard link**. Then delete the original: `cat hardlink` still shows the content.

---

## 7. `cut` – take a small part of a line/file

```bash
cut -b 1-4 myfile.txt
```
`-b` means bytes. This prints bytes 1 to 4. If the file has "This is my file", the result is "This".

---

## 8. `tee` – show output and save it to a file too

```bash
echo "Hello" | tee hello.txt
```
The `|` (pipe) sends output from one command to another. `tee` prints the text on the screen **and** saves it in `hello.txt`. (Think "T" junction, not the drink.)

---

## 9. `sort` – sort lines alphabetically

If a file has lines `z, b, a, e, i`, then `sort file` prints them as `a, b, e, i, z`.

---

## 10. `diff` – difference between two files

```bash
diff file1 file2
```
If both files are exactly the same, there is no output. If they differ, it shows the different lines. `diff` compares **two** files at a time, while `wc` can handle many files.

---

## 11. The `vi` / `vim` editor

`vi` is a text editor that runs inside the terminal (like Notepad but in the shell). `vim` is an improved version.

```bash
vi demofile.txt
```

Important keys:

| Key | What it does |
|-----|--------------|
| `i` | Go into **insert mode** so you can type |
| `Esc` | Leave insert mode |
| `:wq` then Enter | **Write** (save) and **quit** |
| `:q!` | Quit without saving |

After saving, use `cat demofile.txt` to see the content.

---

## Command cheat sheet

| Command | Meaning |
|---------|---------|
| `date` | Show date/time |
| `ls`, `ls -l`, `ls -a` | List files |
| `pwd` | Current folder |
| `cd` | Change folder |
| `mkdir` | Make folder |
| `touch` | Create empty file |
| `rm`, `rm -r`, `rmdir` | Remove file/folder |
| `cat` | Show file content |
| `echo` | Print text |
| `>` / `>>` | Write / append output to file |
| `head`, `tail`, `tail -f` | See beginning / end / live end of file |
| `less`, `more` | Read big files page by page |
| `cp`, `cp -r` | Copy |
| `mv` | Move or rename |
| `wc` | Count lines, words, bytes |
| `ln`, `ln -s` | Hard link, soft link |
| `cut` | Cut part of a file |
| `tee` | Save and show output |
| `sort` | Sort lines |
| `diff` | Compare two files |
| `vi` | Text editor |
