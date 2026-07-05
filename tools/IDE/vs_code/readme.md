# 📖 Reviewing GitHub Pull Requests in VS Code

This guide shows you step‑by‑step how to review PRs directly inside VS Code using the **GitHub Pull Requests and Issues** extension.

---

## 1. Install the Extension
- Open VS Code → Extensions (`Ctrl+Shift+X`)
- Search for **GitHub Pull Requests and Issues**
- Install it

📸 Snapshot: ![Install Extension](https://code.visualstudio.com/assets/docs/editor/githubpr/github-prs.png)

---

## 2. Sign in to GitHub
- Click **Sign in to GitHub** in the sidebar
- Authorize VS Code to access your repos

📸 Snapshot: ![Sign in](https://code.visualstudio.com/assets/docs/editor/githubpr/github-pr-details.png)

---

## 3. Open the Pull Requests View
- In the Activity Bar, click the **GitHub icon**
- Select **Pull Requests**
- You’ll see PRs assigned to you or your team

📸 Snapshot: ![PR List](https://code.visualstudio.com/assets/docs/editor/githubpr/github-prs.png)

---

## 4. Checkout a PR Branch
- Click on a PR → **Checkout**
- VS Code switches to that branch locally

📸 Snapshot: ![Checkout Branch](https://code.visualstudio.com/assets/docs/editor/githubpr/github-pr-details.png)

---

## 5. Review File Changes
- Open the **Files Changed** tab
- See side‑by‑side diffs of modified files

📸 Snapshot: ![Diff View](https://code.visualstudio.com/assets/docs/editor/githubpr/github-pr-diff.png)

---

## 6. Add Inline Comments
- Hover over a line → click the **+** icon
- Type your comment
- Submit it directly to GitHub

📸 Snapshot: ![Inline Comments](https://code.visualstudio.com/assets/docs/editor/githubpr/github-pr-comments.png)

---

## 7. Submit Your Review
- Click **Review Changes**
- Choose:
  - ✅ Approve
  - ❌ Request Changes
  - 💬 Comment

📸 Snapshot: ![Submit Review](https://code.visualstudio.com/assets/docs/editor/githubpr/github-pr-review.png)

---

## ⚡ Tips
- Use **GitLens** for commit history context
- Combine with **REST Client** or **Thunder Client** to test APIs while reviewing
- Press `Ctrl+Shift+P` → type `GitHub: Create Pull Request` to start new PRs

---

✅ With this workflow, you can handle PR reviews entirely inside VS Code — no need to switch to GitHub’s web UI.
