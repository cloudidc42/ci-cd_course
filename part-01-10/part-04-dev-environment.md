# Part 04: ตั้งค่า Development Environment สำหรับ CI/CD

## บทนำ

การมี Development Environment ที่ดีและสม่ำเสมอเป็นรากฐานสำคัญของกระบวนการ CI/CD ที่มีประสิทธิภาพ ในบทนี้เราจะเรียนรู้วิธีตั้งค่า environment ที่สมบูรณ์แบบสำหรับการพัฒนาซอฟต์แวร์สมัยใหม่

**สิ่งที่จะได้เรียนรู้:**
- ติดตั้งและจัดการ programming language runtimes
- ใช้ version managers (asdf, nvm, pyenv)
- ตั้งค่า VS Code สำหรับ CI/CD workflows
- ติดตั้งและใช้งาน Docker Desktop
- ปรับแต่ง terminal environment ด้วย zsh และ oh-my-zsh
- จัดการ dotfiles
- ตั้งค่า pre-commit hooks
- จัดการ environment variables

---

## 4.1 ทำความเข้าใจกับ Development Environment

### ทำไม Dev Environment ถึงสำคัญ?

ลองนึกภาพปัญหาที่เกิดขึ้นบ่อยในทีม:

```
นักพัฒนา A: "บน machine ผม code รัน OK ครับ"
นักพัฒนา B: "แต่บน machine ผมมัน error ตลอดเลย"
DevOps Engineer: "แล้ว CI server ก็ fail อยู่ด้วย..."
```

ปัญหานี้เรียกว่า **"Works on My Machine"** ซึ่งแก้ได้ด้วยการทำให้ environment สม่ำเสมอ

### หลักการสำคัญ

1. **Reproducibility** - ทุกคนใน team ควรมี environment เหมือนกัน
2. **Isolation** - แต่ละ project ควรมี dependencies ของตัวเอง
3. **Automation** - การตั้งค่า environment ควร automate ได้
4. **Documentation** - วิธีตั้งค่าควรมีเอกสารชัดเจน

---

## 4.2 ติดตั้ง Tools บน Linux

### อัพเดท System Package Manager

```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade -y

# ติดตั้ง essential build tools
sudo apt install -y \
  build-essential \
  curl \
  wget \
  git \
  unzip \
  zip \
  software-properties-common \
  apt-transport-https \
  ca-certificates \
  gnupg \
  lsb-release

# CentOS/RHEL/Fedora
sudo dnf update -y
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y curl wget git unzip zip

# Arch Linux
sudo pacman -Syu
sudo pacman -S base-devel curl wget git unzip
```

### ติดตั้ง Git

```bash
# ตรวจสอบ version
git --version

# ตั้งค่า Git (สำคัญมาก!)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main
git config --global core.editor "code --wait"
git config --global pull.rebase false

# ตั้งค่า credential helper
git config --global credential.helper store

# ดู config ทั้งหมด
git config --list --global
```

### ตั้งค่า SSH Key สำหรับ GitHub

```bash
# สร้าง SSH key
ssh-keygen -t ed25519 -C "your.email@example.com"

# เริ่ม ssh-agent
eval "$(ssh-agent -s)"

# เพิ่ม key เข้า agent
ssh-add ~/.ssh/id_ed25519

# แสดง public key เพื่อ copy ไปใส่ GitHub
cat ~/.ssh/id_ed25519.pub

# ทดสอบการเชื่อมต่อ
ssh -T git@github.com
```

---

## 4.3 ติดตั้ง Tools บน macOS

### ติดตั้ง Homebrew

Homebrew เป็น package manager ยอดนิยมสำหรับ macOS

```bash
# ติดตั้ง Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# เพิ่ม Homebrew ใน PATH (สำหรับ Apple Silicon)
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

# ตรวจสอบ
brew --version
brew doctor
```

### ติดตั้ง Tools พื้นฐาน

```bash
# ติดตั้ง essential tools
brew install \
  git \
  curl \
  wget \
  tree \
  jq \
  yq \
  htop \
  tmux \
  fzf \
  ripgrep \
  bat \
  exa \
  fd

# ติดตั้ง GitHub CLI
brew install gh

# Login GitHub CLI
gh auth login
```

### Xcode Command Line Tools

```bash
# ติดตั้ง Xcode Command Line Tools
xcode-select --install

# ตรวจสอบ
xcode-select -p
```

---

## 4.4 ติดตั้ง Tools บน Windows

### Windows Subsystem for Linux (WSL2)

WSL2 เป็นวิธีที่แนะนำสำหรับ development บน Windows

```powershell
# เปิด PowerShell ในฐานะ Administrator
# ติดตั้ง WSL2
wsl --install

# ติดตั้ง Ubuntu distribution
wsl --install -d Ubuntu-22.04

# ตรวจสอบ
wsl --list --verbose

# เปิด Ubuntu
wsl
```

### Windows Package Manager (winget)

