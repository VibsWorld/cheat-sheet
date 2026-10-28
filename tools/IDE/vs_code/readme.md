# 🧩 VS Code GitHub PR Review Cheat Sheet

## 🔧 Setup
- Install **GitHub Pull Requests & Issues** extension  
  `Ctrl+Shift+X` → search → *Install*
- Sign in:  
  `Ctrl+Shift+P` → *GitHub: Sign in*

---

## 🔍 Access Pull Requests
- Sidebar → GitHub icon → *Pull Requests*
- Click a PR → *Checkout* to switch branch

---

## 🧠 Review Flow
1. Open *Files* tab → view diffs  
2. Highlight code → *Add Comment*  
3. *Start Review* → choose:
   - 💬 Comment  
   - ✅ Approve  
   - ❌ Request Changes  
4. *Submit Review* → syncs to GitHub

---

## ⚡ CLI Shortcut
```powershell
gh pr checkout <number>
