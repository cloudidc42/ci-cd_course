# Part 03: Git Branching Strategies

> **ระดับ:** เริ่มต้น-กลาง (Beginner-Intermediate)
> **เวลาที่ใช้:** 5-6 ชั่วโมง
> **ข้อกำหนดเบื้องต้น:** Part 02 - Git Fundamentals

---

## สารบัญ

1. [Branches คืออะไร?](#1-branches-คืออะไร)
2. [คำสั่ง Branch พื้นฐาน](#2-คำสั่ง-branch-พื้นฐาน)
3. [Merging Branches](#3-merging-branches)
4. [Git Rebase](#4-git-rebase)
5. [Conflict Resolution](#5-conflict-resolution)
6. [Cherry-Pick](#6-cherry-pick)
7. [Tagging](#7-tagging)
8. [Git Flow Workflow](#8-git-flow-workflow)
9. [GitHub Flow](#9-github-flow)
10. [GitLab Flow](#10-gitlab-flow)
11. [Trunk-Based Development](#11-trunk-based-development)
12. [Branch Protection Rules](#12-branch-protection-rules)
13. [Pull Request Workflow](#13-pull-request-workflow)
14. [Code Review Best Practices](#14-code-review-best-practices)
15. [แบบฝึกหัด](#15-แบบฝึกหัด)

---

## 1. Branches คืออะไร?

### 1.1 แนวคิด Branch

**Branch** คือ "ทางแยก" จาก timeline หลักของ code ที่ให้คุณทำงานแบบ isolated โดยไม่กระทบ code หลัก

```
ภาพ concept ของ branch:

main:     ─── A ─── B ─── C ─── D ───────────── G ─── H
                              \                /
feature:                       E ─── F ─────/
                               (isolated work)
```

**ทำไมต้องใช้ branches?**
```
ปัญหาถ้าไม่มี branches:
├── ทุกคน commit ไปยัง main โดยตรง
├── code ที่ยังไม่เสร็จปน production code
├── ยาก test feature ก่อน release
└── ถ้า feature พัง → main พัง → production พัง!

ด้วย branches:
├── แต่ละ feature/bug อยู่ใน branch ของตัวเอง
├── ทดสอบ isolated ก่อน merge
├── ถ้า feature พัง → main ไม่กระทบ
└── หลาย features พัฒนาพร้อมกันได้
```

### 1.2 Branch ใน Git เป็นแค่ Pointer

```bash
# Branch ใน Git คือ lightweight pointer ไปยัง commit
# ไม่ได้ copy files ทั้งหมด!

# ดู branches
cat .git/refs/heads/main
# a1b2c3d4e5f6... (SHA-1 hash ของ commit)

cat .git/refs/heads/feature/login
# e7f8g9h0... (SHA-1 hash ของ commit อื่น)

# HEAD ชี้ไปยัง branch ปัจจุบัน
cat .git/HEAD
# ref: refs/heads/main
```

### 1.3 Branch ทำงานอย่างไร?

```
สร้าง branch ใหม่:

ก่อน:
main:    ─── A ─── B ─── C  ← HEAD (อยู่ที่ main)

หลัง git checkout -b feature:
main:    ─── A ─── B ─── C  ← main pointer
feature: ─── A ─── B ─── C  ← feature pointer, HEAD

เมื่อ commit บน feature:
main:    ─── A ─── B ─── C
feature:               \─── D ─── E  ← HEAD

เมื่อ merge กลับ:
main:    ─── A ─── B ─── C ─────────── F  ← HEAD (merge commit)
feature:               \─── D ─── E ─/
```

---

## 2. คำสั่ง Branch พื้นฐาน

### 2.1 ดู Branches

```bash
# ดู local branches
git branch

# ดู local branches พร้อม commit ล่าสุด
git branch -v

# ดู remote branches
git branch -r

# ดูทั้ง local และ remote
git branch -a

# ดู branches ที่ merge แล้ว
git branch --merged

# ดู branches ที่ยังไม่ merge
git branch --no-merged

# ตัวอย่าง output:
# * main          a1b2c3d Add README
#   develop       e4f5g6h Setup project
#   feature/login i7j8k9l Add login form
```

### 2.2 สร้าง Branch

```bash
# สร้าง branch ใหม่ (ยังอยู่ที่ branch เดิม)
git branch feature/user-login

# สร้างและ switch ไปพร้อมกัน (วิธีเก่า)
git checkout -b feature/user-login

# สร้างและ switch (วิธีใหม่ - Git 2.23+)
git switch -c feature/user-login

# สร้าง branch จาก commit เฉพาะ
git branch feature/login abc1234

# สร้าง branch จาก tag
git branch hotfix/v1.2.1 v1.2.0

# สร้าง branch จาก remote branch
git checkout -b feature/login origin/feature/login
git switch -c feature/login --track origin/feature/login
```

### 2.3 Switch Branches

```bash
# Switch ไป branch ที่มีอยู่
git checkout main
git switch main          # วิธีใหม่

# Switch กลับไป branch ก่อนหน้า
git checkout -
git switch -             # วิธีใหม่

# ตัวอย่าง:
git switch main
git switch feature/login
git switch -             # กลับไป feature/login
git switch -             # กลับไป main
```

### 2.4 ลบ Branch

```bash
# ลบ local branch (ที่ merge แล้ว)
git branch -d feature/completed-feature

# Force delete branch (แม้ยังไม่ merge)
git branch -D feature/abandoned-feature

# ลบ remote branch
git push origin --delete feature/old-feature

# หรือ
git push origin :feature/old-feature

# ลบ remote tracking references ที่ไม่มีแล้ว
git remote prune origin

# ดูและลบ branches ที่ merge แล้วทั้งหมด
git branch --merged | grep -v "^\*" | grep -v "main\|develop" | xargs git branch -d
```

### 2.5 Rename Branch

```bash
# Rename branch ปัจจุบัน
git branch -m new-name

# Rename branch อื่น
git branch -m old-name new-name

# Rename main branch (ระวัง! แจ้งทีมก่อน)
git branch -m master main
git push origin main
git push origin --delete master
git push origin -u main
```

---

## 3. Merging Branches

### 3.1 git merge คืออะไร?

Merge คือการนำ changes จาก branch หนึ่งมารวมกับอีก branch หนึ่ง

### 3.2 ประเภทของ Merge

#### Fast-Forward Merge

```bash
# เกิดขึ้นเมื่อ main ไม่มี commits ใหม่หลังจาก branch แตกออก

ก่อน merge:
main:    ─── A ─── B
feature:           \─── C ─── D  ← HEAD

git checkout main
git merge feature

หลัง merge (fast-forward):
main:    ─── A ─── B ─── C ─── D  ← HEAD (แค่ move pointer)
feature:                      ─── D
# ไม่มี merge commit สร้างขึ้น
```

```bash
# ทดสอบ fast-forward merge
git checkout main
git merge feature/no-conflict
# Fast-forward
# index.html | 2 +-
# 1 file changed, 1 insertion(+), 1 deletion(-)
```

#### 3-Way Merge (Recursive Merge)

```bash
# เกิดขึ้นเมื่อ main มี commits ใหม่หลังจาก branch แตกออก

ก่อน merge:
main:    ─── A ─── B ─── E  ← main
feature:           \─── C ─── D  ← feature

git checkout main
git merge feature

หลัง merge (creates merge commit):
main:    ─── A ─── B ─── E ─── F  ← HEAD (F = merge commit)
                     \        /
feature:              C ─── D
```

```bash
# ทดสอบ 3-way merge
git checkout main
git merge feature/with-divergent
# Merge made by the 'recursive' strategy.
```

### 3.3 Merge Options

```bash
# Merge แบบปกติ
git merge feature/login

# No fast-forward (สร้าง merge commit เสมอ)
git merge --no-ff feature/login
# เหมาะสำหรับ feature branches เพื่อเห็น merge history ชัด

# Squash merge (รวม commits เป็น 1 ก่อน merge)
git merge --squash feature/login
git commit -m "feat: add user login feature"
# เหมาะเมื่อ feature branch มี commits ที่ไม่สะอาด

# ยกเลิก merge ที่กำลัง conflict
git merge --abort

# Merge เฉพาะ file
git checkout main
git checkout feature/login -- specific-file.txt
```

### 3.4 Merge Strategy

```bash
# ดู merge strategies
git merge --strategy=ours feature/old
git merge --strategy=recursive -X ours feature/login

# Strategy options:
# ours       = ถ้า conflict ใช้ version ของเรา
# theirs     = ถ้า conflict ใช้ version ของพวกเขา
# patience   = ใช้ patience algorithm (better conflicts)
# recursive  = default สำหรับ 3-way merge
```

---

## 4. Git Rebase

### 4.1 Rebase คืออะไร?

Rebase คือการ "ย้าย" base ของ branch ไปยังจุดใหม่

```
ก่อน rebase:
main:    ─── A ─── B ─── E ─── F  ← main
feature:           \─── C ─── D  ← feature

หลัง git rebase main (รัน บน feature branch):
main:    ─── A ─── B ─── E ─── F  ← main
feature:                   \─── C' ─── D'  ← feature
(C' และ D' คือ commits ใหม่ที่ apply บน F)
```

### 4.2 คำสั่ง Rebase

```bash
# Rebase feature branch บน main
git checkout feature/login
git rebase main

# หรือในบรรทัดเดียว
git rebase main feature/login

# Interactive rebase (แก้ไข commit history)
git rebase -i HEAD~3    # แก้ไข 3 commits ล่าสุด
git rebase -i abc1234   # แก้ไข commits หลังจาก abc1234

# Interactive rebase options:
# pick   = ใช้ commit นี้
# reword = แก้ commit message
# edit   = หยุดเพื่อ amend
# squash = รวมกับ commit ก่อน
# fixup  = รวมกับ commit ก่อน (ทิ้ง message)
# drop   = ลบ commit นี้
```

### 4.3 Interactive Rebase ตัวอย่าง

```bash
# Scenario: มี commits ที่ dirty อยู่
git log --oneline
# abc1234 WIP: working on login
# def5678 fix typo
# ghi9012 fix another typo
# jkl0123 feat: add login form

# Interactive rebase เพื่อ squash commits
git rebase -i HEAD~4

# Editor จะเปิดขึ้น:
# pick jkl0123 feat: add login form
# pick ghi9012 fix another typo
# pick def5678 fix typo
# pick abc1234 WIP: working on login

# แก้เป็น:
# pick jkl0123 feat: add login form
# squash ghi9012 fix another typo
# squash def5678 fix typo
# squash abc1234 WIP: working on login

# บันทึก → เขียน commit message ใหม่:
# feat: add complete login form with validation
#
# - Email and password fields
# - Client-side validation
# - Error message display

# ผลลัพธ์: commits 4 อันกลายเป็น 1 clean commit
```

### 4.4 Merge vs Rebase

```
                MERGE                    REBASE
─────────────────────────────────────────────────
History     ยัง historical            Clean, linear
Merge commit มี merge commit          ไม่มี
Safe        ปลอดภัยกว่า (non-destructive) เขียน history ใหม่
Use when    shared branches           personal branches
PR style    GitHub default            GitLab preference

Rule of thumb:
- "Never rebase on shared/public branches"
- Rebase เฉพาะ feature branches ที่ local ยังไม่ push
- ถ้า push ไปแล้วและทีมอื่น pull แล้ว → ใช้ merge
```

```bash
# ตัวอย่าง workflow ที่ดี
# 1. Work บน feature branch
git checkout -b feature/login

# 2. Commit งาน (บ่อยๆ แม้ message ไม่สะอาด)
git commit -m "WIP login form"
git commit -m "fix form validation"
git commit -m "add error handling"

# 3. ก่อน push หรือ PR → rebase เพื่อ cleanup
git rebase -i HEAD~3
# squash เป็น 1 clean commit

# 4. Rebase กับ main ล่าสุดเพื่อ resolve conflicts ตอนนี้
git rebase main

# 5. Push
git push origin feature/login

# 6. สร้าง PR → reviewer merge ด้วย --no-ff
```

---

## 5. Conflict Resolution

### 5.1 Conflict คืออะไร?

Conflict เกิดขึ้นเมื่อ Git ไม่สามารถ merge changes โดยอัตโนมัติได้ เพราะทั้งสอง branch แก้ไขบรรทัดเดียวกันต่างกัน

```
Scenario ที่ทำให้เกิด conflict:

main branch:      feature branch:
line 10: "red"    line 10: "blue"

Git ไม่รู้ว่าจะใช้ "red" หรือ "blue"
→ CONFLICT!
```

### 5.2 สถานการณ์ที่ทำให้เกิด Conflict

```bash
# 1. Merge conflict
git merge feature/login
# Auto-merging login.py
# CONFLICT (content): Merge conflict in login.py
# Automatic merge failed; fix conflicts and then commit the result.

# 2. Rebase conflict
git rebase main
# CONFLICT (content): Merge conflict in app.py
# error: could not apply abc1234... feat: update login

# 3. Stash pop conflict
git stash pop
# CONFLICT (content): Merge conflict in config.py
```

### 5.3 อ่าน Conflict Markers

```python
# เมื่อมี conflict Git จะใส่ markers ในไฟล์:

<<<<<<< HEAD
def login(email, password):
    # Current branch version
    user = db.find_user(email)
    if user and check_password(password, user.password):
        return create_token(user)
=======
def login(email, password):
    # Incoming branch version (feature/new-auth)
    user = User.query.filter_by(email=email).first()
    if user and user.check_password(password):
        return user.generate_token()
>>>>>>> feature/new-auth

# Marker อธิบาย:
# <<<<<<< HEAD = เริ่ม current branch changes
# ======= = แบ่งระหว่าง two versions
# >>>>>>> feature/new-auth = สิ้นสุด incoming changes
```

### 5.4 Resolve Conflict Step-by-Step

```bash
# Step 1: ดูไฟล์ที่มี conflict
git status
# Unmerged paths:
#   both modified:   login.py
#   both modified:   database.py

# Step 2: เปิดไฟล์และแก้ไข
# วิธีที่ 1: แก้ด้วย text editor (เลือกว่าจะใช้ version ไหน)

# วิธีที่ 2: ใช้ git mergetool
git mergetool
# เปิด tool เช่น vimdiff, meld, VS Code

# วิธีที่ 3: ใช้ VS Code (แนะนำ)
code login.py
# VS Code จะแสดง conflict inline พร้อม buttons:
# "Accept Current Change" = ใช้ HEAD version
# "Accept Incoming Change" = ใช้ theirs version
# "Accept Both Changes" = ใช้ทั้งสอง
# "Compare Changes" = เปรียบเทียบ

# Step 3: หลังแก้ไขแล้ว ลบ conflict markers ให้หมด
# ตรวจสอบว่าไม่มี <<<<<<< ======= >>>>>>> เหลืออยู่

# Step 4: Test ว่า code ยัง work
python -m pytest

# Step 5: Stage ไฟล์ที่แก้แล้ว
git add login.py
git add database.py

# Step 6: Commit (สำหรับ merge) หรือ continue (สำหรับ rebase)
# สำหรับ merge:
git commit
# สำหรับ rebase:
git rebase --continue
```

### 5.5 ตัวอย่าง Conflict Resolution จริง

```python
# login.py ก่อนแก้ (มี conflict markers)

<<<<<<< HEAD
def authenticate_user(email, password):
    """Authenticate user with email and password."""
    user = UserRepository.find_by_email(email)
    if not user:
        raise AuthenticationError("User not found")
    if not bcrypt.verify(password, user.password_hash):
        raise AuthenticationError("Invalid password")
    return generate_jwt_token(user.id)
=======
def authenticate_user(email: str, password: str) -> str:
    """Authenticate user and return access token."""
    user = User.query.filter_by(email=email).first()
    if not user or not user.verify_password(password):
        raise InvalidCredentialsError("Invalid email or password")
    return Token.create(user_id=user.id, expires_in=3600)
>>>>>>> feature/auth-refactor
```

```python
# login.py หลังแก้ (เลือก merge ทั้งสองแบบ)

def authenticate_user(email: str, password: str) -> str:
    """Authenticate user with email and password.
    
    Returns:
        JWT access token string
    Raises:
        AuthenticationError: If credentials are invalid
    """
    user = UserRepository.find_by_email(email)
    if not user:
        raise AuthenticationError("User not found")
    if not user.verify_password(password):
        raise AuthenticationError("Invalid password")
    return Token.create(user_id=user.id, expires_in=3600)
```

### 5.6 ยกเลิก Merge เมื่อมีปัญหา

```bash
# ยกเลิก merge ที่กำลัง conflict
git merge --abort

# ยกเลิก rebase ที่กำลัง conflict
git rebase --abort

# ยกเลิก cherry-pick ที่กำลัง conflict
git cherry-pick --abort

# Reset กลับก่อน merge
git reset --hard ORIG_HEAD
```

### 5.7 เครื่องมือ Merge ที่แนะนำ

```bash
# ตั้งค่า VS Code เป็น merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# ตั้งค่า vimdiff
git config --global merge.tool vimdiff

# ตั้งค่า meld (Linux)
sudo apt install meld
git config --global merge.tool meld

# เปิด merge tool
git mergetool
```

---

## 6. Cherry-Pick

### 6.1 Cherry-Pick คืออะไร?

Cherry-pick คือการเอา commit เฉพาะจาก branch หนึ่งมาใส่ใน branch ปัจจุบัน โดยไม่ merge ทั้ง branch

```
Use case สำคัญ:

main:    ─── A ─── B ─── C ─── F  ← main
hotfix:            \─── D ─── E  ← hotfix (D=bug fix, E=cleanup)

ต้องการ: เอาเฉพาะ D (bug fix) มา main แต่ไม่เอา E
git checkout main
git cherry-pick D

main:    ─── A ─── B ─── C ─── D' ─── F  ← main (D' = copy of D)
```

### 6.2 คำสั่ง Cherry-Pick

```bash
# Cherry-pick commit เดียว
git cherry-pick abc1234

# Cherry-pick หลาย commits
git cherry-pick abc1234 def5678 ghi9012

# Cherry-pick range ของ commits (A..B = จาก A ถึง B, ไม่รวม A)
git cherry-pick A..B

# Cherry-pick range รวม A
git cherry-pick A^..B

# Cherry-pick โดยไม่ commit ทันที (stage เท่านั้น)
git cherry-pick --no-commit abc1234
git cherry-pick -n abc1234

# Cherry-pick และแก้ commit message
git cherry-pick --edit abc1234

# ดู commits ที่ cherry-pick ไปแล้ว
git log --cherry-mark main...feature
```

### 6.3 Cherry-Pick Scenarios

```bash
# Scenario 1: Bug fix ที่ต้องใส่ใน multiple releases

# สมมติ fix อยู่ใน develop
git log develop --oneline
# abc1234 hotfix: fix payment calculation bug ← ต้องการ commit นี้
# def5678 feat: new dashboard
# ghi9012 feat: user profiles

# Cherry-pick ไป main
git checkout main
git cherry-pick abc1234

# Cherry-pick ไป release/v2 ด้วย
git checkout release/v2
git cherry-pick abc1234

# Scenario 2: เอา feature บางส่วนมาทดสอบ
git checkout staging
git cherry-pick feature/partial-implementation..feature/login
```

### 6.4 แก้ Conflict ใน Cherry-Pick

```bash
# ถ้า conflict เกิดขึ้น
git cherry-pick abc1234
# CONFLICT (content): Merge conflict in app.py

# แก้ conflict ในไฟล์
vim app.py

# Stage และ continue
git add app.py
git cherry-pick --continue

# หรือยกเลิก
git cherry-pick --abort
```

---

## 7. Tagging

### 7.1 Tag คืออะไร?

Tag คือ bookmark ที่ชี้ไปยัง commit เฉพาะ มักใช้สำหรับ marking release versions

```
Commits:  ─── A ─── B ─── C ─── D ─── E ─── F
Tags:                 v1.0         v1.1   v2.0
```

### 7.2 ประเภทของ Tag

```bash
# 1. Lightweight Tag (แค่ pointer ไป commit)
git tag v1.0.0

# 2. Annotated Tag (มี metadata - แนะนำ)
git tag -a v1.0.0 -m "Release version 1.0.0"
git tag -a v1.0.0 -m "Release version 1.0.0

## What's new:
- User authentication
- Payment processing
- Admin dashboard

## Bug fixes:
- Fix login timeout issue
- Correct tax calculation"
```

### 7.3 คำสั่ง Tag

```bash
# ดู tags ทั้งหมด
git tag
git tag -l           # list format

# ดู tags ที่ match pattern
git tag -l "v1.*"    # v1.0.0, v1.1.0, v1.2.3...

# สร้าง tag บน commit เฉพาะ
git tag -a v1.0.0 abc1234 -m "Release v1.0.0"

# ดูข้อมูล tag
git show v1.0.0

# Push tag ไป remote (tags ไม่ได้ push อัตโนมัติ!)
git push origin v1.0.0

# Push tags ทั้งหมด
git push origin --tags

# ลบ tag local
git tag -d v1.0.0

# ลบ tag remote
git push origin --delete v1.0.0
git push origin :refs/tags/v1.0.0

# Checkout ไปยัง tag
git checkout v1.0.0
# ⚠️ ทำให้อยู่ใน "detached HEAD" state

# สร้าง branch จาก tag
git checkout -b release/hotfix v1.0.0
```

### 7.4 Semantic Versioning

```
Standard: MAJOR.MINOR.PATCH

MAJOR = breaking changes (v1.x.x → v2.0.0)
MINOR = new features, backward compatible (v1.1.x → v1.2.0)
PATCH = bug fixes (v1.1.1 → v1.1.2)

ตัวอย่าง:
v1.0.0  ← Initial release
v1.0.1  ← Bug fix
v1.1.0  ← New feature added
v1.2.0  ← Another new feature
v2.0.0  ← Breaking change (API changed)
v2.0.1  ← Bug fix

Pre-release:
v2.0.0-alpha.1
v2.0.0-beta.1
v2.0.0-rc.1    ← Release Candidate
v2.0.0         ← Final release
```

---

## 8. Git Flow Workflow

### 8.1 Git Flow คืออะไร?

Git Flow เป็น branching strategy ที่นิยมใช้ในทีมขนาดกลาง-ใหญ่ที่มี release cycles ชัดเจน

เขียนโดย Vincent Driessen ในปี 2010

### 8.2 Branches ใน Git Flow

```
Git Flow Branches:

main (หรือ master)
├── เก็บ production-ready code เท่านั้น
├── ทุก commit = tagged release
└── ห้าม commit ตรงๆ (ต้องมาจาก release หรือ hotfix เท่านั้น)

develop
├── Integration branch
├── ทีมรวม features ที่เสร็จแล้วที่นี่
└── เป็น default branch สำหรับ development

feature/*
├── แตกจาก develop
├── merge กลับไป develop เมื่อเสร็จ
└── ตัวอย่าง: feature/user-auth, feature/payment

release/*
├── แตกจาก develop เมื่อพร้อม release
├── เฉพาะ bug fixes เท่านั้น
├── merge ไป main และ develop
└── ตัวอย่าง: release/1.2.0

hotfix/*
├── แตกจาก main (emergency fixes!)
├── merge ไป main และ develop
└── ตัวอย่าง: hotfix/payment-crash
```

### 8.3 Git Flow Diagram

```
main:    ─────────────────────────────── v1.0 ─────── v1.0.1 ── v1.1
                                            ↑                 ↑      ↑
release:                           1.0 ────/   hotfix: 1.0.1─/      /
                                  ↑                                 /
develop: ──────── ─────────────── ─────────────────────────────────
              ↑    ↓          ↑    ↑
feature:  feat/A ─/    feat/B ─────/
```

### 8.4 Git Flow ทำงานจริง

```bash
# ===============================
# ติดตั้ง git-flow tool (optional)
# ===============================
# macOS
brew install git-flow-avh

# Ubuntu
sudo apt install git-flow

# เริ่มต้น git flow ใน repo
git flow init
# ? Branch name for production releases: main
# ? Branch name for "next release": develop
# ? Feature branches? feature/
# ? Bugfix branches? bugfix/
# ? Release branches? release/
# ? Hotfix branches? hotfix/
# ? Support branches? support/
# ? Version tag prefix? v

# ===============================
# Feature Flow
# ===============================

# เริ่ม feature ใหม่
git flow feature start user-authentication
# หรือ manual:
git checkout develop
git checkout -b feature/user-authentication

# ทำงาน...
git add .
git commit -m "feat: add user login"
git add .
git commit -m "feat: add user logout"

# เสร็จ feature
git flow feature finish user-authentication
# หรือ manual:
git checkout develop
git merge --no-ff feature/user-authentication
git branch -d feature/user-authentication
git push origin develop

# ===============================
# Release Flow
# ===============================

# เริ่ม release
git flow release start 1.2.0
# หรือ manual:
git checkout develop
git checkout -b release/1.2.0

# แก้ bug เล็กๆ / อัพเดต version numbers
echo "1.2.0" > version.txt
git add version.txt
git commit -m "chore: bump version to 1.2.0"

# เสร็จ release
git flow release finish 1.2.0
# หรือ manual:
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release v1.2.0"
git checkout develop
git merge --no-ff release/1.2.0
git branch -d release/1.2.0
git push origin main develop --tags

# ===============================
# Hotfix Flow
# ===============================

# Production มี bug ด่วน!
git flow hotfix start fix-payment-crash
# หรือ manual:
git checkout main
git checkout -b hotfix/fix-payment-crash

# แก้ bug
git add .
git commit -m "fix: resolve crash in payment processing"

# Deploy hotfix
git flow hotfix finish fix-payment-crash
# หรือ manual:
git checkout main
git merge --no-ff hotfix/fix-payment-crash
git tag -a v1.2.1 -m "Hotfix v1.2.1"
git checkout develop
git merge --no-ff hotfix/fix-payment-crash
git branch -d hotfix/fix-payment-crash
git push origin main develop --tags
```

### 8.5 ข้อดี-ข้อเสีย Git Flow

```
ข้อดี:
✅ เหมาะกับ scheduled releases (version-based)
✅ แยก environments ชัดเจน
✅ History ชัดเจนมาก
✅ Hotfix ไม่กระทบ features ที่กำลังพัฒนา

ข้อเสีย:
❌ ซับซ้อนเกินไปสำหรับ continuous delivery
❌ Long-lived branches = more conflicts
❌ Slow feedback loop
❌ ไม่เหมาะกับ web apps ที่ deploy บ่อย
```

---

## 9. GitHub Flow

### 9.1 GitHub Flow คืออะไร?

GitHub Flow เป็น workflow ที่ง่ายกว่า Git Flow เหมาะกับทีมที่ทำ continuous delivery

Scott Chacon (GitHub) เขียน blog post อธิบาย GitHub Flow ในปี 2011

### 9.2 GitHub Flow Rules

```
กฎ 6 ข้อของ GitHub Flow:

1. ทุกอย่างใน main branch สามารถ deploy ได้เสมอ
2. ทำงานใหม่ต้องสร้าง branch จาก main
3. Push ไป remote branch บ่อยๆ
4. เมื่อต้องการ feedback หรือ merge → เปิด Pull Request
5. หลัง review และ approved → merge ไป main
6. เมื่อ merge ไป main แล้ว → deploy ทันที
```

### 9.3 GitHub Flow Diagram

```
main: ─────── A ──────────────── M ────────────── N
                  ↑             ↑                  ↑
feature-1:    B ─ C ─ D ───────/                   │
feature-2:              E ─── F ─ G ───────────────/
```

### 9.4 GitHub Flow ในทางปฏิบัติ

```bash
# 1. สร้าง feature branch จาก main (ที่ fresh)
git checkout main
git pull origin main
git checkout -b feature/add-search-functionality

# 2. ทำงาน และ push บ่อยๆ
git add .
git commit -m "feat: add search input component"
git push origin feature/add-search-functionality

git add .
git commit -m "feat: implement search API integration"
git push origin feature/add-search-functionality

# 3. เปิด Pull Request ใน GitHub
# - Title: "Add search functionality"  
# - Description: อธิบาย changes และ screenshots
# - Assign reviewers
# - Link related issues: "Closes #45"

# 4. Discussion, review, request changes
# ถ้ามีการขอ changes:
git add .
git commit -m "fix: address review comments - improve error handling"
git push origin feature/add-search-functionality

# 5. หลัง approved → merge
# GitHub จะ merge ให้ผ่าน button "Merge pull request"

# 6. Deploy และ monitor

# 7. ลบ branch
git push origin --delete feature/add-search-functionality
git branch -d feature/add-search-functionality
```

### 9.5 ข้อดี-ข้อเสีย GitHub Flow

```
ข้อดี:
✅ Simple มาก (main + feature branches เท่านั้น)
✅ Continuous delivery friendly
✅ Fast feedback loop
✅ เหมาะกับทีมเล็ก-กลาง

ข้อเสีย:
❌ ต้องการ robust CI/CD และ automated testing
❌ ถ้า main มี bugs → production มี bugs
❌ ไม่มี staging branch ชัดเจน
❌ ยากสำหรับหลาย versions production
```

---

## 10. GitLab Flow

### 10.1 GitLab Flow คืออะไร?

GitLab Flow เป็น middle ground ระหว่าง Git Flow (ซับซ้อน) และ GitHub Flow (ง่ายเกินไป)

### 10.2 GitLab Flow with Environment Branches

```
สำหรับทีมที่มีหลาย environments:

main → staging → production

development:                                          
main:        ─── A ─── B ─── C ─── D ─── E  ← ทุก feature merge มาที่นี่
                              ↓
staging:                      C ─── D ─── E  ← auto-deploy จาก main
                                           ↓
production:                                E  ← manual promote
```

```bash
# GitLab Flow ด้วย environment branches

# 1. ทำงานบน feature branch
git checkout -b feature/user-profile main

# 2. Merge ไป main (via MR)
# → Auto-deploy ไป staging

# 3. Test บน staging

# 4. Promote ไป production (merge staging → production)
git checkout production
git merge staging
git push origin production
# → Deploy ไป production
```

### 10.3 GitLab Flow with Release Branches

```
สำหรับ mobile apps หรือ software ที่มีหลาย versions:

main: ─── A ─── B ─── C ─── D ─── E ─── F
                   \            \
release/2.3:        C ─── D'               ← v2.3.x releases
release/2.4:                    E ─── F'   ← v2.4.x releases
```

---

## 11. Trunk-Based Development

### 11.1 Trunk-Based Development คืออะไร?

ทุก developer commit ไปยัง main branch (trunk) ทุกวัน หรืออย่างน้อยทุก 2 วัน

### 11.2 ลักษณะเด่น

```
Core Principles:
├── Main branch ต้อง always be in deployable state
├── Feature flags ควบคุม feature visibility
├── Short-lived branches (< 2 วัน)
├── Continuous integration เป็น requirement
└── Automated testing coverage สูง (> 80%)
```

### 11.3 Trunk-Based Development Flow

```bash
# Small team (2-5 คน): Direct commits to main
git checkout main
git pull
# แก้ไขเล็กๆ...
git add .
git commit -m "feat: add email validation"
git push

# Large team: Short-lived feature branches
git checkout main
git pull
git checkout -b feature/payment-v2  # ← branch อายุสั้น!

# ทำงานเร็วๆ ใน 1-2 วัน
git commit -m "feat: add new payment processing logic"

# Merge กลับทันที ไม่รอ
git checkout main
git pull
git merge feature/payment-v2
git push
git branch -d feature/payment-v2
```

### 11.4 Feature Flags

Feature flags (หรือ feature toggles) ทำให้เราสามารถ merge code ที่ยังไม่เสร็จเข้า main โดยไม่กระทบ users

```python
# config/features.py
FEATURE_FLAGS = {
    "new_payment_ui": os.getenv("FEATURE_NEW_PAYMENT_UI", "false") == "true",
    "ai_recommendations": os.getenv("FEATURE_AI_RECOMMENDATIONS", "false") == "true",
    "dark_mode": True,  # everyone gets dark mode
}

# app/routes.py
from config.features import FEATURE_FLAGS

@app.route("/payment")
def payment():
    if FEATURE_FLAGS["new_payment_ui"]:
        return render_template("payment_v2.html")
    return render_template("payment.html")
```

```yaml
# Environment-based feature flags
# staging environment
FEATURE_NEW_PAYMENT_UI=true
FEATURE_AI_RECOMMENDATIONS=true

# production environment
FEATURE_NEW_PAYMENT_UI=false    # ← ยังไม่พร้อม
FEATURE_AI_RECOMMENDATIONS=false
```

### 11.5 เปรียบเทียบ 3 Strategies

```
                Git Flow    GitHub Flow    Trunk-Based
────────────────────────────────────────────────────────
Complexity      High        Low            Medium
Branch lifetime Weeks/months Days          Hours/days
Release cycle   Scheduled   Continuous     Continuous
CI/CD maturity  Low-medium  Medium         High
Team size       Large       Small-medium   Any
Feature flags   No          No             Yes
Best for        Enterprise  Web apps       High-freq deploy
                products    startups       elite teams
```

---

## 12. Branch Protection Rules

### 12.1 ทำไมต้องมี Branch Protection?

```
ปัญหาที่เกิดขึ้นโดยไม่มี branch protection:
├── ✗ Force push ไป main ทำให้ history หาย
├── ✗ Merge โดยไม่มี review
├── ✗ Merge code ที่ test fail
├── ✗ Delete main branch โดยไม่ตั้งใจ
└── ✗ Junior developer เผลอ push โดยตรงไป main
```

### 12.2 ตั้งค่า Branch Protection บน GitHub

```
GitHub Repository → Settings → Branches → Add rule

สำหรับ "main" branch, เปิดใช้:

☑ Require a pull request before merging
   ☑ Require approvals (minimum 1-2 reviewers)
   ☑ Dismiss stale reviews when new commits are pushed
   ☑ Require review from code owners

☑ Require status checks to pass before merging
   ☑ Require branches to be up to date
   Add status checks: "CI Pipeline / Build", "CI Pipeline / Test"

☑ Require signed commits (สำหรับ security-conscious teams)

☑ Require linear history (ไม่มี merge commits)

☑ Include administrators (บังคับ rules กับทุกคนรวม admin)

☑ Restrict pushes that create matching branches

✓ Do not allow bypassing the above settings
```

### 12.3 CODEOWNERS File

```bash
# สร้าง CODEOWNERS file
cat > .github/CODEOWNERS << 'EOF'
# Global owners (ทุกไฟล์)
* @tech-lead

# Backend code
/src/api/ @backend-team
/src/database/ @backend-team @dba-team

# Frontend code
/src/frontend/ @frontend-team

# Infrastructure
/terraform/ @devops-team
/kubernetes/ @devops-team
/.github/workflows/ @devops-team

# Security-sensitive files
/src/auth/ @security-team @tech-lead
/src/payment/ @security-team @payment-team

# Documentation
/docs/ @tech-writer
EOF

git add .github/CODEOWNERS
git commit -m "docs: add CODEOWNERS for code review assignments"
```

---

## 13. Pull Request Workflow

### 13.1 Pull Request (PR) คืออะไร?

PR (หรือ Merge Request ใน GitLab) คือการขอ merge code จาก branch หนึ่งเข้าอีก branch หนึ่ง พร้อมกับ discussion และ review

### 13.2 สร้าง Pull Request ที่ดี

**PR Title:**
```
✅ ดี:
feat: add user authentication with JWT
fix: resolve checkout crash on mobile devices
refactor: simplify payment processing module

❌ ไม่ดี:
Update code
Bug fix
WIP
Fix stuff
```

**PR Description Template:**
```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## Summary
<!-- อธิบาย briefly ว่า PR นี้ทำอะไร -->

## Changes
<!-- List การเปลี่ยนแปลงหลัก -->
- 
- 
- 

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Testing
<!-- อธิบายว่า test อะไรบ้าง -->
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing done

## Screenshots
<!-- ถ้ามี UI changes -->

## Related Issues
<!-- Link related issues -->
Closes #

## Checklist
- [ ] My code follows the code style of this project
- [ ] I have added tests that prove my fix is effective
- [ ] New and existing unit tests pass
- [ ] I have updated the documentation accordingly
```

### 13.3 PR Workflow ขั้นตอน

```bash
# ===============================
# นักพัฒนา: สร้าง PR
# ===============================

# 1. อัพเดต main ล่าสุด
git checkout main
git pull origin main

# 2. สร้าง feature branch
git checkout -b feature/add-search-api

# 3. ทำงาน, commit บ่อยๆ
git commit -m "feat: add search query parser"
git commit -m "feat: add search API endpoint"
git commit -m "test: add search unit tests"

# 4. Rebase กับ main ล่าสุด (ก่อน push)
git fetch origin
git rebase origin/main

# 5. Push
git push origin feature/add-search-api

# 6. สร้าง PR ผ่าน GitHub UI หรือ CLI
gh pr create \
  --title "feat: add product search API" \
  --body "$(cat .github/pr_template.md)" \
  --reviewer "@john-reviewer,@jane-reviewer" \
  --label "feature,needs-review"

# ===============================
# Reviewer: Review PR
# ===============================

# ดู PR ที่ assigned
gh pr list --assignee @me

# Checkout PR เพื่อทดสอบ
gh pr checkout 123

# Run tests
npm test

# Review ใน GitHub UI:
# - เพิ่ม inline comments บน specific lines
# - Approve หรือ Request Changes

# ===============================
# นักพัฒนา: Address Feedback
# ===============================

# แก้ตาม feedback
git add .
git commit -m "fix: address review feedback - add input validation"
git push origin feature/add-search-api

# ===============================
# Merge PR
# ===============================

# หลัง approved → merge
gh pr merge 123 --merge    # regular merge
gh pr merge 123 --squash   # squash merge
gh pr merge 123 --rebase   # rebase merge

# ลบ branch
git push origin --delete feature/add-search-api
git branch -d feature/add-search-api
```

### 13.4 Draft Pull Requests

```bash
# สร้าง Draft PR (ยังไม่พร้อม review)
gh pr create --draft --title "WIP: add payment feature"

# ใช้ Draft PR เพื่อ:
# - แชร์ progress กับทีม
# - ให้ CI รัน tests ก่อน
# - Request early feedback
# - "Work in progress" marker

# เมื่อพร้อมแล้ว mark as "Ready for review"
gh pr ready 123
```

---

## 14. Code Review Best Practices

### 14.1 Reviewer: คำแนะนำการ Review

```
สิ่งที่ต้อง check:

1. Correctness
   □ Logic ถูกต้องไหม?
   □ Edge cases ถูก handle ไหม?
   □ Error handling เหมาะสมไหม?

2. Security
   □ มี input validation ไหม?
   □ มี SQL injection / XSS risks ไหม?
   □ Secrets hardcoded ไหม?

3. Performance
   □ มี N+1 query problems ไหม?
   □ Memory leaks?
   □ Unnecessary computations?

4. Readability
   □ Code อ่านเข้าใจง่ายไหม?
   □ ชื่อ variables/functions บอก intent ชัดเจนไหม?
   □ Comments อธิบาย "why" ไม่ใช่ "what"?

5. Tests
   □ มี tests ครอบคลุมไหม?
   □ Tests ทดสอบ behavior ไม่ใช่ implementation?

6. Design
   □ Code ทำหน้าที่เดียว (Single Responsibility)?
   □ Duplicate code?
   □ เหมาะกับ codebase overall?
```

### 14.2 ตัวอย่าง Review Comments ที่ดี

```python
# PR: เพิ่ม user search function
def find_users(query):
    users = db.execute(f"SELECT * FROM users WHERE name LIKE '%{query}%'")
    return users
```

**Review Comments:**

```
❌ ไม่ดี:
"This is wrong."
"Bad code."

✅ ดี:
"⚠️ Security: This query is vulnerable to SQL injection.
The `query` parameter is directly interpolated into the SQL string.
Use parameterized queries instead:

```python
def find_users(query):
    users = db.execute(
        "SELECT * FROM users WHERE name LIKE ?",
        (f"%{query}%",)
    )
    return users
```

See OWASP SQL Injection guide: [link]"

✅ Suggestion (not blocking):
"💡 Consider: We could add an index on the `name` column to improve
search performance for large tables. Not blocking this PR, but 
might want to create a follow-up ticket."

✅ Positive feedback:
"✅ Nice: Great error handling here! The fallback to empty list 
prevents the caller from needing to handle None cases."
```

### 14.3 Author: รับ Feedback อย่างมืออาชีพ

```
เมื่อได้รับ review comments:

✅ DO:
- อ่านทุก comment อย่างตั้งใจ
- ถามถ้าไม่เข้าใจ (อย่าเดา)
- ขอบคุณ reviewer (เป็น learning opportunity)
- อธิบายถ้ามีเหตุผลที่ดีกว่า
- Mark comments ว่า "resolved" หลังแก้

❌ DON'T:
- Defensive / argue ทันที
- Ignore comments
- "I'll fix later" (ถ้าสำคัญ แก้เลย)
- Personal จนเกินไป
```

### 14.4 Review Comment Labels

```
ใช้ labels เพื่อ communicate intent:

🔴 [blocking]: ต้องแก้ก่อน merge
🟡 [nit]: เรื่องเล็กน้อย (naming, style) - optional
🟢 [suggestion]: ข้อเสนอแนะ ไม่บังคับ
💬 [question]: ต้องการ clarification
💡 [FYI]: แบ่งปัน knowledge ไม่ต้อง action
👏 [praise]: บอกให้รู้ว่า code ดี

ตัวอย่าง:
🔴 [blocking] This API key should not be hardcoded
🟡 [nit] Prefer `is None` over `== None` in Python  
🟢 [suggestion] Consider extracting this into a helper function
💬 [question] Why did we choose Redis over Memcached here?
💡 [FYI] There's an existing utility for this: utils.format_date()
```

---

## 15. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Branch Operations

```
โจทย์:
1. สร้าง repository ใหม่
2. Commit 3 ครั้งบน main
3. สร้าง branch ชื่อ "feature/calculator"
4. เพิ่ม calculator.py บน feature branch (3 commits)
5. Merge กลับ main
6. ลบ feature branch

เฉลย:
```bash
# 1-2. Setup
mkdir calculator-app && cd calculator-app
git init
echo "# Calculator App" > README.md && git add . && git commit -m "docs: add README"
echo "requirements.txt" > requirements.txt && git add . && git commit -m "chore: add requirements"
echo "def main(): pass" > main.py && git add . && git commit -m "feat: add main module"

# 3. สร้าง branch
git checkout -b feature/calculator

# 4. ทำงานบน feature branch
cat > calculator.py << 'EOF'
def add(a, b):
    return a + b
EOF
git add . && git commit -m "feat: add add function"

cat >> calculator.py << 'EOF'

def subtract(a, b):
    return a - b
EOF
git add . && git commit -m "feat: add subtract function"

cat >> calculator.py << 'EOF'

def multiply(a, b):
    return a * b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
EOF
git add . && git commit -m "feat: add multiply and divide functions"

# 5. Merge
git checkout main
git merge --no-ff feature/calculator -m "feat: merge calculator module"

# 6. ลบ branch
git branch -d feature/calculator
git log --oneline --graph
```

---

### แบบฝึกหัดที่ 2: Conflict Resolution Practice

```
โจทย์:
สร้าง conflict และ resolve มัน

เฉลย:
```bash
mkdir conflict-practice && cd conflict-practice
git init

# Initial commit
cat > story.txt << 'EOF'
Once upon a time, there was a programmer.
The programmer wrote excellent code.
Everyone loved the code.
EOF
git add . && git commit -m "docs: add initial story"

# สร้าง 2 branches ที่จะ conflict
git checkout -b branch-a
cat > story.txt << 'EOF'
Once upon a time, there was a programmer.
The programmer wrote AMAZING and BRILLIANT code.
Everyone loved the code deeply.
EOF
git add . && git commit -m "docs: improve story (branch-a version)"

git checkout main
git checkout -b branch-b
cat > story.txt << 'EOF'
Once upon a time, there was a programmer.
The programmer wrote terrible spaghetti code.
Everyone suffered because of the code.
EOF
git add . && git commit -m "docs: add honest story (branch-b version)"

# Merge branch-a ไป main ก่อน
git checkout main
git merge branch-a

# Merge branch-b → CONFLICT!
git merge branch-b
# CONFLICT (content): Merge conflict in story.txt

# ดู conflict
cat story.txt

# แก้ conflict (เลือก version ที่ดีที่สุด)
cat > story.txt << 'EOF'
Once upon a time, there was a programmer.
The programmer wrote amazing code, though sometimes with bugs.
Everyone appreciated the effort and learned from the mistakes.
EOF

git add story.txt
git commit -m "docs: resolve merge conflict in story"
git log --oneline --graph
```

---

### แบบฝึกหัดที่ 3: Interactive Rebase

```
โจทย์:
1. สร้าง 5 commits ที่ messy
2. ใช้ interactive rebase squash เหลือ 2 commits สะอาด

เฉลย:
```bash
mkdir rebase-practice && cd rebase-practice
git init

# สร้าง messy commits
echo "initial" > app.py && git add . && git commit -m "initial"
echo "add login" >> app.py && git add . && git commit -m "WIP login"
echo "fix typo" >> app.py && git add . && git commit -m "fix typo"
echo "fix again" >> app.py && git add . && git commit -m "fix again"
echo "add tests" > test.py && git add . && git commit -m "add tests finally"
echo "fix test" >> test.py && git add . && git commit -m "fix test"
echo "clean up" >> app.py && git add . && git commit -m "cleanup"

git log --oneline
# abc1234 cleanup
# def5678 fix test
# ghi9012 add tests finally
# jkl0123 fix again
# mno4567 fix typo
# pqr8901 WIP login
# stu2345 initial

# Interactive rebase สุดท้าย 6 commits
git rebase -i HEAD~6

# ใน editor เปลี่ยนเป็น:
# pick pqr8901 WIP login
# squash mno4567 fix typo
# squash jkl0123 fix again
# pick ghi9012 add tests finally
# squash def5678 fix test
# squash abc1234 cleanup

# หลัง rebase
git log --oneline
# ควรเหลือ 2 clean commits + initial
```

---

### แบบฝึกหัดที่ 4: Git Flow Simulation

```
โจทย์:
Simulate Git Flow สำหรับ mini project:
1. สร้าง main และ develop branches
2. พัฒนา 2 features พร้อมกัน
3. ทำ release
4. Simulate hotfix

เฉลย:
```bash
mkdir gitflow-practice && cd gitflow-practice
git init

# Setup
echo "# E-Commerce App v1.0" > README.md
git add . && git commit -m "chore: initial commit"

# สร้าง develop branch
git checkout -b develop

echo "App skeleton" > app.py
git add . && git commit -m "chore: add app skeleton"

# Feature 1: User Auth
git checkout -b feature/user-auth develop
echo "def login(): pass" > auth.py && git add . && git commit -m "feat: add login"
echo "def logout(): pass" >> auth.py && git add . && git commit -m "feat: add logout"
echo "def register(): pass" >> auth.py && git add . && git commit -m "feat: add register"

# Feature 2: Product Catalog (พัฒนาพร้อมกัน)
git checkout develop
git checkout -b feature/product-catalog develop
echo "def list_products(): pass" > products.py && git add . && git commit -m "feat: add product listing"
echo "def get_product(id): pass" >> products.py && git add . && git commit -m "feat: add product detail"

# Merge features ไป develop
git checkout develop
git merge --no-ff feature/user-auth -m "feat: merge user authentication"
git merge --no-ff feature/product-catalog -m "feat: merge product catalog"
git branch -d feature/user-auth
git branch -d feature/product-catalog

# สร้าง release
git checkout -b release/1.0.0 develop
echo "1.0.0" > version.txt
git add . && git commit -m "chore: bump version to 1.0.0"
echo "Release notes" > CHANGELOG.md
git add . && git commit -m "docs: add release notes"

# Finish release
git checkout main
git merge --no-ff release/1.0.0 -m "release: v1.0.0"
git tag -a v1.0.0 -m "Version 1.0.0"
git checkout develop
git merge --no-ff release/1.0.0 -m "chore: merge release back to develop"
git branch -d release/1.0.0

# Hotfix (bug ใน production!)
git checkout main
git checkout -b hotfix/fix-login-crash
echo "def login(): # fixed version" > auth.py
git add . && git commit -m "fix: resolve login crash"

git checkout main
git merge --no-ff hotfix/fix-login-crash -m "hotfix: fix login crash"
git tag -a v1.0.1 -m "Version 1.0.1 - hotfix"
git checkout develop
git merge --no-ff hotfix/fix-login-crash -m "chore: merge hotfix to develop"
git branch -d hotfix/fix-login-crash

# ดู history
git log --oneline --graph --all
```

---

### แบบฝึกหัดที่ 5: GitHub Flow Simulation

```
โจทย์:
Simulate GitHub Flow สำหรับ web project
1. main branch เป็น production-ready เสมอ
2. สร้าง 3 features พร้อมกัน
3. Merge ทีละ feature หลัง "review"

เฉลย:
```bash
mkdir github-flow-practice && cd github-flow-practice
git init

echo "# Web App" > README.md
git add . && git commit -m "chore: initial commit"

# Feature 1: Header component
git checkout -b feature/header-component
echo "<header>Navigation</header>" > header.html
git add . && git commit -m "feat: add header component"
echo "header { background: #333; }" > header.css
git add . && git commit -m "style: add header styles"

# Merge to main (after "review")
git checkout main
git merge --no-ff feature/header-component -m "feat: add header component (#1)"
git branch -d feature/header-component
echo "→ Deploy to production!"

# Feature 2: Footer
git checkout -b feature/footer-component
echo "<footer>Footer</footer>" > footer.html
git add . && git commit -m "feat: add footer component"

# Feature 3: Contact form (starts while footer is in review)
git checkout main
git checkout -b feature/contact-form
cat > contact.html << 'EOF'
<form>
  <input type="email" placeholder="Email">
  <textarea placeholder="Message"></textarea>
  <button type="submit">Send</button>
</form>
EOF
git add . && git commit -m "feat: add contact form"

# Merge footer
git checkout main
git merge --no-ff feature/footer-component -m "feat: add footer component (#2)"
git branch -d feature/footer-component
echo "→ Deploy to production!"

# Merge contact form
git checkout main
git merge --no-ff feature/contact-form -m "feat: add contact form (#3)"
git branch -d feature/contact-form
echo "→ Deploy to production!"

git log --oneline --graph
```

---

### แบบฝึกหัดที่ 6: Tagging และ Versioning

```
โจทย์:
1. สร้าง project และ commit หลายครั้ง
2. สร้าง annotated tags สำหรับ v1.0.0, v1.1.0, v1.1.1
3. ดูข้อมูล tags
4. Checkout ไปยัง tag เก่า

เฉลย:
```bash
mkdir tag-practice && cd tag-practice
git init

# Initial release
echo "v1.0 feature A" > app.py
git add . && git commit -m "feat: add feature A"
echo "v1.0 feature B" >> app.py
git add . && git commit -m "feat: add feature B"

# Tag v1.0.0
git tag -a v1.0.0 -m "Release v1.0.0

Initial release featuring:
- Feature A
- Feature B"

# Development continues
echo "v1.1 feature C" >> app.py
git add . && git commit -m "feat: add feature C"
echo "v1.1 fix bug" >> app.py
git add . && git commit -m "fix: resolve edge case"

# Tag v1.1.0
git tag -a v1.1.0 -m "Release v1.1.0

New features:
- Feature C
Bug fixes:
- Edge case handling"

# Hotfix
echo "critical fix" >> app.py
git add . && git commit -m "fix: critical bug fix"

# Tag v1.1.1
git tag -a v1.1.1 -m "Release v1.1.1 - hotfix"

# ดู tags
git tag
git show v1.0.0
git log --oneline --decorate

# Checkout ไป version เก่า
git checkout v1.0.0
cat app.py  # จะเห็นแค่ v1.0 content

# กลับมา main
git checkout main
```

---

### แบบฝึกหัดที่ 7: Cherry-Pick

```
โจทย์:
1. สร้าง main branch และ develop branch
2. มี hotfix commit บน develop
3. Cherry-pick เฉพาะ hotfix commit ไป main

เฉลย:
```bash
mkdir cherry-practice && cd cherry-practice
git init

# Main branch setup
echo "Production code v1.0" > app.py
git add . && git commit -m "feat: v1.0 production code"

# Develop branch มีหลาย commits
git checkout -b develop
echo "New feature in progress" >> app.py
git add . && git commit -m "feat: new feature WIP"

echo "Another feature" >> app.py
git add . && git commit -m "feat: another feature"

# Critical bug fix (ต้องการ backport ไป main)
echo "# BUG FIX: security patch" >> app.py
git add . && git commit -m "fix: critical security patch"

echo "More development" >> app.py
git add . && git commit -m "feat: more features"

# ดู commits
git log --oneline
# Copy hash ของ "fix: critical security patch"
FIX_HASH=$(git log --oneline | grep "critical security patch" | cut -d' ' -f1)
echo "Fix commit hash: $FIX_HASH"

# Cherry-pick ไป main
git checkout main
git cherry-pick $FIX_HASH

# ดูว่า main มีแค่ initial + fix (ไม่มี features)
git log --oneline
cat app.py  # ควรมีแค่ production code + security patch
```

---

### แบบฝึกหัดที่ 8: Pull Request Template

```
โจทย์:
สร้าง Pull Request template สำหรับ project

เฉลย:
```bash
mkdir pr-template-practice && cd pr-template-practice
git init

# สร้าง PR template
mkdir -p .github
cat > .github/PULL_REQUEST_TEMPLATE.md << 'EOF'
## 📋 Summary
<!-- บอกว่า PR นี้ทำอะไร ใน 1-3 ประโยค -->

## 🔄 Type of Change
- [ ] 🐛 Bug fix
- [ ] ✨ New feature
- [ ] 💥 Breaking change
- [ ] 📝 Documentation
- [ ] ♻️ Refactor
- [ ] 🎨 Style/UI change
- [ ] ⚡ Performance improvement

## 📝 Changes Made
<!-- List การเปลี่ยนแปลงที่สำคัญ -->
- 
- 

## 🧪 Testing
- [ ] Unit tests pass (`npm test` / `pytest`)
- [ ] Integration tests pass
- [ ] Manual testing completed
- [ ] No regressions found

## 📸 Screenshots (if UI changes)
<!-- เพิ่ม before/after screenshots -->

## 🔗 Related Issues
Closes #

## ⚠️ Risks
<!-- ระบุ risks หรือ considerations -->

## 📚 Documentation
- [ ] README updated (if needed)
- [ ] API docs updated (if needed)
- [ ] CHANGELOG updated

## 🔍 Reviewer Notes
<!-- อะไรที่ reviewer ควรให้ความสนใจเป็นพิเศษ -->
EOF

git add .github/PULL_REQUEST_TEMPLATE.md
git commit -m "docs: add pull request template"
```

---

### แบบฝึกหัดที่ 9: Branch Strategy Decision

```
โจทย์:
เลือก branching strategy ที่เหมาะสมสำหรับแต่ละ scenario

Scenario A: 
บริษัทพัฒนา banking app ที่ release ทุก quarter
มีทีม 50+ คน
ต้องผ่าน compliance review ก่อน deploy

Scenario B:
Startup 5 คน พัฒนา SaaS app
Deploy หลายครั้งต่อวัน
ต้องการ fast iteration

Scenario C:
Open source project ที่ support หลาย versions
(v1.x, v2.x ยังต้อง maintain)
Community contributors

เฉลย:
A: Git Flow
   เหตุผล: scheduled releases, compliance gates,
   ทีมใหญ่ต้องการ structured workflow

B: GitHub Flow หรือ Trunk-Based Development
   เหตุผล: continuous deployment, small team,
   ต้องการ speed

C: GitLab Flow with release branches
   เหตุผล: หลาย versions ที่ต้อง maintain,
   community contributors, structured releases
```

---

### แบบฝึกหัดที่ 10: Complete Workflow Practice

```
โจทย์ใหญ่:
Simulate การทำงาน 1 sprint สมบูรณ์ (ทำคนเดียวแต่ simulate 2 developers)

Sprint goal: เพิ่ม Blog feature ให้ web app

Requirements:
- Dev 1: Blog listing page
- Dev 2: Blog detail page
- Hotfix: Fix typo ใน existing page

ใช้ GitHub Flow

เฉลย:
```bash
mkdir sprint-simulation && cd sprint-simulation
git init

# Initial main
echo "<html><body><h1>Web App</h1></body></html>" > index.html
git add . && git commit -m "feat: initial web app"

echo "body { font-family: Arial; }" > style.css
git add . && git commit -m "style: add base styles"

# Dev 1: Blog listing
git checkout -b feature/blog-listing
cat > blog.html << 'EOF'
<html>
<body>
  <h1>Blog Posts</h1>
  <ul>
    <li><a href="#">Post 1</a></li>
    <li><a href="#">Post 2</a></li>
  </ul>
</body>
</html>
EOF
git add . && git commit -m "feat: add blog listing page"

# Dev 2: Blog detail (starts at same time)
git checkout main
git checkout -b feature/blog-detail
cat > blog-detail.html << 'EOF'
<html>
<body>
  <article>
    <h1>Blog Post Title</h1>
    <p>Blog content here...</p>
  </article>
</body>
</html>
EOF
git add . && git commit -m "feat: add blog detail page"

# Hotfix: found typo on main!
git checkout main
git checkout -b hotfix/fix-title-typo
sed 's/Web App/My Web App/' index.html > temp && mv temp index.html
git add . && git commit -m "fix: correct app title typo"

# Merge hotfix first (urgent)
git checkout main
git merge --no-ff hotfix/fix-title-typo -m "fix: merge title typo fix"
git branch -d hotfix/fix-title-typo
echo "🚀 Deployed hotfix to production!"

# Dev 1 finishes and merges
git checkout main
git pull  # get latest changes
git checkout feature/blog-listing
git rebase main  # rebase กับ latest main
git checkout main
git merge --no-ff feature/blog-listing -m "feat: add blog listing page (#PR-1)"
git branch -d feature/blog-listing
echo "🚀 Deployed blog listing!"

# Dev 2 finishes and merges
git checkout feature/blog-detail
git rebase main  # rebase กับ latest
git checkout main
git merge --no-ff feature/blog-detail -m "feat: add blog detail page (#PR-2)"
git branch -d feature/blog-detail
echo "🚀 Deployed blog detail!"

# สรุป sprint
echo "Sprint Summary:"
git log --oneline --graph
echo ""
echo "Files in main:"
ls -la
```

---

## สรุปบทที่ 3

```
Branching Commands:
git branch                     ← list branches
git checkout -b <name>         ← create and switch
git switch -c <name>           ← create and switch (new)
git merge <branch>             ← merge branch
git rebase <branch>            ← rebase onto branch
git branch -d <name>           ← delete merged branch
git branch -D <name>           ← force delete

Conflict Resolution:
git merge --abort              ← abort merge
git rebase --abort             ← abort rebase
git add <file>                 ← mark conflict resolved
git rebase --continue          ← continue rebase

Cherry-pick & Tags:
git cherry-pick <hash>         ← copy commit
git tag -a v1.0 -m "msg"      ← create annotated tag
git push origin --tags         ← push all tags

Branching Strategies:
├── Git Flow        → scheduled releases, large teams
├── GitHub Flow     → continuous delivery, simple
├── GitLab Flow     → environment-based
└── Trunk-Based     → high-frequency deploy, feature flags

Pull Request Best Practices:
├── Small, focused PRs
├── Descriptive title and description
├── Link related issues
├── Add screenshots for UI changes
└── Self-review before requesting review

Code Review:
├── Check correctness, security, performance
├── Constructive and specific feedback
├── Use labels (blocking, nit, suggestion)
└── Positive feedback matters too!
```

---

## บทต่อไป

**Part 04: GitHub Actions เบื้องต้น** - เราจะนำ Git knowledge มาใช้สร้าง CI/CD pipeline แรกด้วย GitHub Actions

---

## แหล่งเรียนรู้เพิ่มเติม

```
Interactive Learning:
- Learn Git Branching: https://learngitbranching.js.org
  (เหมาะมากสำหรับฝึก branching!)

Branching Strategies:
- A successful Git branching model (Git Flow):
  https://nvie.com/posts/a-successful-git-branching-model/
- GitHub Flow:
  https://docs.github.com/en/get-started/quickstart/github-flow
- Trunk-Based Development:
  https://trunkbaseddevelopment.com

เอกสารอ้างอิง:
- Pro Git (Branch chapter):
  https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell
- Atlassian Git Tutorials:
  https://www.atlassian.com/git/tutorials/comparing-workflows

เครื่องมือ:
- git-flow: https://github.com/petervanderdoes/gitflow-avh
- GitHub CLI: https://cli.github.com
```

---

*ยินดีด้วย! คุณเรียน Git Branching Strategies ครบแล้ว พร้อมสำหรับ CI/CD pipeline จริงๆ แล้ว!* 🎉