```powershell
# ติดตั้ง winget (มาพร้อม Windows 11)
# สำหรับ Windows 10 ต้องติดตั้งจาก Microsoft Store

# ติดตั้ง tools
winget install Git.Git
winget install Microsoft.VisualStudioCode
winget install Docker.DockerDesktop
winget install GitHub.cli
winget install Microsoft.WindowsTerminal

# อัพเดท
winget upgrade --all
```

### Chocolatey (Alternative Package Manager)

```powershell
# ติดตั้ง Chocolatey
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# ติดตั้ง tools
choco install git nodejs python golang -y
choco install docker-desktop -y
```

---

## 4.5 Node.js Version Management

### ติดตั้ง nvm (Node Version Manager)

```bash
# ติดตั้ง nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# โหลด nvm (หรือ restart terminal)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# ตรวจสอบ
nvm --version
```

### จัดการ Node.js versions ด้วย nvm

```bash
# ดู versions ที่มีให้ติดตั้ง
nvm ls-remote

# ดู LTS versions
nvm ls-remote --lts

# ติดตั้ง Node.js versions
nvm install 18        # ติดตั้ง Node.js 18 (LTS)
nvm install 20        # ติดตั้ง Node.js 20 (LTS)
nvm install node      # ติดตั้ง version ล่าสุด

# เลือก version ที่ใช้
nvm use 20

# ตั้ง default version
nvm alias default 20

# ดู versions ที่ติดตั้ง
nvm ls

# ตรวจสอบ version ปัจจุบัน
node --version
npm --version
```

### ใช้ .nvmrc ใน Project

```bash
# สร้างไฟล์ .nvmrc ใน project root
echo "20" > .nvmrc

# ใช้ version จาก .nvmrc
nvm use

# ติดตั้ง version จาก .nvmrc (ถ้ายังไม่มี)
nvm install
```

เพิ่มใน `~/.zshrc` หรือ `~/.bashrc` เพื่อ auto-switch version:

```bash
# Auto-use .nvmrc when entering directory
autoload -U add-zsh-hook

load-nvmrc() {
  local nvmrc_path
  nvmrc_path="$(nvm_find_nvmrc)"

  if [ -n "$nvmrc_path" ]; then
    local nvmrc_node_version
    nvmrc_node_version=$(nvm version "$(cat "${nvmrc_path}")")

    if [ "$nvmrc_node_version" = "N/A" ]; then
      nvm install
    elif [ "$nvmrc_node_version" != "$(nvm version)" ]; then
      nvm use
    fi
  elif [ -n "$(PWD=$OLDPWD nvm_find_nvmrc)" ] && [ "$(nvm version)" != "$(nvm version default)" ]; then
    echo "Reverting to nvm default version"
    nvm use default
  fi
}

add-zsh-hook chpwd load-nvmrc
load-nvmrc
```

---

## 4.6 Python Version Management

### ติดตั้ง pyenv

```bash
# Linux - ติดตั้ง dependencies
sudo apt install -y \
  libbz2-dev \
  libncurses-dev \
  libffi-dev \
  libreadline-dev \
  libssl-dev \
  libsqlite3-dev \
  zlib1g-dev \
  liblzma-dev

# ติดตั้ง pyenv
curl https://pyenv.run | bash

# เพิ่มใน ~/.bashrc หรือ ~/.zshrc
export PYENV_ROOT="$HOME/.pyenv"
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"

# reload shell
source ~/.zshrc
```

### จัดการ Python versions

```bash
# ดู versions ที่มี
pyenv install --list

# ดู versions ที่ stable
pyenv install --list | grep -E "^\s+3\.[0-9]+\.[0-9]+$"

# ติดตั้ง Python versions
pyenv install 3.11.7
pyenv install 3.12.1

# ตั้ง global version
pyenv global 3.12.1

# ตั้ง local version (สำหรับ project specific)
cd my-project
pyenv local 3.11.7

# ดู versions ที่ติดตั้ง
pyenv versions

# ตรวจสอบ
python --version
pip --version
```

### Python Virtual Environments

```bash
# สร้าง virtual environment
python -m venv .venv

# activate
source .venv/bin/activate  # Linux/Mac
.venv\Scripts\activate     # Windows

# ติดตั้ง packages
pip install requests flask

# บันทึก dependencies
pip freeze > requirements.txt

# ติดตั้งจาก requirements.txt
pip install -r requirements.txt

# deactivate
deactivate
```

### ใช้ Poetry สำหรับ Python Package Management

```bash
# ติดตั้ง Poetry
curl -sSL https://install.python-poetry.org | python3 -

# เพิ่ม PATH
export PATH="$HOME/.local/bin:$PATH"

# สร้าง project ใหม่
poetry new my-project

# หรือ init ใน project ที่มีอยู่
poetry init

# ติดตั้ง dependencies
poetry add requests
poetry add --dev pytest black mypy

# รัน commands ใน virtual env
poetry run python app.py
poetry run pytest

# activate virtual env
poetry shell
```

