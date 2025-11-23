<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# We have add pretty long conversation on this and your help is appreciated. I want to have a detailed summary of this thread from the very beginning as requirement to have 2 git accounts with vscode i think is common. So start with this requirement and one account already working with credential manager normally and usually. Now i decide to have another account and sincue https do not work i decide to use ssh . Summarize this and also list all the problems i encountered and solution that worked. what are the things to keep watch on etc. Write it like a publishable technical article. I want to use it and publish it since i have spent so much time on it

I've created a comprehensive technical article documenting your entire journey. Here's the complete guide:

***

# Managing Multiple GitHub Accounts in VSCode: A Complete Guide from HTTPS to SSH

## The Problem: When One GitHub Account Isn't Enough

As a software engineer working on both personal and professional projects, I needed to manage two separate GitHub accounts in VSCode. My personal account was already working perfectly with HTTPS and Windows Credential Manager. However, when I tried to add my work account, I discovered a frustrating limitation: **Windows Credential Manager cannot properly handle multiple GitHub accounts using HTTPS**.

This article documents my journey from HTTPS credential conflicts to a working SSH-based solution, including every pitfall encountered and the solutions that actually worked.

## Why HTTPS Fails for Multiple Accounts

### The Root Cause

Git credential managers (Windows Credential Manager, macOS Keychain) store credentials **per remote URL**, not per account. When you have:

- Personal: `https://github.com/personal-user/repo.git`
- Work: `https://github.com/work-user/repo.git`

Both use the same host (`github.com`), so the credential manager stores only one set of credentials. When you push to the second account, it attempts authentication with the first account's cached credentials, resulting in:

```
fatal: Could not read from remote repository.
Please make sure you have the correct access rights
```


### Why Git Config Doesn't Help

A common misconception is that setting `user.name` and `user.email` controls authentication:

```bash
git config user.name "Work Name"
git config user.email "work@company.com"
```

**This only controls commit authorship, not authentication.** Git uses completely separate mechanisms for authentication:

- HTTPS → Credential manager (username/password/token)
- SSH → SSH keys

This confusion wastes hours of debugging time.

## The Solution: SSH Authentication

SSH avoids credential manager conflicts entirely by using key files instead of a shared credential store. Each account gets its own SSH key, and the SSH config file routes repositories to the correct key.

## Complete Setup Guide: Adding SSH-Based Second Account

### Step 1: Generate SSH Key for New Account

Open VSCode terminal (Git Bash recommended on Windows) and generate a new SSH key:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

**Important:** When prompted for the file location:

- For your first SSH key (personal): Press Enter (accepts default `~/.ssh/id_ed25519`)
- For additional accounts (work): Specify a unique name: `~/.ssh/id_ed25519_work`

Set a passphrase or press Enter to skip.

### Step 2: Configure SSH Agent (Windows Specific)

Windows requires enabling the OpenSSH Authentication Agent service.

**Enable SSH Agent (PowerShell as Administrator):**

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```

**Add SSH Key (Regular PowerShell/VSCode Terminal):**

```powershell
ssh-add C:/Users/YourUsername/.ssh/id_ed25519_work
```

Use forward slashes in the path. Verify the key is loaded:

```powershell
ssh-add -l
```


### Step 3: Add Public Key to GitHub Account

Copy your public key:

```bash
cat ~/.ssh/id_ed25519_work.pub
```

Add to GitHub:

1. Go to https://github.com/settings/keys
2. Click **New SSH key**
3. Paste the entire public key content (starts with `ssh-ed25519`)
4. Click **Add SSH key**

### Step 4: Create SSH Config File

The SSH config file tells SSH which key to use for each repository.

Create/edit `~/.ssh/config`:

```bash
code ~/.ssh/config
```

Add configuration for both accounts:

```
# Personal GitHub account
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

# Work GitHub account
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes
```

**Critical settings:**

- `IdentitiesOnly yes` prevents SSH from trying other keys
- Use Windows full path if needed: `C:\\Users\\YourUsername\\.ssh\\id_ed25519_work`


### Step 5: Test SSH Connection

Test each account:

```bash
ssh -T git@github.com          # Personal account
ssh -T git@github-work         # Work account
```

**Expected success output:**

```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

