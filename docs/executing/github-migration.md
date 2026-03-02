# GitHub Migration Plan: `hejijunhao/photon` → `kaminocorp/photon`

Move the Photon repository from the personal account to the Kamino Corp organization.

---

## Pre-Migration

### 1. Create the organization (if not already done)

- Ensure `kaminocorp` org exists at https://github.com/kaminocorp
- Verify you have **Owner** role in the org

### 2. Check for blockers

- [ ] No open pull requests that would lose review context (PR URLs redirect, but CI status checks re-bind to the new location)
- [ ] No in-flight CI runs (`Actions` tab is idle)
- [ ] Note any repo-level **Secrets** — they transfer with the repo, but verify after

---

## Migration

### 3. Transfer the repository

1. Go to https://github.com/hejijunhao/photon/settings
2. Scroll to **Danger Zone** → **Transfer repository**
3. Select `kaminocorp` as the new owner
4. Confirm the transfer

> GitHub automatically sets up redirects from `hejijunhao/photon` → `kaminocorp/photon` for all git operations and web URLs. Existing clones continue to work.

---

## Post-Migration

### 4. Update PyPI Trusted Publishing (critical)

The OIDC trusted publisher is bound to the old repo path. Without this, the next tagged release will fail to publish to PyPI.

1. Go to https://pypi.org/manage/project/photon-imager/settings/publishing/
2. Remove the old publisher (`hejijunhao/photon`)
3. Add a new trusted publisher:
   - **Owner:** `kaminocorp`
   - **Repository:** `photon`
   - **Workflow:** `pypi.yml`
   - **Environment:** `pypi`

### 5. Update hardcoded URLs in the codebase

These files reference `hejijunhao/photon` and need updating to `kaminocorp/photon`:

| File | Line(s) | What |
|---|---|---|
| `Cargo.toml` | 10 | `repository` field |
| `pyproject.toml` | 25–26 | `Repository` and `Issues` URLs |
| `README.md` | 64 | `git clone` URL |
| `README.md` | 269 | `photon-core` git dependency example |
| `docs/distribution.md` | 82 | `git clone` URL |

### 6. Update local git remote

```bash
git remote set-url origin https://github.com/kaminocorp/photon.git
```

### 7. Verify CI workflows

- [ ] Push a commit (the URL updates from step 5 are a good candidate) and confirm CI passes
- [ ] `GITHUB_TOKEN` is auto-provisioned — no action needed
- [ ] GitHub-hosted runners (`macos-14`, `ubuntu-latest`, `ubuntu-24.04-arm`) work identically under orgs

### 8. Verify GitHub Releases

- [ ] Existing releases are preserved and accessible at `kaminocorp/photon/releases`
- [ ] Tag a test release (or wait for the next real one) to confirm `release.yml` workflow works

### 9. Verify PyPI publishing

- [ ] After step 4, trigger a PyPI publish (tag a version or manually run `pypi.yml`) to confirm trusted publishing works under the new org

---

## Rollback

If something goes wrong, you can transfer the repo back:
1. Go to https://github.com/kaminocorp/photon/settings → **Transfer repository** → back to `hejijunhao`
2. Revert the PyPI trusted publisher config

---

## What won't break

- **Existing git clones** — GitHub redirects cover `fetch`/`push` indefinitely
- **Cargo builds** — the `repository` field is metadata-only, not used for compilation
- **Stars, watchers, issues, PRs** — all transfer with the repo
- **GitHub Actions workflows** — no hardcoded account references in CI configs