---

## 4.7 Go Version Management

### ติดตั้ง Go

```bash
# ดาวน์โหลด Go จาก official website
# https://go.dev/dl/

# Linux
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz

# เพิ่ม PATH
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.zshrc
echo 'export GOPATH=$HOME/go' >> ~/.zshrc
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.zshrc

source ~/.zshrc

# ตรวจสอบ
go version
```

### ใช้ asdf สำหรับ Go version management

```bash
# ติดตั้ง asdf plugin
asdf plugin add golang https://github.com/asdf-community/asdf-golang.git

# ติดตั้ง Go version
asdf install golang 1.22.0

# ตั้ง global version
asdf global golang 1.22.0

# ตรวจสอบ
go version
```

---

## 4.8 Java Version Management

### ติดตั้ง SDKMAN!

```bash
# ติดตั้ง SDKMAN!
curl -s "https://get.sdkman.io" | bash

# โหลด SDKMAN!
source "$HOME/.sdkman/bin/sdkman-init.sh"

# ตรวจสอบ
sdk version
```

### จัดการ Java versions ด้วย SDKMAN!

```bash
# ดู Java versions ที่มี
sdk list java

# ติดตั้ง Java
sdk install java 21.0.2-tem    # Eclipse Temurin 21
sdk install java 17.0.10-tem   # Eclipse Temurin 17

# เลือก version
sdk use java 21.0.2-tem

# ตั้ง default version
sdk default java 21.0.2-tem

# ดู version ปัจจุบัน
java --version
javac --version

# ติดตั้ง Maven
sdk install maven

# ติดตั้ง Gradle
sdk install gradle
```

---

## 4.9 asdf - Universal Version Manager

asdf เป็น version manager ที่รองรับหลาย language ด้วย plugin system

### ติดตั้ง asdf

```bash
# Clone asdf
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0

# เพิ่มใน ~/.zshrc
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.zshrc
echo '. "$HOME/.asdf/completions/asdf.bash"' >> ~/.zshrc

# reload
source ~/.zshrc

# ตรวจสอบ
asdf version
```

### ใช้งาน asdf

```bash
# ดู plugins ที่มี
asdf plugin list all

# ติดตั้ง plugins
asdf plugin add nodejs https://github.com/asdf-vm/asdf-nodejs.git
asdf plugin add python https://github.com/danhper/asdf-python.git
asdf plugin add golang https://github.com/asdf-community/asdf-golang.git
asdf plugin add java https://github.com/halcyon/asdf-java.git

# ดู plugin ที่ติดตั้ง
asdf plugin list

# ดู versions ที่มี
asdf list all nodejs
asdf list all python

# ติดตั้ง versions
asdf install nodejs 20.11.0
asdf install python 3.12.1

# ตั้ง global versions
asdf global nodejs 20.11.0
asdf global python 3.12.1

# ตั้ง local versions (สร้างไฟล์ .tool-versions)
asdf local nodejs 18.19.0

# ดู versions ที่ติดตั้ง
asdf list nodejs
```

### ไฟล์ .tool-versions

```bash
# สร้างไฟล์ .tool-versions ใน project root
cat > .tool-versions << EOF
nodejs 20.11.0
python 3.12.1
golang 1.22.0
java temurin-21.0.2+13
EOF

# ติดตั้ง versions ทั้งหมดจากไฟล์
asdf install
```

---

## 4.10 VS Code Extensions สำหรับ CI/CD

### ติดตั้ง VS Code

```bash
# macOS
brew install --cask visual-studio-code

# Linux (Ubuntu)
sudo snap install --classic code

# หรือจาก .deb
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
sudo sh -c 'echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/trusted.gpg.d/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'
sudo apt install apt-transport-https
sudo apt update
sudo apt install code
```

### Essential Extensions

```bash
# ติดตั้ง extensions ผ่าน command line
# YAML
code --install-extension redhat.vscode-yaml

# Docker
code --install-extension ms-azuretools.vscode-docker

# GitHub Actions
code --install-extension github.vscode-github-actions

# GitLens
code --install-extension eamodio.gitlens

# Remote Development
code --install-extension ms-vscode-remote.vscode-remote-extensionpack

# ESLint
code --install-extension dbaeumer.vscode-eslint

# Prettier
code --install-extension esbenp.prettier-vscode

# Python
code --install-extension ms-python.python
code --install-extension ms-python.black-formatter
code --install-extension ms-python.mypy-type-checker

# Go
code --install-extension golang.go

# Java Extension Pack
code --install-extension vscjava.vscode-java-pack

# REST Client (แทน Postman)
code --install-extension humao.rest-client

# EditorConfig
code --install-extension editorconfig.editorconfig

# Dotenv
code --install-extension mikestead.dotenv

# Todo Tree
code --install-extension gruntfuggly.todo-tree

# Better Comments
code --install-extension aaron-bond.better-comments
```