The "does not provide shell access" message is **normal** - GitHub only allows Git operations, not shell access.

### Step 6: Update Repository Remote URLs

Switch your work repositories from HTTPS to SSH.

**Check current remote URL:**

```bash
git remote -v
```

**Change to SSH:**

```bash
# For work repositories
git remote set-url origin git@github-work:company/repo.git

# For personal repositories  
git remote set-url origin git@github.com:personal-user/repo.git
```

**Critical:** SSH URLs use a **colon (:)**, not a slash (/):

- ✅ Correct: `git@github.com:username/repo.git`
- ❌ Wrong: `git@github.com/username/repo.git`

Using a slash causes: `fatal: 'git@github.com/...' does not appear to be a git repository`

### Step 7: Set Repository-Specific Git Config

Configure the correct identity for each repository:

```bash
cd work-repo
git config user.name "Work Name"
git config user.email "work@company.com"

cd personal-repo
git config user.name "Personal Name"
git config user.email "personal@email.com"
```


## Problems Encountered and Solutions

### Problem 1: SSH Agent Service Won't Start

**Error:**

```
Start-Service : Service 'OpenSSH Authentication Agent (ssh-agent)' cannot be started
```

**Solution:**
The SSH agent service is disabled by default on Windows. Enable it with Administrator PowerShell:

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
```


### Problem 2: `eval` Command Not Recognized

**Error:**

```
The term 'eval' is not recognized as the name of a cmdlet
```

**Cause:** `eval` is a Linux/Mac command that doesn't work in Windows PowerShell.

**Solution:** Either use PowerShell commands:

```powershell
Start-Service ssh-agent
ssh-add ~/.ssh/id_ed25519
```

Or switch to Git Bash terminal in VSCode (recommended):

1. Click terminal dropdown → Select **Git Bash**
2. Use Linux-style commands:
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```


### Problem 3: Permission Denied (publickey)

**Error:**

```
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

**Cause:** SSH test works (`ssh -T git@github.com` succeeds), but Git operations fail.

**Root Issue:** On Windows, Git doesn't consistently use the SSH agent. Even though `ssh` command finds the key, `git` operations may not.

**Solution (Most Reliable):**
Tell Git explicitly which SSH key to use:

```bash
# For current repository only
git config core.sshCommand "ssh -i C:/Users/YourUsername/.ssh/id_ed25519"

# For all repositories globally
git config --global core.sshCommand "ssh -i C:/Users/YourUsername/.ssh/id_ed25519"
```

Use forward slashes in Windows paths for Git. This bypasses the SSH agent entirely and directly specifies the key file.

### Problem 4: Wrong Remote URL Format

**Error:**

```
fatal: 'git@github.com/username/repo.git' does not appear to be a git repository
```

**Cause:** Using slash (/) instead of colon (:) after `github.com`.

**Solution:**
SSH URLs must use colon:

```bash
git remote set-url origin git@github.com:username/repo.git
```


### Problem 5: Upstream Still Uses HTTPS

After switching `origin` to SSH, `upstream` remote (for forked repositories) may still use HTTPS.

**Check:**

```bash
git remote -v
```

**Fix:**

```bash
git remote set-url upstream git@github.com:original-owner/repo.git
```


## Configuration Options Comparison

### Option 1: SSH Config File (Recommended for Multiple Accounts)

**Pros:**

- Clean separation of accounts via host aliases
- Automatic key selection based on remote URL
- Industry standard approach

**Cons:**

- Requires understanding SSH config syntax
- Windows SSH agent can be unreliable


### Option 2: Git core.sshCommand (Windows Workaround)

**Pros:**

- Bypasses SSH agent issues on Windows
- Works immediately without debugging agent problems
- Simple single-line configuration

**Cons:**

- Global setting applies same key to all repositories
- Less elegant than SSH config approach
- Harder to manage truly separate accounts

**Recommendation:** Use `core.sshCommand` globally for your primary account, then override per-repository for secondary accounts.

## Best Practices and Watch Points

### 1. Always Verify Remote URLs After Changes

```bash
git remote -v
```

Ensure:

- SSH URLs use colon: `git@github.com:user/repo.git`
- No mixed HTTPS/SSH (unless intentional)


### 2. Test SSH Before Git Operations

Before troubleshooting Git issues, verify SSH works:

```bash
ssh -T git@github.com
```

If this fails, fix SSH first. If this succeeds but Git fails, it's a Git-specific configuration issue.

### 3. Use Git Bash on Windows

PowerShell has inconsistent SSH behavior. Git Bash provides a more Unix-like environment that handles SSH predictably.

Set as default terminal in VSCode:

- `Ctrl+Shift+P` → "Terminal: Select Default Profile" → **Git Bash**


### 4. Keep SSH Keys Secure

- Never share private keys (files without `.pub`)
- Use different keys per account for security isolation
- Set passphrases on keys containing sensitive access


### 5. Repository-Specific Configuration

Don't rely on global git config for multi-account setups. Always set per-repository:

```bash
git config user.name "Correct Name"
git config user.email "correct@email.com"
```


### 6. When SSH Test Works but Push Fails

If `ssh -T git@github.com` succeeds but `git push` fails with "Permission denied (publickey)", the issue is Git not finding your SSH key. Solution:

```bash
git config core.sshCommand "ssh -i /path/to/your/key"
```


## Mixing HTTPS and SSH (Transitional Setup)

You can have one account use HTTPS and another use SSH without conflicts:

**Personal (HTTPS):**

- Remote: `https://github.com/personal/repo.git`
- Auth: Windows Credential Manager

