---
layout: single
title: "WSL Cheatsheet"
date: 2025-10-15 10:30:00 +0530
categories: [notes]
tags: [wsl, linux, windows, bash, powershell, dev-setup, jekyll]
excerpt: "A practical, extensible guide to WSL: fundamentals, Bash vs PowerShell, filesystems, networking, performance, daily recipes, and troubleshooting."
classes: single
---

# WSL Cheatsheet — From First Principles to Power User

* TOC
{:toc}

---

## 1. First Principles

### 1.1 OS, Kernel, Userspace
- **Kernel**: hardware, processes, memory, filesystems, networking.
- **Userspace**: shells (Bash/PowerShell), tools, libraries, apps.
- **Windows vs Linux**: different kernels → different syscalls & tools.

### 1.2 Where WSL fits
- **WSL** lets you run a **Linux userspace** on Windows.
- **WSL1**: translates Linux syscalls to Windows kernel calls.
- **WSL2**: runs a **real Linux kernel** in a lightweight VM (higher compatibility/perf).

### 1.3 WSL1 vs WSL2 (quick compare)

| Aspect              | WSL1                            | WSL2 (recommended)                                 |
|---------------------|----------------------------------|----------------------------------------------------|
| Kernel              | Translation layer               | Real Linux kernel (managed VM)                     |
| FS perf in `~/`     | N/A                             | **Fast**                                           |
| FS perf in `/mnt/*` | Generally OK                    | **Slower**; use for access, not hot dev            |
| File watching       | Bridged                         | Native in `~/`; **flaky on `/mnt/*`**              |
| Networking          | Shares host IP                  | NAT/virtualized (different IP)                     |
| Compatibility       | Good                            | **Very high**                                      |

**Rule of thumb:** keep repos in **`~/`**; use `/mnt/c|d|e` only to *access* Windows files.

---

## 2. Shells & Environments

### 2.1 Bash (Linux) vs PowerShell (Windows)

| Topic        | Bash                                  | PowerShell                       |
|--------------|---------------------------------------|----------------------------------|
| Prompt       | ``$``                                  | ``PS C:\>``                      |
| Scripting    | ``#!/usr/bin/env bash``                | ``.ps1`` (PowerShell syntax)     |
| Set env var  | ``FOO=bar cmd`` or ``export FOO=bar`` | ``$env:FOO='bar'; cmd``          |
| Paths        | ``/home/user``                        | ``C:\Users\User``                |
| Pipe model   | Text streams                          | .NET objects                     |

> Use **Bash in WSL** for Linux tools; **PowerShell** for Windows automation.

### 2.2 Profiles
- **`~/.bashrc`** (WSL/Bash): runs on interactive shells. Put aliases/exports here.
- **`~/.profile`**: login shell config (runs once per login).
- **PowerShell profile**: `echo $PROFILE` shows the path; put functions/aliases there.

**Common Bash additions:**
```bash
# ~/.bashrc
export GEM_HOME="$HOME/.gems"
export PATH="$HOME/.gems/bin:$PATH"
alias ll='ls -lah'
alias gs='git status -sb'
alias gp='git pull --rebase'
alias jserve='bundle exec jekyll serve --livereload --drafts --future --force_polling'
```

### 2.3 Environment Variables
- Temporary (one command): `FOO=1 cmd`
- Session-wide: `export FOO=1`
- Persistent: put in `~/.bashrc` then `source ~/.bashrc`


## 3. Filesystems & Paths

### 3.1 Where things live

On a real Linux machine, the root of OS `/` is on a Linux-formatted filesystem, (ext4, xfs, btrfs, etc.) sitting on a disk/partition and the home directory is typically on:
```bash
/home/<linux-user>     # shorthand: ~/
```