### VS Code Settings สำหรับ CI/CD

สร้างไฟล์ `.vscode/settings.json` ใน project:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.rulers": [80, 120],
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.eol": "\n",

  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter",
    "editor.tabSize": 4
  },

  "[go]": {
    "editor.defaultFormatter": "golang.go",
    "editor.tabSize": 4,
    "editor.insertSpaces": false
  },

  "[yaml]": {
    "editor.defaultFormatter": "redhat.vscode-yaml",
    "editor.tabSize": 2
  },

  "yaml.schemas": {
    "https://json.schemastore.org/github-workflow.json": ".github/workflows/*.yml"
  },

  "eslint.enable": true,
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact"
  ],

  "python.linting.enabled": true,
  "python.linting.pylintEnabled": false,
  "python.linting.flake8Enabled": true,

  "terminal.integrated.defaultProfile.linux": "zsh",
  "terminal.integrated.fontFamily": "MesloLGS NF",

  "git.enableSmartCommit": true,
  "git.autofetch": true,
  "git.confirmSync": false,

  "githubActions.workflows.pinned.workflows": []
}
```

### VS Code Extensions Configuration

สร้างไฟล์ `.vscode/extensions.json`:

```json
{
  "recommendations": [
    "redhat.vscode-yaml",
    "ms-azuretools.vscode-docker",
    "github.vscode-github-actions",
    "eamodio.gitlens",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "ms-python.python",
    "editorconfig.editorconfig",
    "mikestead.dotenv",
    "humao.rest-client"
  ]
}
```

---

## 4.11 Docker Desktop

### ติดตั้ง Docker Desktop

**macOS:**
```bash
brew install --cask docker
# หรือดาวน์โหลดจาก https://docs.docker.com/desktop/mac/install/
```

**Linux (Docker Engine):**
```bash
# ลบ old versions
sudo apt remove docker docker-engine docker.io containerd runc

# ติดตั้ง dependencies
sudo apt update
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

# เพิ่ม Docker GPG key
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# เพิ่ม repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# ติดตั้ง Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# เพิ่ม user เข้า docker group
sudo usermod -aG docker $USER
newgrp docker

# ทดสอบ
docker run hello-world
```

**Windows:**
```powershell
winget install Docker.DockerDesktop
# หรือดาวน์โหลดจาก https://docs.docker.com/desktop/windows/install/
```

### คำสั่ง Docker พื้นฐาน

```bash
# ตรวจสอบ Docker
docker --version
docker compose version

# Pull image
docker pull ubuntu:22.04
docker pull node:20-alpine
docker pull python:3.12-slim

# รัน container
docker run -it ubuntu:22.04 bash
docker run -d -p 3000:3000 node:20-alpine

# ดู containers
docker ps          # running containers
docker ps -a       # ทุก containers

# ดู images
docker images

# หยุด container
docker stop <container_id>

# ลบ container
docker rm <container_id>

# ลบ image
docker rmi <image_id>

# Docker Compose
docker compose up -d
docker compose down
docker compose logs -f

# ทำความสะอาด
docker system prune -a
```

---

## 4.12 Terminal Environment - zsh และ oh-my-zsh

### ติดตั้ง zsh

```bash
# Ubuntu/Debian
sudo apt install -y zsh

# macOS (มาพร้อมอยู่แล้ว แต่อาจต้องอัพเดท)
brew install zsh

# ตรวจสอบ
zsh --version

# ตั้ง zsh เป็น default shell
chsh -s $(which zsh)

# logout และ login ใหม่ หรือรัน
exec zsh
```

### ติดตั้ง oh-my-zsh

```bash
# ติดตั้ง oh-my-zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### ติดตั้ง Powerlevel10k Theme

```bash
# clone theme
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k

# แก้ไข ~/.zshrc
# เปลี่ยน ZSH_THEME="robbyrussell" เป็น:
ZSH_THEME="powerlevel10k/powerlevel10k"

# ติดตั้ง fonts (MesloLGS NF)
# ดาวน์โหลดจาก https://github.com/romkatv/powerlevel10k#fonts

# reload
source ~/.zshrc

# ตั้งค่า theme (wizard จะเริ่มอัตโนมัติ)
p10k configure
```

### Plugins สำหรับ oh-my-zsh

```bash
# ติดตั้ง zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

# ติดตั้ง zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# ติดตั้ง zsh-completions
git clone https://github.com/zsh-users/zsh-completions \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-completions

# แก้ไข ~/.zshrc เพิ่ม plugins
plugins=(
  git
  docker
  docker-compose
  kubectl
  aws
  node
  npm
  python
  golang
  zsh-autosuggestions
  zsh-syntax-highlighting
  zsh-completions
  fzf
  z
)
```

### Useful Aliases สำหรับ CI/CD

เพิ่มใน `~/.zshrc`:

