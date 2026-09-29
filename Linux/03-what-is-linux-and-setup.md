# Chapter 3: What Is Linux OS? How to Set Up a Linux Server

## 1. What is an operating system (OS)?

An **operating system** is a big program that helps all your applications run on your computer. It sits between the hardware and the applications.

Common operating systems:

- Windows
- macOS
- Linux

Most laptop users use Windows, because laptops are sold with Windows installed by default. But most **servers and applications in companies run on Linux**.

## 2. What is Linux?

- Linux was created in **1991** by **Linus Torvalds**.
- It is **open source**: anyone can read the code, contribute and improve it.
- It is **free** to use.
- It has very strong security.
- It supports multitasking.
- You do not usually need an antivirus.
- It does not force sudden updates in the middle of an important meeting or production issue (a joke about "Windows is updating...").

### Windows vs Linux (short comparison)

| Point | Windows | Linux |
|-------|---------|-------|
| Cost | Paid commercial license | Free and open source |
| Antivirus | Usually needed | Usually not needed |
| Updates | Can restart your work at bad times | You control updates |
| Usage on servers | Less | Very high (about 90% of applications run on it) |
| Security | Good | Very strong |

### Linux distributions (flavors)

Linux comes in many versions called *distributions* or *distros*: **Ubuntu, Kali, CentOS, Red Hat, Fedora, Arch**, and more.

There is also a paid enterprise version, Red Hat Enterprise Linux (RHEL), where you pay for support. Most others are free.

---

## 3. Ways to get a Linux machine

If you are a Windows user, you have several options:

1. **Dual boot**: install Linux next to Windows on the same laptop.
2. **Virtual machine (VirtualBox)**: install Linux inside Windows as a virtual computer.
3. **WSL** (Windows Subsystem for Linux): run Linux inside Windows.
4. **Cloud virtual machine**: create a Linux server on **AWS, Azure or GCP**.
5. **Vagrant**: a tool by HashiCorp that helps you create and run virtual machines easily.

Any one method is enough. The course uses **AWS EC2**.

## 4. Remote access tools (how you reach a far-away server)

- **RDP** (Remote Desktop Protocol): to access a remote Windows desktop.
- **SSH** (Secure Shell): to access a remote Linux server (covered in Chapter 5).
- **AnyDesk** and similar tools.

A DevOps engineer often sits at home and troubleshoots servers far away, so this is important.

---

## 5. The heart of Linux: Kernel, Shell, Bootloader

### Kernel

- Every operating system has a "heart". In Linux it is the **kernel**.
- The kernel contains the programs and processes needed to run the operating system.
- It is written in the **C programming language**.
- Its name is the **Linux kernel** (from its founder Linus).
- It talks to the hardware (disk, RAM, CPU, printer, camera).

### Shell

- We do not know C, and we should not have to. So we need a program that talks to the kernel for us.
- That program is the **shell**. It works as a **gateway/interface between you and the kernel**.
- You type **shell commands** in a **terminal**, and the shell passes them to the kernel.
- Example: you want to create a folder. The kernel already has a C program for it. You tell the shell `mkdir myfolder`, the shell asks the kernel, and the kernel creates the folder.

### Bootloader

- When you press the power button of a computer, electricity goes to the hard disk where the operating system is installed.
- The **bootloader** is the first process that runs. It runs the files that are needed to start the operating system.
- Think of your mother waking everyone in the morning: "Wake up, a new day has begun!" The bootloader wakes up all the other processes.
- A famous Linux bootloader is **GRUB** which stands for **GRand Unified Bootloader**.
- Interview question: "Name a bootloader of Linux." Answer: **GRUB**.

### Linux system architecture (simple flow)

```
Applications / Utilities (terminal, editors)
        |
      Shell
        |
      Kernel
        |
     Hardware (CPU, RAM, disk, printer, camera...)
```

You use an application (like a terminal) -> it talks to the shell -> the shell talks to the kernel -> the kernel controls the hardware.

---

## 6. Everything starts from the root folder `/`

In Linux, everything starts from the **root directory** written as `/` (slash).

Important folders inside `/`:

| Folder | Purpose |
|--------|---------|
| `/home` | Home folders of normal users |
| `/usr` | User programs and utilities |
| `/bin` | Binaries (basic command programs) |
| `/etc` | Configuration files |
| `/var` | Variable data such as logs (`/var/log`) |
| `/tmp` | Temporary files |
| `/root` | Home folder of the root (super) user |

Example: to see logs you can go to `cd /var/log`.

`cd` means **change directory**.

---

## 7. Processes in Linux

- A **process** is a program that is running.
- When you run `top`, you see all processes.
- The very first process started by the system has **PID 1** (Process ID 1). It is started by the boot process.
- Many processes run in the background to keep the system working.

### Process states

| State | Meaning |
|-------|---------|
| Running | The process is working now |
| Sleeping | The process is waiting and not doing work right now |
| Stopped | The process was stopped |
| Terminated / Killed | The process has ended |
| Zombie | The process has finished but is still listed, doing nothing useful |

---

## 8. A few useful hardware/system commands

| Command | Use |
|---------|-----|
| `top` | Show running processes and CPU usage |
| `df -h` | Show disk usage in human-readable form |
| `free -h` | Show RAM (memory) usage |
| `uname` | Show the platform/OS name |

Note: on macOS some commands like `free` are different, but on Linux they work as described.

---

## 9. Hands-on: creating a Linux server on AWS EC2

**EC2** means **Elastic Compute Cloud**. It gives you virtual servers on the cloud. A cloud is basically a data center where you can create a machine remotely.

Steps:

1. Create an **AWS account** (sign up, verify with OTP, add a debit/credit card).
2. Search for **EC2** in the AWS console.
3. Click **Launch instance**.
4. Give a name, for example `linux-for-devops`.
5. Choose the operating system: **Ubuntu** (it is *free-tier eligible*). Do **not** choose Windows because it may cost money.
6. Choose the instance type: **t2.micro** (also free-tier eligible).
7. Create a **key pair** (create new key pair) and download the `.pem` file. Keep it safe.
8. Launch the instance.
9. The instance shows **Pending** first, then **Running**.
10. Click **Connect** to open a terminal. You will see "Welcome to Ubuntu".

Now you have a Linux server (an EC2 instance) to practice on. It is a server, and it has Linux (Ubuntu) installed.

## Quick summary

- Linux is a free, open-source OS created by Linus Torvalds in 1991.
- Kernel = the heart. Shell = the interface to the kernel. Bootloader (GRUB) starts the system.
- Everything in Linux starts from `/`.
- PID 1 is the first process.
- You can practice Linux on AWS EC2 using a free-tier Ubuntu t2.micro machine.