In WSL2, we run a real linux kernel inside a lighweight virtual machine. All ilnux files including `/home/<linux-user>` resides on a virtual disk on windows. The location is generally on a ext4.vhdx at:
```bash
C:\Users\<WindowsUser>\AppData\Local\Packages\<YourDistroPackage>\LocalState\ext4.vhdx
```
- **But don't access or change this file directly!**
- When we browse `\wsl$\Ubuntu\home\<user>\` from Windows explorer, we are looking inside that VHDX.
- Because `/home/<user>` is inside the VHDX, it’s fast and emits reliable file-change events (ideal for dev loops).

Windows drives are mounted into Linux under `/mnt` using a special filesystem driver called *DrvFs*.
Think of this as a bridge that lets Linux processes read/write Windows files. It’s super convenient, but:
- It emulates Linux-style permissions over NTFS
- It bridges file-change notifications (sometimes misses events),
- It’s generally slower for workloads with many small files.

That’s why tools like Jekyll/NPM/Webpack work best from ~/ (inside the VHDX), and need --force_polling if you we develop under /mnt/*.


- **Linux Home:**  `~/` = `/home/<user>` -> best place for code?
- **Windows drives under WSL:** `/mnt/c`, `/mnt/d`, -> convenient access, slower for dev loops

#### The short map of paths

| Concept| WSL (Ubuntu) path| What it is| Windows-visible path|
|----------------------|--------------------------------------------------|------------------------------------------------------------|--------------------------------------------------------|
| **Linux home** (your files, configs) | `/home/<your-linux-user>`<br>`~` (shorthand)          | Native **Linux filesystem** managed by WSL.<br>Fast I/O, reliable file-watch. | `\\wsl$\Ubuntu\home\<your-linux-user>\`<br>(open in Explorer) |
| **Windows user home** | `/mnt/c/Users/<YourWindowsName>`               | Your Windows profile, seen from **WSL** via the `/mnt/*` bridge. | `C:\Users\<YourWindowsName>`                         |
| **Windows drives**    | `/mnt/c/`, `/mnt/d/`, `/mnt/e/`                | Automatic mounts of Windows drives **inside WSL**.<br>Convenient access, but slower for dev & flaky file-watch. | `C:\`, `D:\`, `E:\`                                  |

#### How to see where we are and what it's mounted on 
```bash
pwd                 # current dir
df -h ~             # free/used space of your Linux home filesystem
findmnt ~           # shows the mount & filesystem backing your home

###### RESULT ->
prida01@DhanushPC:/mnt/e/MyBlog$ pwd
/mnt/e/MyBlog
prida01@DhanushPC:/mnt/e/MyBlog$ echo $HOME
/home/prida01
prida01@DhanushPC:/mnt/e/MyBlog$ df -h
Filesystem      Size  Used Avail Use% Mounted on
none            3.9G     0  3.9G   0% /usr/lib/modules/6.6.87.2-microsoft-standard-WSL2
none            3.9G  4.0K  3.9G   1% /mnt/wsl
drivers         364G  304G   61G  84% /usr/lib/wsl/drivers
/dev/sdd        251G  2.8G  236G   2% /
none            3.9G  488K  3.9G   1% /mnt/wslg
none            3.9G     0  3.9G   0% /usr/lib/wsl/lib
rootfs          3.9G  2.7M  3.9G   1% /init
none            3.9G  4.0K  3.9G   1% /run
none            3.9G     0  3.9G   0% /run/lock
none            3.9G     0  3.9G   0% /run/shm
none            3.9G     0  3.9G   0% /run/user
none            3.9G   76K  3.9G   1% /mnt/wslg/versions.txt
none            3.9G   76K  3.9G   1% /mnt/wslg/doc
C:\             364G  304G   61G  84% /mnt/c
D:\             293G   90G  204G  31% /mnt/d
E:\             262G   19G  244G   8% /mnt/e
```


Convert paths:
```bash
#bash
wslpath 'E:\MyBlog\GITBLOG'      # -> /mnt/e/MyBlog/GITBLOG
```

#### Shifting from `/mnt/` to `~`(using code I'd used previously) ->
```bash
mkdir -p ~/code
rsync -a --info=progress2 /mnt/e/MyBlog/GITBLOG/P-Dhanush.github.io/ ~/code/P-Dhanush.github.io/
cd ~/code/P-Dhanush.github.io
bundle install
bundle exec jekyll serve --livereload --drafts --future   # I was able to proceed without --force_polling
```

### 3.2 Performance & file-watching

- Hot-reload tools (Jekyll/Webpack/nodemon) watch files.
- In `~/`: events are reliable.
- In `/mnt/*`: events can be missed → use polling flags (e.g., `--force_polling`) or move repo to `~/`.

## 4. Processes, Networks, Interop

### 4.1 Run Windows apps from WSL and vice versa
```bash
# From WSL:
explorer.exe .          # open current folder in Windows Explorer
notepad.exe README.md   # open file in Notepad
powershell.exe -Command "Get-Process"   # run a PowerShell command

# From Windows PowerShell to WSL:
wsl ls -la
wsl -e bash -lc "echo hello from linux"
```

### 4.2 Networking Basics
- WSL2 uses NAT; it has its own IP.
- Access Windows services from WSL via `localhost` (mirrored by WSL).
- Access WSL services from Windows usually via `localhost:PORT` (modern WSL enables this by default). If not, check firewall.

## 5. Package Managers & Toolchains
### 5.1 apt vs winget/choco

Inside WSL: use `apt` for Linux packages.
Windows side: `winget` or `choco` for Windows apps.

### 5.2 Common stacks (quick installs)
```bash
# Compilers / build tools
sudo apt update
sudo apt install -y build-essential git curl

# Python (example)
sudo apt install -y python3 python3-venv python3-pip

# Node via nvm
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts

# Ruby/Jekyll
sudo apt install -y ruby-full zlib1g-dev
gem install bundler
```

## 6. Git & Projects
### 6.1 Where to clone

Prefer: `~/code/<project>` (fast, reliable).
Avoid for hot dev: `/mnt/c/...` (OK for occasional access/copies).

### 6.2 Useful Git setup
```bash
git config --global user.name  "Your Name"
git config --global user.email you@example.com
git config --global init.defaultBranch main
git config --global core.autocrlf input
git fetch --all --prune
```

Daily flows:

```bash
git pull --rebase
git stash push -u -m "WIP" && git stash pop
git log --oneline -n 5
```

## 7. Troubleshooting