```bash
# ===== Git Aliases =====
alias g='git'
alias gs='git status'
alias ga='git add'
alias gc='git commit'
alias gcm='git commit -m'
alias gca='git commit --amend'
alias gp='git push'
alias gpf='git push --force-with-lease'
alias gl='git pull'
alias glog='git log --oneline --graph --decorate'
alias gd='git diff'
alias gds='git diff --staged'
alias gb='git branch'
alias gco='git checkout'
alias gcb='git checkout -b'
alias gst='git stash'
alias gsp='git stash pop'

# ===== Docker Aliases =====
alias d='docker'
alias dc='docker compose'
alias dcu='docker compose up -d'
alias dcd='docker compose down'
alias dcl='docker compose logs -f'
alias dps='docker ps'
alias dpsa='docker ps -a'
alias di='docker images'
alias dprune='docker system prune -af'

# ===== Kubernetes Aliases =====
alias k='kubectl'
alias kget='kubectl get'
alias kdesc='kubectl describe'
alias klog='kubectl logs'
alias kex='kubectl exec -it'
alias kns='kubectl config set-context --current --namespace'

# ===== Node.js Aliases =====
alias ni='npm install'
alias nr='npm run'
alias nrd='npm run dev'
alias nrb='npm run build'
alias nrt='npm run test'
alias nrl='npm run lint'
alias nci='npm ci'

# ===== Python Aliases =====
alias py='python3'
alias pip='pip3'
alias venv='python3 -m venv .venv'
alias activate='source .venv/bin/activate'

# ===== Utilities =====
alias ll='ls -la'
alias la='ls -A'
alias l='ls -CF'
alias ..='cd ..'
alias ...='cd ../..'
alias cls='clear'
alias path='echo -e ${PATH//:/\\n}'
alias ports='ss -tulpn'
alias myip='curl -s https://api.ipify.org'
```

---

## 4.13 Dotfiles Management

### ทำไมต้อง Manage Dotfiles?

Dotfiles คือไฟล์ configuration ที่ขึ้นต้นด้วย `.` เช่น `.zshrc`, `.gitconfig`, `.vimrc` การจัดการ dotfiles ที่ดีช่วยให้:
- ย้าย setup ระหว่าง machines ได้ง่าย
- ทีมสามารถ share configuration ได้
- มี version history ของ configuration

### โครงสร้าง Dotfiles Repository

```
~/dotfiles/
├── .zshrc
├── .gitconfig
├── .gitignore_global
├── .tmux.conf
├── .vimrc
├── .editorconfig
├── install.sh
├── README.md
└── scripts/
    ├── install_tools.sh
    └── setup_macos.sh
```

### สร้าง Dotfiles Repository

```bash
# สร้าง dotfiles directory
mkdir ~/dotfiles
cd ~/dotfiles

# สร้าง symlinks
ln -sf ~/dotfiles/.zshrc ~/.zshrc
ln -sf ~/dotfiles/.gitconfig ~/.gitconfig

# หรือใช้ GNU Stow
sudo apt install stow  # Ubuntu
brew install stow      # macOS

# ใช้ Stow สร้าง symlinks อัตโนมัติ
cd ~/dotfiles
stow zsh git vim
```

### install.sh

```bash
#!/bin/bash
set -e

echo "Installing dotfiles..."

DOTFILES_DIR="$HOME/dotfiles"

# สร้าง symlinks
for file in .zshrc .gitconfig .gitignore_global .tmux.conf .editorconfig; do
  if [ -f "$HOME/$file" ]; then
    echo "Backing up $file"
    mv "$HOME/$file" "$HOME/$file.backup.$(date +%Y%m%d)"
  fi
  ln -sf "$DOTFILES_DIR/$file" "$HOME/$file"
  echo "Linked $file"
done

echo "Dotfiles installed successfully!"
```

### .gitconfig ตัวอย่าง

```ini
[user]
    name = Your Name
    email = your.email@example.com

[core]
    editor = code --wait
    excludesfile = ~/.gitignore_global
    autocrlf = input
    pager = delta

[init]
    defaultBranch = main

[pull]
    rebase = false

[push]
    default = current
    autoSetupRemote = true

[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    lg = log --oneline --graph --decorate --all
    undo = reset HEAD~1 --mixed
    unstage = reset HEAD --
    last = log -1 HEAD
    visual = !gitk

[color]
    ui = auto

[delta]
    navigate = true
    light = false
    side-by-side = true

[merge]
    conflictstyle = diff3

[diff]
    colorMoved = default
```

---

## 4.14 Pre-commit Hooks

### ทำความเข้าใจ Git Hooks

Git hooks เป็น scripts ที่รันอัตโนมัติเมื่อเกิด events ใน git เช่น commit, push, merge

### ติดตั้ง pre-commit Framework

```bash
# ติดตั้งด้วย pip
pip install pre-commit

# หรือด้วย Homebrew
brew install pre-commit

# ตรวจสอบ
pre-commit --version
```

### สร้าง .pre-commit-config.yaml

