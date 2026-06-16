# Mintlify Docs Sync — Deploy Key Rotation

**Full runbook (canonical):** [Notion — Mintlify Docs Sync — Deploy Key Rotation](https://app.notion.com/p/3816d359062f8142af81f292c3715822)

This file is the in-repo mirror so engineers reading the workflow YAML can find the rotation procedure without leaving the editor. **If the two diverge, the Notion page wins.**

---

## Symptom

The cron workflow [`pull-from-mintlify-docs.yml`](../../.github/workflows/pull-from-mintlify-docs.yml) fails with:

```
fatal: could not read Username for 'https://github.com': No such device or address
```

…or, if SSH auth is misconfigured:

```
Permission denied (publickey).
```

## Why it can break

The workflow on `TeelOrg/docs` clones `TeelOrg/mintlify-docs` (private) over SSH using a read-only deploy key. Two halves:

| Half | Lives at | Permission |
| --- | --- | --- |
| Public key | `TeelOrg/mintlify-docs` → Settings → Deploy keys | Read-only |
| Private key | `TeelOrg/docs` → Actions secret `MINTLIFY_DOCS_DEPLOY_KEY` | Encrypted at rest |

If either half is missing/rotated/wrong, the sync stops. The private half is **write-only** from outside the runner — GitHub does not expose secret values back through any API or UI.

## Rotation procedure (90-second version)

> Prereq: admin on `TeelOrg/mintlify-docs` (or someone who has it), `gh` CLI with `repo` scope.

```bash
# 1. Generate fresh ed25519 keypair (no passphrase — CI can't type one)
KEYDIR=$(mktemp -d /tmp/teel-deploy-key.XXXXXX)
ssh-keygen -t ed25519 -C "teel-docs-sync@github-actions" -f "$KEYDIR/id_ed25519" -N "" -q
cat "$KEYDIR/id_ed25519.pub"   # send to the admin

# 2. Admin: install the public half on TeelOrg/mintlify-docs
#    Settings → Deploy keys → Add deploy key
#    Title: teel-docs-sync (read-only, from TeelOrg/docs Actions)
#    "Allow write access" UNCHECKED

# 3. Smoke-test locally
GIT_SSH_COMMAND="ssh -i $KEYDIR/id_ed25519 -o StrictHostKeyChecking=accept-new -o BatchMode=yes" \
  git ls-remote git@github.com:TeelOrg/mintlify-docs.git HEAD
# expect: a SHA prints; SSH greeting says "Hi TeelOrg/mintlify-docs!" (repo-scoped, not user-scoped)

# 4. Store the private half on TeelOrg/docs (gh reads stdin — no history exposure)
gh secret set MINTLIFY_DOCS_DEPLOY_KEY -R TeelOrg/docs < "$KEYDIR/id_ed25519"

# 5. Verify the workflow
gh workflow run "Pull from TeelOrg/mintlify-docs" -R TeelOrg/docs
gh run watch -R TeelOrg/docs

# 6. After green: admin removes the OLD deploy key; clean up local
rm -rf "$KEYDIR"
```

## Principle: rotate, don't restore

There's no path to recover a lost private key. If yours is gone (deliberately, as cleanup, or by accident), don't try to find it — generate a new keypair and run the procedure above. Rotation is cheap and gives a clean audit trail.

## Why deploy key, not PAT

- Not tied to any individual user — survives team rotations
- Scoped to one repo — minimum blast radius
- Read-only enforced at the GitHub layer (not just in the workflow)
- Avoids `TeelOrg` fine-grained PAT org-approval friction

## History

- 2026-06-15: `TeelOrg/mintlify-docs` flipped to private; bare `git clone https://...` started failing.
- 2026-06-16: Migrated to SSH deploy key. PR [#3](https://github.com/TeelOrg/docs/pull/3).
