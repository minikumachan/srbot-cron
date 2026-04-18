# SRBOT Cron Runner (public)

Public companion repo that executes SRBOT's offline-safe cron jobs
(PayPay token refresh + GoFile keep-alive) using free unlimited
GitHub Actions minutes.

The private source repo is never exposed — its path is resolved at
runtime from repo secrets.
A **GitHub App** gives this public workflow a **10-minute ephemeral
token** per run to check out the private source, then the token dies.

## Why this layout

| Goal                                    | Solved by                        |
| --------------------------------------- | -------------------------------- |
| Unlimited free GHA minutes              | This repo is public              |
| Private source never leaves its repo    | Only ephemeral token, no PAT     |
| Token rotation overhead = 0             | GitHub App auto-mints per run    |
| Easy revocation if anything leaks       | Revoke App installation → done   |

## One-time setup (≈15 minutes)

### 1. Create the public repo (2 min)

1. On GitHub, create a new public repo (suggested name: `srbot-cron`).
2. Copy the `.github/` directory from this template into the repo root.
3. Commit + push.

### 2. Create a GitHub App (5 min)

1. Open <https://github.com/settings/apps/new>.
2. **App name**: `SRBOT Cron` (must be unique on GitHub, append digits if taken).
3. **Homepage URL**: anything — your bot's main page is fine.
4. **Webhook**: uncheck "Active". We don't use webhooks.
5. **Repository permissions**:
   - **Contents**: `Read-only`
   - **Metadata**: `Read-only` (required by default)
6. Leave account permissions and all other scopes untouched.
7. **Where can this GitHub App be installed?**: "Only on this account".
8. Click **Create GitHub App**.
9. On the created app page, scroll to **Private keys** → **Generate a private key**.
   Save the downloaded `.pem` file — you need its full contents in step 4.
10. Note the **App ID** shown near the top of the settings page.

### 3. Install the App on the private repo (1 min)

1. On the App page, click **Install App** in the left sidebar.
2. Install on your account.
3. Choose **Only select repositories** → select your private source repo.
4. Click **Install**.

### 4. Register secrets on the public repo (5 min)

Public repo → **Settings → Secrets and variables → Actions → New repository secret**:

| Name                 | Value                                                                                  |
| -------------------- | -------------------------------------------------------------------------------------- |
| `PRIVATE_REPO`       | Full `owner/name` path of the private source repo                                     |
| `PRIVATE_REPO_NAME`  | Just the repo name (no owner); used by the GitHub App token install scope             |
| `APP_ID`             | Numeric App ID from step 2.10                                                          |
| `APP_PRIVATE_KEY`    | **Full** contents of the `.pem` file including the `-----BEGIN…` / `-----END…` lines  |
| `DATABASE_URL`       | Neon connection string (same as Koyeb env var)                                         |
| `CIPHER_KEY`         | 256-bit hex, same as production                                                        |
| `CRYPTO_SECRET_KEY`  | 256-bit hex, same as production                                                        |
| `PROXY_WORKER_URL`   | Cloudflare Worker URL (same as Koyeb)                                                  |
| `PROXY_SECRET`       | Shared HMAC secret (same as Koyeb + Worker)                                            |

### 5. Verify (2 min)

1. Public repo → **Actions** tab → pick one of the workflows.
2. Click **Run workflow** → **Run workflow** (triggers a manual run).
3. Expect ✅ within ~1 minute. If red, open the run logs to see which
   step failed (most common: a typo in secret names or missing
   `APP_PRIVATE_KEY` header lines).

## Rotating credentials

| Thing to rotate          | How                                                            |
| ------------------------ | -------------------------------------------------------------- |
| App private key          | App settings → Private keys → Generate → update repo secret    |
| Revoke App entirely      | Account settings → Applications → Installed → Revoke           |
| `PROXY_SECRET`           | Update Koyeb env, Worker secret, AND this repo's secret        |
| `DATABASE_URL`           | Update Koyeb + here together                                   |

No PAT to rotate — that's the point.

## File index

```
.github/workflows/
  paypay-refresh.yml       # 30-min cron — refresh PayPay access_tokens
  srcloud-keepalive.yml    # daily cron  — HEAD sweep on GoFile files
```

Both workflows share the same App-token-mint + private-checkout steps;
the only difference is the script they run and the cron cadence.