```yaml
# .pre-commit-config.yaml
repos:
  # General hooks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
        name: Trim trailing whitespace
      - id: end-of-file-fixer
        name: Fix end of files
      - id: check-yaml
        name: Check YAML files
      - id: check-json
        name: Check JSON files
      - id: check-toml
        name: Check TOML files
      - id: check-merge-conflict
        name: Check for merge conflicts
      - id: check-added-large-files
        name: Check for large files
        args: ['--maxkb=1000']
      - id: detect-private-key
        name: Detect private keys
      - id: mixed-line-ending
        name: Check line endings

  # Commit message format
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.13.0
    hooks:
      - id: commitizen
        stages: [commit-msg]

  # Secret detection
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        name: Detect secrets
        args: ['--baseline', '.secrets.baseline']

  # JavaScript/TypeScript
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.56.0
    hooks:
      - id: eslint
        files: \.[jt]sx?$
        types: [file]
        additional_dependencies:
          - eslint@8.56.0
          - eslint-config-prettier@9.1.0

  # Python
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.2.0
    hooks:
      - id: ruff
        name: Ruff linter
        args: [--fix]
      - id: ruff-format
        name: Ruff formatter

  # Dockerfile
  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint
        name: Hadolint Dockerfile linter

  # Shell scripts
  - repo: https://github.com/shellcheck-py/shellcheck-py
    rev: v0.9.0.6
    hooks:
      - id: shellcheck
        name: Shellcheck

  # GitHub Actions
  - repo: https://github.com/rhysd/actionlint
    rev: v1.6.26
    hooks:
      - id: actionlint
        name: Actionlint (GitHub Actions linter)
```

### ติดตั้งและใช้งาน pre-commit

```bash
# ติดตั้ง hooks ใน repository
pre-commit install

# ติดตั้ง commit-msg hook
pre-commit install --hook-type commit-msg

# รัน hooks บนไฟล์ทั้งหมด
pre-commit run --all-files

# รัน hook เฉพาะ
pre-commit run trailing-whitespace

# อัพเดท hooks
pre-commit autoupdate

# ข้าม hooks (emergency เท่านั้น!)
git commit -m "message" --no-verify
```

### Commitizen สำหรับ Conventional Commits

```bash
# ติดตั้ง commitizen
pip install commitizen

# หรือ npm
npm install -g commitizen cz-conventional-changelog

# ตั้งค่าใน package.json
{
  "config": {
    "commitizen": {
      "path": "cz-conventional-changelog"
    }
  }
}

# ใช้งาน
cz commit  # หรือ git cz
```

---

## 4.15 EditorConfig

EditorConfig ช่วยให้ editor settings สม่ำเสมอทั้ง team

### สร้าง .editorconfig

```ini
# .editorconfig
# EditorConfig is awesome: https://EditorConfig.org

# top-most EditorConfig file
root = true

# Unix-style newlines with a newline ending every file
[*]
end_of_line = lf
insert_final_newline = true
charset = utf-8
trim_trailing_whitespace = true

# 2 space indentation (default)
[*.{js,ts,jsx,tsx,json,yaml,yml,css,scss,html}]
indent_style = space
indent_size = 2

# 4 space indentation for Python
[*.py]
indent_style = space
indent_size = 4
max_line_length = 88

# Tab indentation for Go
[*.go]
indent_style = tab
indent_size = 4

# 4 space for Java
[*.java]
indent_style = space
indent_size = 4

# Makefiles require tabs
[Makefile]
indent_style = tab

# Markdown
[*.md]
trim_trailing_whitespace = false
max_line_length = 120

# GitHub Actions
[.github/workflows/*.yml]
indent_style = space
indent_size = 2
```

---

## 4.16 Environment Variables Management

### ทำไม .env ถึงสำคัญ?

```bash
# BAD! - อย่า hardcode secrets
const apiKey = "sk-abc123def456"

# GOOD! - ใช้ environment variables
const apiKey = process.env.API_KEY
```

### โครงสร้างไฟล์ .env

```bash
# .env.example (commit ไปใน repository)
DATABASE_URL=postgres://user:password@localhost:5432/myapp
REDIS_URL=redis://localhost:6379
API_KEY=your-api-key-here
SECRET_KEY=your-secret-key-here
NODE_ENV=development
PORT=3000
LOG_LEVEL=debug

# .env (ไม่ commit! ใส่ใน .gitignore)
DATABASE_URL=postgres://admin:MySecretPass123@db.example.com:5432/production
REDIS_URL=redis://redis.example.com:6379
API_KEY=sk-abc123realkey456
SECRET_KEY=superSecretKey789
NODE_ENV=production
PORT=8080
LOG_LEVEL=warn
```

### .gitignore สำหรับ Environment Files

```gitignore
# Environment files
.env
.env.local
.env.development.local
.env.test.local
.env.production.local
.env.*.local

# Secrets
*.pem
*.key
*.p12
*.pfx
secrets/
config/secrets.yml
```

