# Claude Code on macOS — Sandbox Setup Guide

## 1. What is Claude Code, and how does it differ from Claude Web UI?

### Claude Web UI (claude.ai)
Claude Web UI is a browser-based chat interface. By default it is conversational only, but for certain tasks (file creation, running code, using tools) Anthropic spins up a **sandboxed Linux container in the cloud**. This is why you may see paths like `/mnt/user-data/uploads`, `/home/claude/`, or `/mnt/skills/` — these are directories inside that ephemeral cloud container, not your local machine. The container resets between sessions.

### Claude Code (CC)
Claude Code is a **CLI tool you install locally** (`npm install -g @anthropic-ai/claude-code`). It runs on **your own machine** and has direct access to your local filesystem, terminal, and shell. It calls the Claude API in the cloud for intelligence, but all execution happens locally.

### Key Differences

| Feature | Claude Web UI | Claude Code |
|---|---|---|
| Where it runs | Browser + Anthropic cloud container | Your local machine |
| Filesystem access | Sandboxed cloud container | Your actual local files |
| Internet access | Restricted/proxied | Your network |
| Tools | Built-in set (bash, file creation, web search, etc.) | Extensible via MCP servers |
| Persistence | Container resets between sessions | Persistent local environment |
| Setup required | None | `npm install` or `curl` installer |

> **Common misconception:** The Web UI is _not_ running Claude Code remotely. They are separate products. Claude Code has no cloud execution environment of its own.

---

## 2. Running Claude Code Safely on macOS — Option: Separate User Account

Because Claude Code has full access to your local shell and filesystem, running it under your main user account carries real risk (access to SSH keys, `.env` files, browser credentials, etc.).

A lightweight native macOS mitigation is to create a **dedicated standard (non-admin) user account** for Claude Code. This uses macOS's built-in per-user file permissions to isolate CC from your main user's data:

- `claudebox` (the sandbox user) cannot read `/Users/YOUR_MAIN_USER`
- It has no `sudo` / admin rights
- Your SSH keys, credentials, and personal files are invisible to it
- No extra software (Docker, VMs) required

---

## 3. CLI Steps: Create User, Install CC, and Install Skills

### Step 1: Create the sandbox user

```bash
# Create new user
sudo dscl . -create /Users/claudebox
sudo dscl . -create /Users/claudebox UserShell /bin/zsh
sudo dscl . -create /Users/claudebox RealName "Claude Sandbox"
sudo dscl . -create /Users/claudebox UniqueID 600
sudo dscl . -create /Users/claudebox PrimaryGroupID 20
sudo dscl . -create /Users/claudebox NFSHomeDirectory /Users/claudebox

# Set a password
sudo passwd claudebox

# Create the home directory
sudo createhomedir -c -u claudebox
```

> **Note:** The `createhomedir` command may print a `getcwd: cannot access parent directories` warning — this is harmless noise from sudo's shell init. If the last line reads `created (/Users/claudebox)`, it succeeded.

**Verify:**
```bash
# Confirm user and home directory exist
dscl . -read /Users/claudebox UniqueID RealName NFSHomeDirectory UserShell
ls /Users/claudebox

# Confirm claudebox is NOT an admin (should return nothing)
dscl . -read /Groups/admin GroupMembership | grep claudebox
```

---

### Step 2: Switch to the sandbox user

```bash
su - claudebox
```

All subsequent steps run **as `claudebox`**, not your main user.

---

### Step 3: Install Claude Code

```bash
# Install via official installer (no Node.js required)
curl -fsSL https://claude.ai/install.sh | bash

# Reload shell so the binary is on PATH
source ~/.zshrc

# Verify installation
claude --version
claude doctor
```

---

### Step 4: Authenticate

```bash
claude
# Follow the login prompt — it will open a browser URL
# Log in with your Claude Pro / Max / Teams account
```

---

### Step 5: Install Skills

Skills live in `~/.claude/skills/` by convention.

**Option A — Copy from your main user** (run from your main user, not claudebox):
```bash
sudo cp -r /Users/YOUR_MAIN_USER/.claude/skills /Users/claudebox/.claude/
sudo chown -R claudebox /Users/claudebox/.claude/skills
```

**Option B — Clone from a git repo** (run as claudebox):
```bash
mkdir -p ~/.claude
cd ~/.claude
git clone https://github.com/YOUR_SKILLS_REPO skills
```

---

### Step 6: Harden the sandbox (optional but recommended)

```bash
# Exit back to your main user first
exit

# Prevent claudebox from reading your main user's home directory
chmod 700 /Users/YOUR_MAIN_USER

# Remove claudebox from admin group if it appears there
sudo dseditgroup -o edit -d claudebox -t user admin
```

---

### Daily Use

```bash
# Switch to sandbox user whenever you want to run Claude Code
su - claudebox
cd /Users/claudebox/projects/my-project
claude
```
