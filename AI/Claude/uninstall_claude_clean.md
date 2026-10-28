
# Guide: Clean Uninstall of Claude Code (Windows)

This guide provides the steps to completely remove Claude Code from your system, including all cached token usage, logs, credentials, and configuration files.

## ⚠️ Warning
This process is **irreversible**. It will delete:
- Your login session and authentication tokens.
- All conversation history.
- Project memories and custom settings.
- Local caches and telemetry logs.

---

## Step 1: Uninstall the CLI Package
Claude Code was installed as a global npm package. To remove the executable, run the following command in your terminal:

```bash
npm uninstall -g @anthropic-ai/claude-code
```

## Step 2: Wipe Global Configuration and Data
The CLI stores all user-specific data in a hidden directory in your user profile. Deleting this folder removes all tokens, caches, and logs.

### Option A: Using PowerShell (Recommended)
Run the following command to recursively delete the configuration folder:

```powershell
Remove-Item -Path "$HOME\.claude" -Recurse -Force
```

### Option B: Using File Explorer (Manual)
1. Open File Explorer and navigate to `C:\Users\vibs2\`.
2. Ensure "Hidden items" are visible (**View** $\rightarrow$ **Show** $\rightarrow$ **Hidden items**).
3. Locate the folder named `.claude`.
4. Right-click the folder and select **Delete**.

## Step 3: Verification
To confirm that the uninstall was successful, open a new terminal window and type:

```bash
claude
```

The system should return an error indicating that the command `claude` is not recognized.

---

## Summary of Deleted Components
- **Authentication:** `.credentials.json`
- **Usage Statistics:** `stats-cache.json`
- **Conversation History:** `history.jsonl`
- **Local State:** `cache/`, `sessions/`, `projects/`, and `telemetry/` directories.
- **User Settings:** `settings.json` and `settings.local.json`.