### ใช้ dotenv สำหรับ Node.js

```bash
# ติดตั้ง
npm install dotenv

# โหลดใน application
# ไว้บนสุดของ main file
require('dotenv').config()

# หรือ ES modules
import 'dotenv/config'

# ใช้งาน
const port = process.env.PORT || 3000
const dbUrl = process.env.DATABASE_URL
```

### ใช้ python-dotenv สำหรับ Python

```bash
pip install python-dotenv
```

```python
from dotenv import load_dotenv
import os

# โหลด .env
load_dotenv()

# ใช้งาน
database_url = os.getenv("DATABASE_URL")
api_key = os.getenv("API_KEY")
port = int(os.getenv("PORT", "3000"))
```

### direnv - Auto-load Environment Variables

```bash
# ติดตั้ง direnv
sudo apt install direnv  # Ubuntu
brew install direnv      # macOS

# เพิ่มใน ~/.zshrc
eval "$(direnv hook zsh)"

# สร้างไฟล์ .envrc ใน project
cat > .envrc << EOF
export DATABASE_URL="postgres://localhost:5432/myapp"
export NODE_ENV="development"
export PORT="3000"
EOF

# อนุญาต direnv
direnv allow

# ทดสอบ
echo $DATABASE_URL
```

---

## 4.17 Project Scaffolding Templates

### สร้าง Project Template Script

```bash
#!/bin/bash
# scripts/init-project.sh

set -e

PROJECT_NAME=${1:-"my-project"}
PROJECT_TYPE=${2:-"node"}

echo "Creating $PROJECT_TYPE project: $PROJECT_NAME"

# สร้าง directory structure
mkdir -p "$PROJECT_NAME"/{src,tests,docs,.github/workflows}

cd "$PROJECT_NAME"

# Initialize git
git init
git branch -m main

# สร้าง .gitignore
cat > .gitignore << 'EOF'
# Dependencies
node_modules/
.venv/
vendor/

# Build outputs
dist/
build/
*.egg-info/
target/

# Environment
.env
.env.local
.env.*.local

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Coverage
coverage/
.coverage
htmlcov/
EOF

# สร้าง .editorconfig
cat > .editorconfig << 'EOF'
root = true

[*]
end_of_line = lf
insert_final_newline = true
charset = utf-8
trim_trailing_whitespace = true

[*.{js,ts,json,yaml,yml}]
indent_style = space
indent_size = 2

[*.py]
indent_style = space
indent_size = 4
EOF

# สร้าง .env.example
cat > .env.example << 'EOF'
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://localhost:5432/myapp
EOF

echo "Project $PROJECT_NAME created successfully!"
```

### Makefile สำหรับ Common Commands

```makefile
# Makefile
.PHONY: install test lint build clean dev help

# Default target
.DEFAULT_GOAL := help

## install: ติดตั้ง dependencies
install:
	npm ci

## dev: รัน development server
dev:
	npm run dev

## test: รัน tests
test:
	npm test

## test-coverage: รัน tests พร้อม coverage
test-coverage:
	npm test -- --coverage

## lint: รัน linter
lint:
	npm run lint

## lint-fix: รัน linter และ fix
lint-fix:
	npm run lint -- --fix

## format: format code
format:
	npm run format

## build: build project
build:
	npm run build

## clean: ลบ build artifacts
clean:
	rm -rf dist/ node_modules/ coverage/

## docker-build: build Docker image
docker-build:
	docker build -t $(IMAGE_NAME):$(TAG) .

## docker-run: รัน Docker container
docker-run:
	docker run -p 3000:3000 $(IMAGE_NAME):$(TAG)

## help: แสดง help
help:
	@echo "Available targets:"
	@grep -E '^## [a-zA-Z_-]+:' Makefile | sed 's/^## /  /' | column -t -s ':'
```

---

## 4.18 Workshop - ตั้งค่า Complete Development Environment

### Workshop 1: ตั้งค่า Node.js Project

```bash
# 1. ติดตั้ง nvm และ Node.js
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.zshrc
nvm install 20
nvm use 20

# 2. สร้าง project
mkdir workshop-node && cd workshop-node
npm init -y

# 3. ติดตั้ง development tools
npm install --save-dev \
  eslint \
  prettier \
  @typescript-eslint/eslint-plugin \
  @typescript-eslint/parser \
  typescript \
  jest \
  ts-jest \
  @types/jest \
  husky \
  lint-staged

# 4. ตั้งค่า TypeScript
npx tsc --init

# 5. สร้าง .eslintrc.js
cat > .eslintrc.js << 'EOF'
module.exports = {
  parser: '@typescript-eslint/parser',
  plugins: ['@typescript-eslint'],
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
  ],
  rules: {
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/explicit-function-return-type': 'warn',
  },
};
EOF

# 6. สร้าง .prettierrc
cat > .prettierrc << 'EOF'
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2
}
EOF

# 7. ตั้งค่า Husky
npx husky init

# 8. สร้าง pre-commit hook
cat > .husky/pre-commit << 'EOF'
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npx lint-staged
EOF

# 9. เพิ่ม lint-staged config ใน package.json
cat > lint-staged.config.js << 'EOF'
module.exports = {
  '*.{js,ts}': ['eslint --fix', 'prettier --write'],
  '*.{json,yaml,yml,md}': ['prettier --write'],
};
EOF

# 10. เพิ่ม scripts ใน package.json
npm pkg set scripts.lint="eslint src --ext .ts"
npm pkg set scripts.lint:fix="eslint src --ext .ts --fix"
npm pkg set scripts.format="prettier --write 'src/**/*.ts'"
npm pkg set scripts.test="jest"
npm pkg set scripts.build="tsc"

echo "Node.js project setup complete!"
```

