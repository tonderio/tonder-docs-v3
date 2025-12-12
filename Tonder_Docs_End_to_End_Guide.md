# 📘 Tonder Documentation Update Guide
_End-to-End Workflow: Local → GitHub → Live Docs_

## Purpose
This guide describes the standard process to update Tonder documentation locally and publish changes safely to production.

---

## Tooling
- VS Code + Mintlify extension
- Mintlify CLI
- Git
- GitHub Desktop (recommended)
- Node.js + npm

---

## 1. Initial Setup (one-time)

```bash
npm install -g mintlify
git clone git@github.com:<org>/tonder-docs-v3.git
cd tonder-docs-v3
```

---

## 2. Start a Docs Update

### Sync main
```bash
git checkout main
git pull origin main
```

### Create a feature branch
```bash
git checkout -b docs/<section>/<short-description>
```

Example:
```bash
git checkout -b docs/direct-integration/update-endpoints
```

---

## 3. Run Docs Locally

```bash
mintlify dev
# or
npm run dev
```

Open:
```
http://localhost:3000
```

---

## 4. Edit Content

- Edit `.mdx` files in VS Code
- Save files (Cmd + S)
- Preview changes live in browser

---

## 5. Archive Legacy Pages (optional)

Move outdated docs to:
```
_old-pages/
```

Example:
```
_old-pages/choose-integration-old.mdx
```

---

## 6. Clean Up Artifacts

Never commit `.DS_Store` files.

Add to `.gitignore`:
```gitignore
.DS_Store
```

---

## 7. Review Changes

- Open Source Control in VS Code
- Verify only intended files changed
- Check Mintlify preview

---

## 8. Commit

Use clear commit messages:
```text
docs: clarify refund limitations
chore: archive legacy page
```

---

## 9. Push Branch

```bash
git push -u origin <branch>
```

Or click **Push origin** in GitHub Desktop.

---

## 10. Open Pull Request

- Base: `main`
- Compare: feature branch
- Add extended description explaining scope and intent

---

## 11. Merge

- Review Files Changed
- Merge PR
- Delete branch

---

## 12. Sync Local Main

```bash
git checkout main
git pull origin main
git branch -d <branch>
```

---

## 13. Publish

If auto-deploy is enabled:
- Merge triggers deployment

Manual:
```bash
mintlify deploy
```

---

## Final Checklist

- [ ] Preview verified locally
- [ ] No `.DS_Store`
- [ ] Correct files changed
- [ ] PR merged
- [ ] Docs live