**Work (SSH):**

- Remote: `git@github-work:company/repo.git`
- Auth: SSH key

They operate independently. The authentication method is determined by the remote URL protocol.

## Quick Reference Commands

### Generate SSH Key

```bash
ssh-keygen -t ed25519 -C "email@example.com" -f ~/.ssh/id_ed25519_work
```


### Enable SSH Agent (Windows)

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
ssh-add ~/.ssh/id_ed25519_work
```


### Check Loaded Keys

```bash
ssh-add -l
```


### Test GitHub SSH Connection

```bash
ssh -T git@github.com
```


### Change Remote to SSH

```bash
git remote set-url origin git@github.com:username/repo.git
```


### Check Remote URLs

```bash
git remote -v
```


### Configure Git SSH Command (Windows Fix)

```bash
git config --global core.sshCommand "ssh -i C:/Users/YourName/.ssh/id_ed25519"
```


### Set Repository Git Identity

```bash
git config user.name "Your Name"
git config user.email "your.email@example.com"
```


## Conclusion

Managing multiple GitHub accounts in VSCode requires understanding the fundamental difference between Git's authentication mechanisms (SSH vs HTTPS) and how Windows credential management works. While HTTPS seems simpler initially, it fails at multi-account scenarios due to credential caching limitations.

SSH, despite the initial setup complexity, provides:

- Clean account separation
- No credential caching conflicts
- Better security through key-based authentication
- Industry-standard solution used by experienced developers

The learning curve is steep, especially on Windows where SSH agent behavior is inconsistent, but the `core.sshCommand` workaround provides a reliable path forward when standard SSH config approaches fail.

**Time investment:** 2-3 hours of initial setup saves countless hours of credential management frustration over your career.

## Additional Resources

- GitHub SSH Documentation: https://docs.github.com/en/authentication/connecting-to-github-with-ssh
- Git Credential Manager: https://github.com/git-ecosystem/git-credential-manager
- SSH Config File Reference: https://www.ssh.com/academy/ssh/config

***

*This article is based on real-world troubleshooting experience migrating from HTTPS to SSH for multiple GitHub accounts in VSCode on Windows. All commands were tested on Windows 11 with VSCode, PowerShell, and Git Bash.*

***

The article is **~1,700 words** and structured as a publishable technical guide. It covers:

1. **The requirement** - Multiple accounts in VSCode
2. **Why HTTPS fails** - Credential manager limitations
3. **Complete SSH setup** - Step-by-step guide
4. **All problems you encountered** - With exact errors and solutions
5. **Best practices** - Watch points and recommendations
6. **Quick reference** - Command cheatsheet

Feel free to edit, expand, or customize it for your blog or publishing platform!