### Workshop 2: สร้าง Pre-commit Workflow

```bash
# 1. สร้าง project ใหม่
mkdir workshop-precommit && cd workshop-precommit
git init

# 2. ติดตั้ง pre-commit
pip install pre-commit

# 3. สร้าง .pre-commit-config.yaml
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: detect-private-key

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.2.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
EOF

# 4. ติดตั้ง hooks
pre-commit install

# 5. ทดสอบ - สร้างไฟล์ Python ที่มีปัญหา
cat > test_file.py << 'EOF'
import os
import sys
import json

def hello_world():
    x = 1  # unused variable
    print("Hello World")   
    
EOF

# 6. commit (hooks จะรันอัตโนมัติ)
git add .
git commit -m "test: add test file"

echo "Pre-commit hooks are working!"
```

---

## 4.19 Exercises

### Exercise 1: ตั้งค่า Complete Environment

**โจทย์:** ตั้งค่า development environment สำหรับ Node.js project ที่มี:
- Node.js 20 ผ่าน nvm
- ESLint + Prettier
- Pre-commit hooks
- EditorConfig
- .env management

**Expected output:**
```bash
$ git commit -m "feat: initial setup"
# ESLint check... Passed
# Prettier format check... Passed
# No trailing whitespace... Passed
[main (root-commit) abc1234] feat: initial setup
```

### Exercise 2: Dotfiles Setup

**โจทย์:** สร้าง dotfiles repository ที่มี:
- `.zshrc` พร้อม aliases สำหรับ CI/CD
- `.gitconfig` พร้อม useful aliases
- `install.sh` สำหรับ auto-setup

### Exercise 3: Multi-language Project

**โจทย์:** สร้าง project ที่ใช้ asdf จัดการหลาย languages:
- Node.js 20
- Python 3.12
- Go 1.22

พร้อม `.tool-versions` file และ Makefile สำหรับ commands

### Exercise 4: Docker Development Environment

**โจทย์:** สร้าง `docker-compose.yml` สำหรับ local development ที่มี:
- App container (Node.js)
- PostgreSQL database
- Redis cache

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgres://postgres:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

## 4.20 สรุป

ในบทนี้เราได้เรียนรู้:

1. **การติดตั้ง tools** บน Linux, macOS, และ Windows
2. **Version managers** - nvm, pyenv, asdf, SDKMAN! สำหรับจัดการ language versions
3. **VS Code setup** พร้อม extensions และ settings ที่จำเป็น
4. **Docker Desktop** และคำสั่งพื้นฐาน
5. **Terminal enhancement** ด้วย zsh, oh-my-zsh, และ aliases
6. **Dotfiles management** เพื่อ sync configuration ระหว่าง machines
7. **Pre-commit hooks** เพื่อ automate code quality checks
8. **EditorConfig** เพื่อให้ editor settings สม่ำเสมอ
9. **Environment variables** management ด้วย .env files
10. **Project scaffolding** templates

### Checklist ก่อนเริ่ม Project

```
[ ] ติดตั้ง version manager (nvm/pyenv/asdf)
[ ] กำหนด language version ใน .nvmrc หรือ .tool-versions
[ ] ตั้งค่า .editorconfig
[ ] ตั้งค่า linter และ formatter
[ ] เพิ่ม .env.example
[ ] เพิ่ม .gitignore ที่ครบถ้วน
[ ] ตั้งค่า pre-commit hooks
[ ] สร้าง Makefile สำหรับ common commands
[ ] เพิ่ม .vscode/settings.json และ extensions.json
[ ] Document การตั้งค่าใน README.md
```

### แหล่งข้อมูลเพิ่มเติม

- [nvm GitHub](https://github.com/nvm-sh/nvm)
- [pyenv GitHub](https://github.com/pyenv/pyenv)
- [asdf Website](https://asdf-vm.com/)
- [oh-my-zsh Website](https://ohmyz.sh/)
- [pre-commit Website](https://pre-commit.com/)
- [Docker Documentation](https://docs.docker.com/)
- [EditorConfig Website](https://editorconfig.org/)

---

**ต่อไป:** Part 05 - Introduction to GitHub Actions
