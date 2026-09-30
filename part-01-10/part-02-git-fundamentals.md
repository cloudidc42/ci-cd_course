# Part 02: Version Control ด้วย Git

> **ระดับ:** เริ่มต้น (Beginner)
> **เวลาที่ใช้:** 4-6 ชั่วโมง
> **ข้อกำหนดเบื้องต้น:** ทักษะพื้นฐานการใช้ Terminal/Command Line

---

## สารบัญ

1. [Git คืออะไร และทำไมต้องใช้?](#1-git-คืออะไร-และทำไมต้องใช้)
2. [ติดตั้ง Git](#2-ติดตั้ง-git)
3. [Git Configuration เบื้องต้น](#3-git-configuration-เบื้องต้น)
4. [Git Repository Basics](#4-git-repository-basics)
5. [Core Git Commands](#5-core-git-commands)
6. [Working with Remote Repositories](#6-working-with-remote-repositories)
7. [.gitignore](#7-gitignore)
8. [Git Log, Diff, Status](#8-git-log-diff-status)
9. [Undoing Changes](#9-undoing-changes)
10. [Git Stash](#10-git-stash)
11. [SSH Keys Setup](#11-ssh-keys-setup)
12. [GitHub/GitLab Account Setup](#12-githubgitlab-account-setup)
13. [Git Best Practices](#13-git-best-practices)
14. [แบบฝึกหัด 10+ ข้อ](#14-แบบฝึกหัด)

---

## 1. Git คืออะไร และทำไมต้องใช้?

### 1.1 ปัญหาก่อนมี Version Control

```
สถานการณ์จริงก่อนมี Git:

project_folder/
├── app.py
├── app_backup.py
├── app_v2.py
├── app_v2_final.py
├── app_v2_final_REAL.py
├── app_v2_final_REAL_2.py
├── app_v2_final_REAL_2_tested.py
└── app_DONT_USE.py

คำถาม: version ไหนคือ production version???
```

**ปัญหาที่เจอ:**
- ไม่รู้ว่า version ไหน latest
- ไม่รู้ว่าใครแก้ไขอะไร เมื่อไหร่
- ถ้าแก้แล้วพัง ย้อนกลับยากมาก
- ทำงานหลายคนพร้อมกันไม่ได้

### 1.2 Git แก้ปัญหาอย่างไร?

```
ด้วย Git:

project_folder/
└── app.py  ← ไฟล์เดียว แต่มี history ครบ

git log:
commit a1b2c3d - John: "Fix payment calculation bug" (2 ชั่วโมงที่แล้ว)
commit e4f5g6h - Jane: "Add user authentication" (เมื่อวาน)
commit i7j8k9l - John: "Initial project setup" (3 วันที่แล้ว)

สามารถ:
✅ รู้ว่าใครทำอะไรเมื่อไหร่
✅ ย้อนกลับไป version ไหนก็ได้
✅ ทำงานหลายคนพร้อมกันได้
✅ เปรียบเทียบ version ต่างๆ ได้
```

### 1.3 Git Architecture

```
Git มี 3 "พื้นที่" หลัก:

Working Directory    Staging Area     Local Repository
(ไฟล์ที่กำลังแก้)   (เตรียม commit)   (ประวัติ commits)
      │                   │                  │
      │   git add         │   git commit     │
      │ ──────────────→   │ ─────────────→   │
      │                   │                  │
      │ ←──────────────   │                  │
      │  git checkout     │                  │
      │                   │                  │
      │ ←────────────────────────────────    │
      │         git checkout <commit>        │
```

**และยังมี Remote Repository:**
```
Local Repository    Remote Repository
(เครื่องของคุณ)     (GitHub/GitLab)
      │                    │
      │   git push         │
      │ ─────────────────→ │
      │                    │
      │ ←───────────────── │
      │      git pull      │
```

---

## 2. ติดตั้ง Git

### 2.1 ติดตั้งบน macOS

```bash
# วิธีที่ 1: ใช้ Homebrew (แนะนำ)
brew install git

# วิธีที่ 2: ใช้ Xcode Command Line Tools
xcode-select --install
# กด "Install" เมื่อมี popup

# ตรวจสอบการติดตั้ง
git --version
# ควรได้: git version 2.x.x
```

### 2.2 ติดตั้งบน Ubuntu/Debian Linux

```bash
# อัพเดต package list ก่อน
sudo apt update

# ติดตั้ง Git
sudo apt install git -y

# ตรวจสอบ
git --version
```

### 2.3 ติดตั้งบน CentOS/RHEL/Fedora

```bash
# CentOS/RHEL
sudo yum install git -y

# Fedora
sudo dnf install git -y

# ตรวจสอบ
git --version
```

### 2.4 ติดตั้งบน Windows

```powershell
# วิธีที่ 1: ดาวน์โหลดจาก https://git-scm.com/download/win
# วิธีที่ 2: ใช้ winget
winget install Git.Git

# วิธีที่ 3: ใช้ Chocolatey
choco install git

# ตรวจสอบใน Git Bash หรือ PowerShell
git --version
```

### 2.5 ตรวจสอบการติดตั้ง

```bash
# ดู git version
git --version

# ดู git location
which git        # Linux/macOS
where git        # Windows

# ดู git help
git help
git help commit  # help สำหรับ command เฉพาะ
```

---

## 3. Git Configuration เบื้องต้น

### 3.1 ตั้งค่า Identity

สิ่งแรกที่ต้องทำหลังติดตั้ง Git คือบอก Git ว่าเราคือใคร

```bash
# ตั้งชื่อ (ชื่อนี้จะปรากฏใน commit history)
git config --global user.name "John Doe"

# ตั้ง email (ควรเป็น email เดียวกับ GitHub/GitLab)
git config --global user.email "john.doe@example.com"

# ตรวจสอบ
git config --global user.name
git config --global user.email
```

### 3.2 ตั้งค่า Default Editor

```bash
# ใช้ VS Code
git config --global core.editor "code --wait"

# ใช้ Vim
git config --global core.editor "vim"

# ใช้ nano (ง่ายกว่า Vim สำหรับมือใหม่)
git config --global core.editor "nano"

# ใช้ Notepad++ (Windows)
git config --global core.editor "'C:/Program Files/Notepad++/notepad++.exe' -multiInst -notabbar -nosession -noPlugin"
```

### 3.3 ตั้งค่า Default Branch Name

```bash
# ตั้ง default branch name เป็น "main" (modern standard)
git config --global init.defaultBranch main
```

### 3.4 ตั้งค่า Line Endings

```bash
# macOS/Linux
git config --global core.autocrlf input

# Windows
git config --global core.autocrlf true
```

### 3.5 ตั้งค่า Aliases (shortcuts)

```bash
# สร้าง shortcuts ที่ใช้บ่อย
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.undo "reset HEAD~1 --soft"

# ทดสอบ alias
git st          # เหมือนกับ git status
git lg          # pretty log
```

### 3.6 ดู Configuration ทั้งหมด

```bash
# ดู config ทั้งหมด
git config --list

# ดู config file โดยตรง
cat ~/.gitconfig

# ตัวอย่างไฟล์ ~/.gitconfig
# [user]
#     name = John Doe
#     email = john.doe@example.com
# [core]
#     editor = code --wait
#     autocrlf = input
# [init]
#     defaultBranch = main
# [alias]
#     st = status
#     co = checkout
```

### 3.7 Config Levels

```bash
# Git config มี 3 ระดับ:

# 1. System (ทุก user บน machine นี้)
git config --system user.name "..."
# เก็บที่: /etc/gitconfig

# 2. Global (user ปัจจุบัน ทุก repo)
git config --global user.name "..."
# เก็บที่: ~/.gitconfig

# 3. Local (เฉพาะ repo นี้เท่านั้น)
git config --local user.name "..."
# เก็บที่: .git/config

# Priority: Local > Global > System
```

---

## 4. Git Repository Basics

### 4.1 git init - สร้าง Repository ใหม่

```bash
# สร้าง directory ใหม่และ init git
mkdir my-project
cd my-project
git init

# Output:
# Initialized empty Git repository in /home/user/my-project/.git/

# ดูว่ามี .git folder
ls -la
# drwxr-xr-x  .git
# (hidden folder เก็บ git data ทั้งหมด)

# ดูโครงสร้าง .git
ls .git/
# HEAD  config  description  hooks/  info/  objects/  refs/
```

```bash
# สร้าง git repo ใน folder ที่มีอยู่แล้ว
cd existing-project
git init

# Init repo ใน folder อื่นโดยไม่ต้อง cd
git init /path/to/new-repo
```

### 4.2 git clone - คัดลอก Repository จาก Remote

```bash
# Clone จาก GitHub ด้วย HTTPS
git clone https://github.com/username/repository.git

# Clone แล้วเปลี่ยนชื่อ folder
git clone https://github.com/username/repository.git my-custom-name

# Clone ด้วย SSH (ต้องตั้งค่า SSH key ก่อน)
git clone git@github.com:username/repository.git

# Clone เฉพาะ branch เดียว
git clone --branch develop https://github.com/username/repo.git

# Clone แบบ shallow (ดึงแค่ commit ล่าสุด เร็วกว่า)
git clone --depth 1 https://github.com/username/repo.git

# ดูผล clone
ls -la repository/
```

### 4.3 โครงสร้าง .git Directory

```bash
# ดูโครงสร้างภายใน .git
tree .git/
# .git/
# ├── HEAD              ← ชี้ไปยัง current branch
# ├── config            ← local git config
# ├── description       ← ชื่อ repo (GitWeb ใช้)
# ├── hooks/            ← git hooks scripts
# │   ├── pre-commit.sample
# │   ├── pre-push.sample
# │   └── ...
# ├── info/
# │   └── exclude       ← gitignore สำหรับ local เท่านั้น
# ├── objects/          ← ข้อมูล commits, trees, blobs
# │   ├── info/
# │   └── pack/
# └── refs/             ← branch และ tag references
#     ├── heads/        ← local branches
#     └── tags/         ← tags

# ดู current branch
cat .git/HEAD
# ref: refs/heads/main
```

---

## 5. Core Git Commands

### 5.1 git status - ดูสถานะปัจจุบัน

```bash
# ดูสถานะ
git status

# ตัวอย่าง output:
# On branch main
# Your branch is up to date with 'origin/main'.
#
# Changes to be committed:
#   (use "git restore --staged <file>..." to unstage)
#         new file:   newfile.txt
#
# Changes not staged for commit:
#   (use "git add <file>..." to update what will be committed)
#   (use "git restore <file>..." to discard changes in working directory)
#         modified:   existing.txt
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#         untracked.txt

# Short format
git status -s
# M  existing.txt    ← Modified, staged
#  M other.txt       ← Modified, unstaged
# ?? untracked.txt   ← Untracked
# A  newfile.txt     ← Added (staged)
```

### 5.2 git add - เพิ่มไฟล์ไปยัง Staging Area

```bash
# เพิ่มไฟล์เดียว
git add filename.txt

# เพิ่มหลายไฟล์
git add file1.txt file2.txt file3.txt

# เพิ่มทุกไฟล์ใน directory ปัจจุบัน
git add .

# เพิ่มทุกไฟล์ทั้ง repo
git add -A
git add --all

# เพิ่มไฟล์แบบ interactive (เลือกทีละส่วน)
git add -p filename.txt
# Git จะแสดง hunk ทีละส่วน ให้กด:
# y = เพิ่ม hunk นี้
# n = ข้าม hunk นี้
# s = แยก hunk เล็กลง
# q = ออก

# เพิ่มไฟล์ทุกตัวที่ match pattern
git add "*.txt"
git add src/

# ตรวจสอบว่า stage อะไรบ้าง
git status
```

### 5.3 git commit - บันทึก Snapshot

```bash
# Commit พร้อม message
git commit -m "Add user authentication feature"

# Commit แบบเปิด editor ให้เขียน message ยาวๆ
git commit

# Commit ทุกไฟล์ที่ track แล้ว (ข้าม staging)
git commit -am "Fix typo in README"
# -a = add ทุก tracked files
# -m = message

# Commit ด้วย body message ที่ยาวกว่า
git commit -m "feat: add user authentication

This commit adds JWT-based user authentication including:
- Login endpoint with email/password
- Token generation and validation  
- Password hashing with bcrypt
- Rate limiting on login attempts

Closes #123"

# Amend commit ล่าสุด (แก้ message หรือเพิ่มไฟล์)
git add forgotten-file.txt
git commit --amend
# หรือ
git commit --amend -m "Updated commit message"
# ⚠️ อย่า amend commits ที่ push ไปแล้ว!
```

### 5.4 Commit Message Best Practices

```bash
# Format ที่ดี (Conventional Commits):
# <type>(<scope>): <subject>
#
# <body>
#
# <footer>

# Types:
# feat     = new feature
# fix      = bug fix
# docs     = documentation only
# style    = formatting (no code change)
# refactor = code restructure
# test     = adding tests
# chore    = build process, tools

# ตัวอย่างที่ดี:
git commit -m "feat(auth): add Google OAuth login"
git commit -m "fix(payment): correct tax calculation for EU countries"
git commit -m "docs(api): update endpoint documentation"
git commit -m "test(user): add unit tests for password reset"

# ตัวอย่างที่ไม่ดี:
git commit -m "fix"           # ❌ ไม่บอกว่าแก้อะไร
git commit -m "changes"       # ❌ ไม่มีความหมาย
git commit -m "WIP"           # ❌ ยังไม่เสร็จ ไม่ควร commit
git commit -m "asdfgh"        # ❌ random text
```

### 5.5 git push - ส่ง Commits ไปยัง Remote

```bash
# Push branch ปัจจุบัน
git push

# Push และตั้ง upstream (ครั้งแรก)
git push -u origin main
# -u หรือ --set-upstream = บอก git ว่า remote branch คือ origin/main

# Push branch เฉพาะ
git push origin feature/my-feature

# Push ทุก branches
git push --all origin

# Push tags
git push --tags

# Force push (อันตราย! ใช้เมื่อจำเป็น)
git push --force origin feature/my-feature
# แนะนำใช้ --force-with-lease แทน (ปลอดภัยกว่า)
git push --force-with-lease origin feature/my-feature

# ลบ remote branch
git push origin --delete feature/old-feature
```

### 5.6 git pull - ดึง Code จาก Remote

```bash
# Pull (fetch + merge)
git pull

# Pull จาก remote เฉพาะ
git pull origin main

# Pull ด้วย rebase แทน merge
git pull --rebase
git pull --rebase origin main

# Pull แบบ fast-forward เท่านั้น
git pull --ff-only

# ดูว่า pull จะทำอะไรก่อน (dry run)
git fetch
git log HEAD..origin/main --oneline
```

### 5.7 git fetch - ดึงข้อมูลจาก Remote

```bash
# Fetch ข้อมูลจาก origin
git fetch

# Fetch จาก remote เฉพาะ
git fetch origin

# Fetch ทุก remotes
git fetch --all

# Fetch และลบ remote branches ที่ถูกลบแล้ว
git fetch --prune

# Fetch เฉพาะ branch เดียว
git fetch origin main:main
```

**ความแตกต่างระหว่าง fetch และ pull:**
```
git fetch:
├── ดึงข้อมูลจาก remote
├── อัพเดต remote tracking branches (origin/main, etc.)
└── ไม่แตะ working directory หรือ local branches

git pull = git fetch + git merge:
├── ดึงข้อมูลจาก remote
└── merge เข้า current branch ทันที
```

---

## 6. Working with Remote Repositories

### 6.1 git remote - จัดการ Remote Repositories

```bash
# ดู remotes ทั้งหมด
git remote
# origin

# ดูพร้อม URL
git remote -v
# origin  https://github.com/user/repo.git (fetch)
# origin  https://github.com/user/repo.git (push)

# เพิ่ม remote ใหม่
git remote add upstream https://github.com/original/repo.git

# เปลี่ยน URL ของ remote
git remote set-url origin git@github.com:user/repo.git

# ลบ remote
git remote remove upstream

# เปลี่ยนชื่อ remote
git remote rename origin old-origin
```

### 6.2 ทำงานกับ Fork

```bash
# Scenario: Fork project จาก GitHub แล้วต้องการ sync

# 1. Clone fork ของคุณ
git clone https://github.com/YOUR-USERNAME/repo.git
cd repo

# 2. เพิ่ม upstream (repo ต้นทาง)
git remote add upstream https://github.com/ORIGINAL-OWNER/repo.git

# 3. ดู remotes
git remote -v
# origin    https://github.com/YOUR-USERNAME/repo.git (fetch)
# origin    https://github.com/YOUR-USERNAME/repo.git (push)
# upstream  https://github.com/ORIGINAL-OWNER/repo.git (fetch)
# upstream  https://github.com/ORIGINAL-OWNER/repo.git (push)

# 4. Sync กับ upstream
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

---

## 7. .gitignore

### 7.1 .gitignore คืออะไร?

`.gitignore` คือไฟล์ที่บอก Git ว่าไม่ต้องติดตามไฟล์หรือ folder เหล่านี้

### 7.2 ทำไมต้องใช้ .gitignore?

```
ไฟล์ที่ไม่ควร commit ขึ้น Git:
├── .env, .env.local     ← credentials/secrets (อันตรายมาก!)
├── node_modules/        ← ขนาดใหญ่, สร้างใหม่ได้
├── __pycache__/         ← Python compiled files
├── .DS_Store            ← macOS metadata
├── *.log                ← log files
├── build/, dist/        ← compiled code
└── .idea/, .vscode/     ← IDE settings (บางทีก็ commit)
```

### 7.3 สร้าง .gitignore

```bash
# สร้างไฟล์ .gitignore
touch .gitignore

# เพิ่ม content
cat > .gitignore << 'EOF'
# Dependencies
node_modules/
vendor/

# Environment files (IMPORTANT: never commit these!)
.env
.env.local
.env.*.local

# Build output
dist/
build/
*.egg-info/
__pycache__/
*.pyc
*.pyo

# Logs
*.log
logs/
npm-debug.log*
yarn-debug.log*

# OS files
.DS_Store
.DS_Store?
._*
Thumbs.db
ehthumbs.db

# IDE files
.idea/
*.swp
*.swo
*~

# Testing
coverage/
.coverage
.pytest_cache/
htmlcov/

# Terraform
*.tfstate
*.tfstate.backup
.terraform/

EOF
```

### 7.4 .gitignore Patterns

```bash
# Pattern Rules:
# Lines starting with # are comments
# Blank lines are ignored
# / at the end = directory only
# / at the beginning = relative to root only
# ! = negate (include this file)
# * = any string (except /)
# ** = any path
# ? = any single character

# ตัวอย่าง patterns:

# Ignore all .log files
*.log

# Ignore all files in logs/ directory
logs/

# Ignore temp.txt in root only
/temp.txt

# Ignore everything in build/ directory
build/

# Ignore all .pdf files in docs/
docs/*.pdf

# Ignore all .pdf files recursively
**/*.pdf

# Don't ignore IMPORTANT.log even though we ignore *.log
!IMPORTANT.log

# Ignore todo.txt in any directory
todo.txt

# Only ignore todo.txt in root
/todo.txt
```

### 7.5 Global .gitignore

```bash
# สร้าง global gitignore สำหรับไฟล์ OS/Editor
cat > ~/.gitignore_global << 'EOF'
# macOS
.DS_Store
.AppleDouble
.LSOverride
._*
.Spotlight-V100
.Trashes

# Windows
Thumbs.db
ehthumbs.db
Desktop.ini

# Linux
*~

# VS Code
.vscode/
*.code-workspace

# JetBrains
.idea/
*.iml

# Vim
*.swp
*.swo
EOF

# บอก git ให้ใช้ global gitignore
git config --global core.excludesfile ~/.gitignore_global
```

### 7.6 Untrack ไฟล์ที่ commit ไปแล้ว

```bash
# ถ้า commit ไฟล์ที่ควร ignore ไปแล้ว

# 1. เพิ่ม pattern ใน .gitignore
echo "secrets.txt" >> .gitignore

# 2. Remove จาก git index (แต่ยังเก็บไฟล์ไว้)
git rm --cached secrets.txt

# 3. หากต้องการ remove directory
git rm -r --cached node_modules/

# 4. Commit การเปลี่ยนแปลง
git add .gitignore
git commit -m "chore: remove tracked files that should be ignored"
```

### 7.7 ใช้ gitignore.io

```bash
# สร้าง .gitignore อัตโนมัติจาก gitignore.io
# ไปที่: https://www.toptal.com/developers/gitignore

# หรือใช้ curl
curl -sL https://www.toptal.com/developers/gitignore/api/node,python,macos,linux > .gitignore
```

---

## 8. Git Log, Diff, Status

### 8.1 git log - ดู Commit History

```bash
# Log พื้นฐาน
git log

# Log แบบสั้น (หนึ่ง commit ต่อบรรทัด)
git log --oneline

# Log พร้อม graph
git log --oneline --graph

# Log พร้อม branches ทั้งหมด
git log --oneline --graph --all --decorate

# Log จำนวน commits ที่ต้องการ
git log -5
git log -n 10

# Log ของ author เฉพาะ
git log --author="John Doe"

# Log ใน date range
git log --after="2024-01-01" --before="2024-12-31"

# Log ที่เกี่ยวกับไฟล์เฉพาะ
git log -- filename.txt

# Log ที่มี keyword ใน message
git log --grep="fix"

# Log พร้อมดู changes (patch)
git log -p

# Log แบบ stats
git log --stat

# Format log เอง
git log --format="%h %an %ar %s"
# %h = short hash
# %H = full hash
# %an = author name
# %ae = author email
# %ar = relative date
# %ai = absolute date
# %s = subject (first line of message)
```

### 8.2 git diff - เปรียบเทียบความแตกต่าง

```bash
# ดู unstaged changes (working dir vs staging area)
git diff

# ดู staged changes (staging area vs last commit)
git diff --staged
git diff --cached

# ดู changes ระหว่าง commits
git diff abc1234 def5678

# ดู changes ระหว่าง commit กับ HEAD
git diff HEAD~1 HEAD

# ดู changes ระหว่าง branches
git diff main feature/my-feature

# ดู changes ของไฟล์เฉพาะ
git diff -- filename.txt

# ดูแค่ชื่อไฟล์ที่เปลี่ยน
git diff --name-only
git diff --name-status

# ดู diff แบบ stats
git diff --stat

# ตัวอย่าง diff output:
# diff --git a/app.py b/app.py
# index abc1234..def5678 100644
# --- a/app.py
# +++ b/app.py
# @@ -10,7 +10,8 @@
#  def calculate_price(amount):
# -    return amount * 1.07
# +    tax_rate = 0.07
# +    return amount * (1 + tax_rate)
```

### 8.3 git show - ดู Commit เฉพาะ

```bash
# ดู commit ล่าสุด
git show

# ดู commit เฉพาะ
git show abc1234

# ดู HEAD-2 (2 commits ก่อน HEAD)
git show HEAD~2

# ดูแค่ stats
git show --stat abc1234

# ดูไฟล์ใน commit เฉพาะ
git show abc1234:path/to/file.txt
```

---

## 9. Undoing Changes

### 9.1 Overview ของการ Undo

```
สถานการณ์:                     วิธีแก้:
─────────────────────────────────────────────────────
แก้ไขไฟล์แล้วอยาก discard     git restore <file>
ไฟล์อยู่ใน staging อยากเอาออก  git restore --staged <file>
อยาก undo commit ล่าสุด       git reset HEAD~1
อยาก undo commit เก่า          git revert <commit>
ลบไฟล์โดยไม่ตั้งใจ             git restore <file>
```

### 9.2 git restore - คืนค่า Working Directory

```bash
# Discard changes ใน working directory (ยังไม่ได้ add)
git restore filename.txt

# Discard changes ทุกไฟล์
git restore .

# เอาไฟล์ออกจาก staging area (แต่เก็บ changes ไว้)
git restore --staged filename.txt

# เอาไฟล์ทั้งหมดออกจาก staging
git restore --staged .

# Restore ไฟล์จาก commit เฉพาะ
git restore --source abc1234 filename.txt

# Restore จาก HEAD~2
git restore --source HEAD~2 filename.txt

# ⚠️ WARNING: git restore จะลบ changes ที่ยังไม่ commit อย่างถาวร!
```

### 9.3 git reset - ย้อนกลับ Commits

```bash
# Reset แบบต่างๆ:

# --soft: เอา commits กลับมา แต่ changes ยังอยู่ใน staging
git reset --soft HEAD~1   # ย้อนกลับ 1 commit
git reset --soft abc1234  # ย้อนกลับไป commit นั้น

# --mixed (default): เอา commits กลับมา changes กลับไป unstaged
git reset HEAD~1          # ย้อนกลับ 1 commit
git reset --mixed HEAD~3  # ย้อนกลับ 3 commits

# --hard: เอา commits กลับมา และลบ changes ทิ้ง (อันตราย!)
git reset --hard HEAD~1   # ย้อนกลับ 1 commit พร้อมลบ changes
git reset --hard abc1234  # ย้อนกลับไปและลบ changes

# ตัวอย่างการใช้:
# ลืม add ไฟล์ หรือ commit message ผิด:
git reset --soft HEAD~1
# แก้ไขแล้ว commit ใหม่

# อยาก unstage ไฟล์:
git reset HEAD filename.txt
# เหมือนกับ git restore --staged filename.txt

# ⚠️ อย่า reset commits ที่ push ไปแล้ว!
```

### 9.4 git revert - ยกเลิก Commit อย่างปลอดภัย

```bash
# revert สร้าง commit ใหม่ที่ยกเลิก changes ของ commit เก่า
# ปลอดภัยกว่า reset เพราะไม่ล้าง history

# Revert commit เฉพาะ
git revert abc1234

# Revert โดยไม่เปิด editor
git revert abc1234 --no-edit

# Revert commit ล่าสุด
git revert HEAD

# Revert แต่ไม่ commit ทันที (เตรียมไว้ก่อน)
git revert --no-commit abc1234

# Revert หลาย commits
git revert abc1234 def5678 ghi9012

# เมื่อไหร่ควรใช้ revert แทน reset:
# - ใช้ revert เมื่อต้องการ undo commit ที่ push ไปแล้ว
# - ใช้ reset เฉพาะ local commits ที่ยังไม่ push
```

### 9.5 ตัวอย่าง Scenarios การ Undo

```bash
# Scenario 1: แก้ไขไฟล์แล้วเสียใจ (ยังไม่ add)
echo "bad code" >> app.py
git restore app.py                    # ✅ คืนค่าเดิม

# Scenario 2: add ไปแล้วอยากเอาออก
git add app.py
git restore --staged app.py           # ✅ เอาออกจาก staging

# Scenario 3: commit ไปแล้วอยาก uncommit (local เท่านั้น)
git commit -m "oops wrong commit"
git reset --soft HEAD~1               # ✅ ยกเลิก commit แต่เก็บ changes

# Scenario 4: commit message ผิด
git commit -m "wronf mespag"
git commit --amend -m "correct message"  # ✅ แก้ message

# Scenario 5: push ไปแล้ว ต้อง revert
git revert HEAD
git push                              # ✅ ปลอดภัย

# Scenario 6: ลบไฟล์โดยไม่ตั้งใจ
rm important.txt
git restore important.txt             # ✅ คืน file กลับมา

# Scenario 7: อยากกลับไปดู version เก่าของไฟล์
git show HEAD~5:app.py > app_old.py   # ✅ บันทึกเป็นไฟล์ใหม่
```

---

## 10. Git Stash

### 10.1 git stash คืออะไร?

Stash คือการ "เก็บงานชั่วคราว" เมื่อต้องสลับ context โดยยังไม่พร้อม commit

```
Scenario ที่ต้องใช้ stash:

กำลัง code feature ใหม่ค้างอยู่
→ มี urgent bug ต้องแก้ด่วน
→ ไม่อยากทำ commit งานที่ยังไม่เสร็จ
→ ใช้ stash เก็บงานชั่วคราว
→ แก้ bug แล้ว commit
→ เอา stash กลับมา continue งาน
```

### 10.2 คำสั่ง git stash

```bash
# Stash changes ปัจจุบัน (tracked files เท่านั้น)
git stash

# Stash พร้อม message
git stash push -m "work in progress: user profile feature"
# หรือ (เก่า)
git stash save "work in progress: user profile feature"

# Stash รวม untracked files
git stash -u
git stash --include-untracked

# Stash รวม ignored files ด้วย
git stash -a
git stash --all

# ดู stashes ทั้งหมด
git stash list
# stash@{0}: On main: work in progress: user profile feature
# stash@{1}: WIP on feature/login: abc1234 fix auth
# stash@{2}: WIP on main: def5678 add home page

# นำ stash กลับมา (ล่าสุด และลบออกจาก list)
git stash pop

# นำ stash เฉพาะกลับมา
git stash pop stash@{2}

# นำ stash กลับมา แต่เก็บไว้ใน list ด้วย
git stash apply
git stash apply stash@{1}

# ดู changes ใน stash
git stash show
git stash show stash@{1}
git stash show -p stash@{0}   # แบบ diff

# ลบ stash เฉพาะ
git stash drop stash@{1}

# ลบ stashes ทั้งหมด
git stash clear

# สร้าง branch จาก stash
git stash branch new-branch-name stash@{0}
```

### 10.3 Stash Workflow ตัวอย่าง

```bash
# ขั้นตอนที่ 1: กำลัง work บน feature
git checkout -b feature/user-profile
# แก้ไข user.py, profile.html, styles.css

# ขั้นตอนที่ 2: มี urgent bug fix
git stash push -m "WIP: user profile page"

# ขั้นตอนที่ 3: แก้ bug
git checkout main
git checkout -b hotfix/login-bug
# แก้ไข auth.py
git commit -m "fix: resolve login authentication bug"
git checkout main
git merge hotfix/login-bug

# ขั้นตอนที่ 4: กลับไป work ต่อ
git checkout feature/user-profile
git stash pop
# ต่อ code ได้เลย!
```

---

## 11. SSH Keys Setup

### 11.1 ทำไมต้องใช้ SSH?

```
HTTPS authentication:
- ต้อง enter username/password ทุกครั้ง
- ใช้ Personal Access Token (PAT)

SSH authentication:
- ไม่ต้อง enter password
- ปลอดภัยกว่า
- เหมาะสำหรับ automation
```

### 11.2 สร้าง SSH Key Pair

```bash
# สร้าง SSH key pair (Ed25519 - แนะนำ)
ssh-keygen -t ed25519 -C "your.email@example.com"
# -t = type (ed25519 หรือ rsa)
# -C = comment (ใส่ email เพื่อ identify)

# หรือ RSA (ถ้า server เก่า)
ssh-keygen -t rsa -b 4096 -C "your.email@example.com"

# ระหว่าง generate จะถามว่า:
# Enter file: กด Enter เพื่อใช้ path default (~/.ssh/id_ed25519)
# Enter passphrase: ใส่ passphrase (แนะนำ) หรือกด Enter ข้าม

# ดูไฟล์ที่สร้าง
ls ~/.ssh/
# id_ed25519      ← private key (ห้ามแชร์เด็ดขาด!)
# id_ed25519.pub  ← public key (ใช้ add ใน GitHub)
# known_hosts     ← hosts ที่เคย connect

# ดู public key
cat ~/.ssh/id_ed25519.pub
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... your.email@example.com
```

### 11.3 เพิ่ม SSH Key ใน GitHub

```bash
# 1. Copy public key
cat ~/.ssh/id_ed25519.pub | pbcopy   # macOS
cat ~/.ssh/id_ed25519.pub | xclip    # Linux (ต้องติดตั้ง xclip)
cat ~/.ssh/id_ed25519.pub            # Windows (copy ด้วยมือ)

# 2. ไปที่ GitHub.com
# Settings → SSH and GPG keys → New SSH key
# Title: "My MacBook Pro" (ชื่อ device)
# Key type: Authentication Key
# Key: paste public key ที่ copy มา
# Click "Add SSH key"

# 3. ทดสอบ connection
ssh -T git@github.com
# Hi username! You've successfully authenticated...
```

### 11.4 SSH Agent (เพื่อไม่ต้อง enter passphrase บ่อยๆ)

```bash
# Start SSH agent
eval "$(ssh-agent -s)"

# Add private key ไปยัง agent
ssh-add ~/.ssh/id_ed25519

# บน macOS เพิ่ม --apple-use-keychain
ssh-add --apple-use-keychain ~/.ssh/id_ed25519

# ดู keys ที่ add ไว้
ssh-add -l

# ตั้งค่า ~/.ssh/config เพื่อให้ auto-load
cat > ~/.ssh/config << 'EOF'
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
  
Host gitlab.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
EOF
```

### 11.5 ใช้หลาย SSH Keys

```bash
# ถ้ามีหลาย GitHub accounts หรือ GitLab

# สร้าง keys แยก
ssh-keygen -t ed25519 -C "work@company.com" -f ~/.ssh/id_ed25519_work
ssh-keygen -t ed25519 -C "personal@gmail.com" -f ~/.ssh/id_ed25519_personal

# ตั้งค่า ~/.ssh/config
cat > ~/.ssh/config << 'EOF'
# Work GitHub
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work

# Personal GitHub
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
EOF

# ใช้ SSH URL พิเศษ
git clone git@github-work:company/project.git
git clone git@github-personal:username/project.git

# หรือเปลี่ยน remote URL ที่มีอยู่
git remote set-url origin git@github-work:company/project.git
```

---

## 12. GitHub/GitLab Account Setup

### 12.1 สร้าง GitHub Account

```
1. ไปที่ github.com
2. Click "Sign up"
3. กรอก: username, email, password
4. ยืนยัน email
5. เลือก plan (Free ก็ใช้ได้)
```

### 12.2 ตั้งค่า GitHub Profile

```bash
# ตั้งค่าสำคัญใน GitHub:
# Settings → Profile:
# - ใส่ชื่อจริง
# - ใส่ bio
# - ใส่ location
# - ใส่ website/blog

# Settings → Security:
# - Enable 2FA (Two-Factor Authentication) ← สำคัญมาก!
```

### 12.3 Personal Access Token (PAT)

```bash
# ถ้าต้องใช้ HTTPS แทน SSH

# สร้าง PAT:
# GitHub Settings → Developer settings → Personal access tokens
# → Tokens (classic) → Generate new token

# ตั้งค่า:
# Note: "Git CLI - My MacBook"
# Expiration: 90 days
# Scopes: repo (full control)

# ใช้ PAT แทน password:
git clone https://github.com/username/repo.git
# Username: your-github-username
# Password: ghp_xxxxxxxxxxxx (PAT ที่สร้าง)

# หรือตั้งค่า credential store
git config --global credential.helper store
# หรือ cache (30 นาที)
git config --global credential.helper cache
git config --global credential.helper 'cache --timeout=3600'
```

### 12.4 สร้าง Repository ใหม่บน GitHub

```bash
# วิธีที่ 1: ผ่าน GitHub Web UI
# 1. Click "+" → "New repository"
# 2. ตั้งชื่อ repository
# 3. เลือก Public หรือ Private
# 4. (Optional) Add README, .gitignore, License
# 5. Click "Create repository"

# วิธีที่ 2: ใช้ GitHub CLI (gh)
# ติดตั้ง: https://cli.github.com

# Login
gh auth login

# สร้าง repo
gh repo create my-project --public
gh repo create my-project --private
gh repo create my-project --public --clone

# หลังสร้างแล้ว เชื่อมต่อ local repo
cd my-project
git remote add origin git@github.com:username/my-project.git
git push -u origin main
```

### 12.5 GitHub CLI Commands ที่ใช้บ่อย

```bash
# Install GitHub CLI
# macOS:
brew install gh

# Ubuntu:
sudo apt install gh

# Login
gh auth login

# สร้าง PR
gh pr create --title "Add new feature" --body "Description here"

# List PRs
gh pr list

# View PR
gh pr view 123

# Merge PR
gh pr merge 123

# Clone repo
gh repo clone username/repo

# Fork repo
gh repo fork username/repo

# View repo
gh repo view username/repo
```

---

## 13. Git Best Practices

### 13.1 Commit Best Practices

```bash
# ✅ DO: Commit เล็กๆ บ่อยๆ
git commit -m "feat: add user login endpoint"
git commit -m "test: add login endpoint tests"
git commit -m "docs: update API documentation"

# ❌ DON'T: Commit ใหญ่ๆ ที่ทำหลายอย่าง
git commit -m "add login, fix bugs, update docs, refactor auth"

# ✅ DO: Commit ที่สมบูรณ์ (passes tests)
# ❌ DON'T: Commit code ที่ broken

# ✅ DO: Write meaningful commit messages
git commit -m "fix: resolve race condition in payment processing"
# ❌ DON'T: 
git commit -m "fix"
git commit -m "."
git commit -m "test123"
```

### 13.2 Branch Best Practices

```bash
# ✅ DO: ตั้งชื่อ branch ที่ descriptive
git checkout -b feature/user-authentication
git checkout -b fix/login-race-condition
git checkout -b hotfix/payment-critical-bug
git checkout -b docs/update-api-readme

# ❌ DON'T:
git checkout -b john-branch
git checkout -b new-feature
git checkout -b test123

# ✅ DO: ลบ branch หลัง merge
git branch -d feature/completed-feature

# ✅ DO: Pull ก่อน push เสมอ
git pull --rebase origin main
git push origin feature/my-feature
```

### 13.3 Security Best Practices

```bash
# ❌ ห้ามทำเด็ดขาด:
git add .env              # commit secrets
git add credentials.json  # commit API keys
git push --force origin main  # force push main

# ✅ ควรทำ:
# 1. ใส่ .env ใน .gitignore เสมอ
echo ".env" >> .gitignore

# 2. ใช้ environment variables แทน hardcode
# ❌ BAD:
API_KEY = "sk-abc123xyz"

# ✅ GOOD:
import os
API_KEY = os.environ.get('API_KEY')

# 3. Scan หา secrets ก่อน commit
# ใช้ git-secrets หรือ gitleaks
brew install gitleaks
gitleaks detect --source . --verbose

# 4. ถ้า commit secrets ไปแล้ว:
# a. เปลี่ยน secret ทันที!
# b. ลบออกจาก history ด้วย git filter-repo
pip install git-filter-repo
git filter-repo --path credentials.json --invert-paths
# c. Force push (แจ้งทีมด้วย!)
```

### 13.4 Workflow Best Practices

```bash
# ✅ สร้าง .gitignore ก่อนเสมอ
echo "node_modules/" > .gitignore
git add .gitignore
git commit -m "chore: add gitignore"

# ✅ Pull ก่อนเสมอ
git pull origin main
# ก่อนจะสร้าง branch ใหม่

# ✅ Test ก่อน commit
npm test
git commit -m "feat: add feature"

# ✅ Review changes ก่อน commit
git diff --staged
git commit

# ✅ ใช้ git status บ่อยๆ
git status  # ดูสถานะก่อนทำอะไร
```

---

## 14. แบบฝึกหัด

### แบบฝึกหัดที่ 1: ติดตั้งและตั้งค่า Git

```
โจทย์:
1. ติดตั้ง Git บนเครื่องของคุณ
2. ตั้งค่า user.name และ user.email
3. ตั้งค่า default editor เป็น VS Code
4. สร้าง alias สำหรับ: status, log (แบบ graph), checkout
5. แสดง git config --list ทั้งหมด

ตรวจสอบ:
$ git --version     (ต้องมี version ออกมา)
$ git config --list (ต้องมี user.name และ user.email)
```

**เฉลย:**
```bash
# 1. ติดตั้ง (macOS)
brew install git

# 2. ตั้งค่า identity
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# 3. ตั้งค่า editor
git config --global core.editor "code --wait"

# 4. สร้าง aliases
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.co checkout

# 5. ดู config
git config --list
```

---

### แบบฝึกหัดที่ 2: สร้าง Repository แรก

```
โจทย์:
1. สร้าง directory ชื่อ "my-first-repo"
2. Initialize git repository
3. สร้างไฟล์ README.md พร้อม content
4. Add ไฟล์ไปยัง staging
5. Commit ด้วย message ที่เหมาะสม
6. แสดง git log

เฉลย:
```bash
mkdir my-first-repo
cd my-first-repo
git init

cat > README.md << 'EOF'
# My First Repository

This is my first Git repository.

## About
- Created as part of CI/CD course
- Learning Git fundamentals
EOF

git add README.md
git status  # ตรวจสอบว่า staged
git commit -m "docs: add initial README"
git log
```

---

### แบบฝึกหัดที่ 3: Working with Files

```
โจทย์:
1. ใน repo จากข้อ 2 สร้างไฟล์ 3 ไฟล์: app.py, config.py, .env
2. เพิ่ม .gitignore ที่ ignore .env
3. Add เฉพาะ app.py และ config.py (ไม่ add .env!)
4. Commit
5. ตรวจสอบว่า .env ไม่ถูก track

เฉลย:
```bash
# สร้างไฟล์
echo 'print("Hello World")' > app.py
echo 'DEBUG = True' > config.py
echo 'SECRET_KEY=mysecret123' > .env

# สร้าง .gitignore
echo ".env" > .gitignore

# ตรวจสอบว่า git ignore .env
git status
# ควรเห็น app.py, config.py, .gitignore แต่ไม่เห็น .env

# Add และ commit
git add app.py config.py .gitignore
git commit -m "feat: add initial application files"

# ยืนยัน .env ไม่ถูก track
git status
git ls-files  # แสดงไฟล์ที่ track อยู่
```

---

### แบบฝึกหัดที่ 4: Commit History

```
โจทย์:
1. แก้ไข app.py อย่างน้อย 3 ครั้ง (commit ทุกครั้ง)
2. ใช้ git log --oneline --graph ดู history
3. ใช้ git diff ดูความแตกต่าง
4. ใช้ git show ดู commit เฉพาะ

ตัวอย่าง:
```bash
# Commit 1: เพิ่ม function
cat > app.py << 'EOF'
def greet(name):
    return f"Hello, {name}!"

if __name__ == "__main__":
    print(greet("World"))
EOF
git add app.py
git commit -m "feat: add greet function"

# Commit 2: แก้ไข function
cat > app.py << 'EOF'
def greet(name, language="en"):
    if language == "th":
        return f"สวัสดี, {name}!"
    return f"Hello, {name}!"

if __name__ == "__main__":
    print(greet("World"))
    print(greet("โลก", "th"))
EOF
git add app.py
git commit -m "feat: add Thai language support to greet"

# Commit 3: เพิ่ม test
cat > test_app.py << 'EOF'
from app import greet

def test_greet_english():
    assert greet("World") == "Hello, World!"

def test_greet_thai():
    assert greet("โลก", "th") == "สวัสดี, โลก!"
EOF
git add test_app.py
git commit -m "test: add unit tests for greet function"

# ดู history
git log --oneline --graph
git diff HEAD~2 HEAD app.py
git show HEAD~1
```

---

### แบบฝึกหัดที่ 5: Undoing Changes

```
โจทย์:
1. แก้ไขไฟล์แล้ว discard changes ด้วย git restore
2. Add ไฟล์แล้ว unstage ด้วย git restore --staged
3. Commit แล้ว undo ด้วย git reset --soft
4. Commit แล้ว revert ด้วย git revert

เฉลย:
```bash
# 1. Discard working dir changes
echo "bad code" >> app.py
git status
git restore app.py
git status  # changes should be gone

# 2. Unstage
echo "something" >> README.md
git add README.md
git status   # staged
git restore --staged README.md
git status   # unstaged

# 3. Reset soft
echo "new line" >> app.py
git add app.py
git commit -m "oops: wrong commit"
git log --oneline
git reset --soft HEAD~1
git status   # changes back in staging

# 4. Revert (ปลอดภัยสำหรับ shared commits)
git add app.py
git commit -m "temp: test revert"
git log --oneline
COMMIT_HASH=$(git log --oneline -1 | cut -d' ' -f1)
git revert $COMMIT_HASH --no-edit
git log --oneline
```

---

### แบบฝึกหัดที่ 6: Git Stash Workflow

```
โจทย์:
Simulate สถานการณ์:
1. กำลัง work บน feature ใหม่ (แก้ไขไฟล์แต่ยังไม่ commit)
2. ต้องแก้ urgent bug (stash งาน, แก้ bug, commit)
3. กลับมา work feature ต่อ (pop stash)

เฉลย:
```bash
# 1. เริ่ม work บน feature
echo "# New Feature" >> feature.py
echo "def new_feature(): pass" >> feature.py
git status   # modified files

# 2. Urgent bug มา! Stash งานก่อน
git stash push -m "WIP: new feature implementation"
git stash list

# 3. แก้ bug
echo "# Bug Fix" >> bugfix.py
git add bugfix.py
git commit -m "fix: resolve critical bug"

# 4. กลับมา work feature
git stash pop
git status  # งานเดิมกลับมา
cat feature.py
```

---

### แบบฝึกหัดที่ 7: SSH Key Setup

```
โจทย์:
1. สร้าง SSH key pair
2. เพิ่ม public key ใน GitHub
3. Test connection: ssh -T git@github.com
4. Clone repo ด้วย SSH URL

เฉลย:
```bash
# 1. สร้าง SSH key
ssh-keygen -t ed25519 -C "your@email.com"
# กด Enter ทุก prompt

# 2. ดู public key
cat ~/.ssh/id_ed25519.pub
# Copy output ทั้งหมด

# 3. ไปที่ GitHub:
# Settings → SSH and GPG keys → New SSH key
# Paste public key

# 4. Test
ssh -T git@github.com
# Hi username! You've successfully authenticated...

# 5. Clone ด้วย SSH
git clone git@github.com:username/repo.git
```

---

### แบบฝึกหัดที่ 8: Push และ Pull

```
โจทย์:
1. สร้าง repository ใหม่บน GitHub (ชื่อ "git-practice")
2. Connect local repo กับ remote
3. Push commits ไป GitHub
4. ดู repository บน GitHub
5. แก้ไขไฟล์บน GitHub web editor
6. Pull การเปลี่ยนแปลงลงมา

เฉลย:
```bash
# 1. สร้าง repo บน GitHub ก่อน (ผ่าน UI หรือ gh CLI)
gh repo create git-practice --public

# 2. Connect local และ push
cd my-first-repo
git remote add origin git@github.com:USERNAME/git-practice.git
git push -u origin main

# 3. แก้ไขบน GitHub:
# ไปที่ GitHub → click ไฟล์ → click edit (pencil icon)
# แก้ไข content → commit changes

# 4. Pull มา
git pull origin main
git log --oneline
```

---

### แบบฝึกหัดที่ 9: .gitignore

```
โจทย์:
สร้าง .gitignore สำหรับ Python project ที่ครอบคลุม:
- Python compiled files
- Virtual environment
- Environment files
- Testing coverage
- IDE files
- OS specific files

เฉลย:
```bash
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
pip-wheel-metadata/
share/python-wheels/
*.egg-info/
.installed.cfg
*.egg

# Virtual environment
.env
.venv
env/
venv/
ENV/
env.bak/
venv.bak/

# Environment variables (NEVER commit!)
.env
.env.local
.env.*.local
*.env

# Testing
.pytest_cache/
.coverage
htmlcov/
.tox/
.nox/

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
EOF

git add .gitignore
git commit -m "chore: add comprehensive Python .gitignore"
```

---

### แบบฝึกหัดที่ 10: Commit Message Practice

```
โจทย์:
เขียน commit message ที่ถูกต้องสำหรับ scenarios ต่อไปนี้:

1. เพิ่มฟังก์ชัน user registration
2. แก้ bug ที่ login หน้า crash เมื่อ email เว้นว่าง
3. อัพเดต README documentation
4. Refactor payment code ให้ clean ขึ้น
5. เพิ่ม unit tests สำหรับ user service

เฉลย:
```
1. feat(auth): add user registration endpoint

   Add POST /api/auth/register with email/password validation,
   password hashing, and duplicate email check.
   
   Closes #45

2. fix(auth): handle empty email on login page

   Previously the login form would crash when submitting
   with an empty email field. Added client-side validation
   to show error message instead.
   
   Fixes #123

3. docs: update README with API setup instructions

4. refactor(payment): simplify payment processing logic

   Extract payment validation into separate function,
   remove duplicate code, improve error messages.
   No functional changes.

5. test(user): add unit tests for UserService class

   Add 15 unit tests covering:
   - User creation validation
   - Password hashing
   - Email uniqueness check
   - User retrieval methods
```

---

### แบบฝึกหัดที่ 11: Recovery Practice

```
โจทย์:
Simulate disaster recovery:
1. Accidentally เพิ่ม .env file ใน commit
2. ค้นหาว่า commit hash คืออะไร
3. Remove file จาก history ด้วย git filter-repo
4. Verify ว่าไฟล์หายไปจาก history แล้ว

⚠️ ทำในสภาพแวดล้อม test เท่านั้น!

เฉลย:
```bash
# Setup test scenario
mkdir test-recovery && cd test-recovery
git init

echo "Normal content" > app.py
git add app.py && git commit -m "initial commit"

# สมมติว่าลืม add .env ใน gitignore และ commit ไปแล้ว
echo "SECRET_KEY=super_secret_123" > .env
git add .env  # ❌ ไม่ควรทำ
git commit -m "oops: accidentally committed secrets"

echo "More content" >> app.py
git add app.py && git commit -m "add more content"

# ตรวจสอบว่ามี .env ใน history
git log --all --oneline
git show HEAD~1:.env   # จะเห็น secret!

# แก้ไข: ใช้ git filter-repo
pip install git-filter-repo
git filter-repo --path .env --invert-paths

# Verify
git log --all --oneline
git show HEAD~1:.env   # ควร error (ไม่มีแล้ว)

# ⚠️ ถ้า push ไปแล้ว: ต้อง force push + แจ้ง team!
# และ rotate secrets ทันที!
```

---

## สรุปบทที่ 2

```
คำสั่งสำคัญที่ต้องจำ:

การเริ่มต้น:
git init                      ← สร้าง repo ใหม่
git clone <url>               ← clone จาก remote

การ track changes:
git status                    ← ดูสถานะ
git add <file>                ← stage file
git commit -m "message"       ← commit
git diff                      ← ดู unstaged changes
git diff --staged             ← ดู staged changes

การทำงานกับ remote:
git push origin main          ← push ไป remote
git pull                      ← pull จาก remote
git fetch                     ← fetch (ไม่ merge)
git remote -v                 ← ดู remotes

การย้อนกลับ:
git restore <file>            ← discard working dir changes
git restore --staged <file>   ← unstage
git reset --soft HEAD~1       ← undo commit (keep changes)
git revert <hash>             ← safe undo (สร้าง commit ใหม่)

ประวัติ:
git log                       ← ดู history
git log --oneline --graph     ← pretty history
git show <hash>               ← ดู commit เฉพาะ

อื่นๆ:
git stash                     ← เก็บงานชั่วคราว
git stash pop                 ← เอางานกลับมา
```

---

## บทต่อไป

**Part 03: Git Branching Strategies** - เรียนรู้การทำงานกับ branches และ strategies ต่างๆ เช่น Git Flow, GitHub Flow, Trunk-based Development

---

## แหล่งเรียนรู้เพิ่มเติม

```
Interactive Learning:
- Learn Git Branching: https://learngitbranching.js.org
- Git Immersion: https://gitimmersion.com

เอกสารอ้างอิง:
- Git Official Docs: https://git-scm.com/doc
- Pro Git Book (ฟรี): https://git-scm.com/book/en/v2
- GitHub Docs: https://docs.github.com

Cheat Sheets:
- GitHub Git Cheat Sheet: https://education.github.com/git-cheat-sheet-education.pdf
- Atlassian Git Cheat Sheet: https://www.atlassian.com/git/tutorials/atlassian-git-cheatsheet

เครื่องมือ:
- GitHub CLI: https://cli.github.com
- GitLens (VS Code): https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens
```

---

*ยินดีด้วย! คุณรู้จัก Git fundamentals ครบแล้ว! ตอนนี้เราพร้อมเรียน Branching Strategies* 🎉
