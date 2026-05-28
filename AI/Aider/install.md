Here's a complete Windows 10 setup guide. First, a quick clarification: **Ollama runs models locally on your PC** (not in the cloud). Qwen-Coder will download and run on your machine, keeping your code private.

## Step 1: Install Prerequisites

### 1. Python (3.8+)
Download from [python.org](https://python.org) → Check **"Add Python to PATH"** during installation.

### 2. Git
Download from [git-scm.com](https://git-scm.com/download/win) (accept defaults).

### 3. GitHub CLI (`gh`)
```powershell
# In PowerShell (Run as Administrator)
winget install --id GitHub.cli
```
Or download from [cli.github.com](https://cli.github.com)

### 4. Ollama
Download from [ollama.com](https://ollama.com) → Install → **Reboot** (required for Windows GPU support).

## Step 2: Install Aider

Open **PowerShell** (not CMD) and run:

```powershell
# Install pipx (isolated Python apps)
pip install --user pipx
pipx ensurepath

# Close and reopen PowerShell, then:
pipx install aider-chat

# Verify
aider --version
```

If `pipx` fails, use: `pip install --user aider-chat`

## Step 3: Download Qwen-Coder

Open PowerShell and pull the model (choose size based on your VRAM):

```powershell
# 1.5B params = runs on CPU, ~1GB RAM
ollama pull qwen2.5-coder:1.5b

# OR 7B params = needs ~6GB VRAM (GTX 1060 6GB+)
ollama pull qwen2.5-coder:7b

# OR 14B params = needs ~10GB VRAM (RTX 3060/RX 6700 XT+)
ollama pull qwen2.5-coder:14b

# Test it works
ollama run qwen2.5-coder:7b
# Type "hello" then /exit
```

## Step 4: Configure Aider with Ollama

Create a config file so you don't have to type the model every time:

**PowerShell:**
```powershell
notepad $env:USERPROFILE\.aider.conf.yml
```

**Paste this content:**
```yaml
model: ollama/qwen2.5-coder:7b
auto-commits: false
dark-mode: true
```

*(Change `7b` to whatever size you downloaded)*

## Step 5: Usage (CLI Integration)

Navigate to your GitHub repo and launch:

```powershell
cd C:\path\to\your\repo
aider
```

### Key Features with `gh` integration:

**Inside Aider**, you can run shell commands with `!` or `/run`:
```bash
# Inside Aider prompt:
!gh repo view
!gh pr list
!gh status

# Or use /run
/run gh issue list
```

**Aider automatically:**
- Sees your git context (`gh` commands work in the same directory)
- Suggests changes to files
- Lets you edit with natural language

### Example Workflow:
```powershell
# 1. Clone a repo
gh repo clone owner/repo
cd repo

# 2. Start Aider with Ollama
aider

# 3. Inside Aider:
# Add files to context
/add src\main.py

# Ask for changes (uses local Qwen-Coder)
Refactor the authentication logic to use async/await

# Run GH commands without leaving
!gh pr create --title "Refactor auth" --body "Updated auth logic"

# Exit
/quit
```

## Troubleshooting Windows 10

**"aider not recognized"**
```powershell
# Add to PATH manually
$env:Path += ";$env:USERPROFILE\.local\bin"
# Or if using pip: $env:Path += ";$env:USERPROFILE\AppData\Roaming\Python\Python3x\Scripts"
```

**Ollama connection refused**
Ensure Ollama is running in system tray (check hidden icons). Or start it:
```powershell
ollama serve
```

**Slow performance**
If you don't have a GPU, use smaller models:
```yaml
# In .aider.conf.yml
model: ollama/qwen2.5-coder:1.5b
```

**PowerShell Execution Policy error**
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

## Alternative: Using Aider with actual Cloud LLMs

If you *also* want to use Claude/GPT-4 for complex tasks (switching between local and cloud), just override the model flag:

```powershell
# Local (free, private)
aider --model ollama/qwen2.5-coder:7b

# Cloud (powerful, requires API key)
aider --model claude-3-5-sonnet-20240620
$env:ANTHROPIC_API_KEY="your-key-here"
```

**Pro tip**: Use Windows Terminal (from Microsoft Store) for better Unicode support and copy/paste when Aider displays code diffs.
