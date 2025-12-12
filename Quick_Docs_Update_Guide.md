# ⚡ Quick Docs Update Guide (TL;DR)

Use this checklist for **any documentation change**.

---

## 1️⃣ Sync and branch

```bash
git checkout main
git pull origin main
git checkout -b docs/<section>/<short-description>
```

Example:
```bash
git checkout -b docs/direct-integration/update-endpoints
```

---

## 2️⃣ Run docs locally

```bash
mintlify dev
# or
npm run dev
```

Open: `http://localhost:3000`

---

## 3️⃣ Edit docs

- Edit `.mdx` files in **VS Code**
- Save (Cmd + S)
- Verify changes render correctly in the browser

---

## 4️⃣ Optional: archive old content

If keeping a legacy version:
```
_old-pages/<page-name>-old.mdx
```

Do **not** leave backup files in active folders.

---

## 5️⃣ Clean up before commit

- ❌ No `.DS_Store`
- ❌ No temp / backup files
- ✅ Only intended files changed

Add to `.gitignore` if missing:
```
.DS_Store
```

---

## 6️⃣ Commit

Use clear messages:
```
docs: update refund documentation
chore: archive legacy page
```

---

## 7️⃣ Push branch

```bash
git push -u origin <branch>
```

Or click **Push origin** in GitHub Desktop.

---

## 8️⃣ Open Pull Request

- **Base:** main
- **Compare:** your docs branch
- Describe **what changed** and **why**
- Review **Files changed** carefully

---

## 9️⃣ Merge

- Confirm no conflicts
- Merge PR
- Delete branch (recommended)

---

## 🔟 Publish

- If Mintlify auto-deploys from main: ✅ done
- If manual:
```bash
mintlify deploy
```

---

## ✅ Final sanity check

- Docs render locally
- No junk files
- PR diff is clean
- Live docs updated after merge

---

> **Rule of thumb:** If it renders correctly in Mintlify locally and the PR diff is clean, it's safe to merge.
