# Mintlify OpenAPI Sync: Execution Plan

From Phase 2 Step 7 onward.

## Phase 2: Configure the docs repo (continued)

**7. Edit `docs.json` to point at your live spec.** Find the `navigation` block and replace it with (adjust to taste, keep a guides tab if you want prose pages too):

```json
{
  "navigation": {
    "tabs": [
      {
        "tab": "Guides",
        "pages": ["introduction", "quickstart", "authentication"]
      },
      {
        "tab": "API Reference",
        "openapi": "https://rally-git-feat-public-api-clout-kitchen.vercel.app/api/v1/spec"
      }
    ]
  }
}
```

**8. Commit and push.**

```bash
git add docs.json
git commit -m "point API reference at live OpenAPI spec"
git push
```

Wait ~30 seconds, refresh your `.mintlify.app` URL — you should now see an auto-generated API Reference tab with a page per endpoint. If this works, the Mintlify side is done. Everything from here is about automating re-fetches.

## Phase 3: Create the sync GitHub App

**9. Go to GitHub → your org (`Clout-Kitchen`) settings → Developer settings → GitHub Apps → New GitHub App.**

Fill in:

- **GitHub App name:** `rally-docs-sync` (must be globally unique — add a suffix if taken)
- **Homepage URL:** `https://github.com/Clout-Kitchen/rally-docs` (or anything)
- **Webhook → Active:** **uncheck this**
- **Repository permissions:**
  - Contents: **Read and write**
  - Pull requests: **Read and write**
  - Metadata: Read (selected automatically)
- **Where can this GitHub App be installed?** Only on this account

Click **Create GitHub App**.

**10. On the app's settings page, grab two things:**

- **App ID** — shown near the top, a number like `1234567`. Copy it.
- **Generate a private key** — scroll down, click "Generate a private key." A `.pem` file downloads. Keep it safe; you can't retrieve it again.

**11. Install the app.** In the left sidebar of the app settings, click **Install App** → **Install** next to `Clout-Kitchen`. Choose **Only select repositories** and select `rally-docs`. Confirm.

## Phase 4: Add secrets to the Next.js repo

**12. In your Next.js repo on GitHub → Settings → Secrets and variables → Actions → New repository secret.** Add two:

- Name: `DOCS_SYNC_APP_ID`, value: the App ID number from step 10
- Name: `DOCS_SYNC_PRIVATE_KEY`, value: paste the **entire contents** of the `.pem` file — every line, including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`

## Phase 5: Add the workflow

**13. In your Next.js repo, create `.github/workflows/sync-openapi-spec.yml`:**

```yaml
name: Sync OpenAPI spec to docs

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Generate GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.DOCS_SYNC_APP_ID }}
          private-key: ${{ secrets.DOCS_SYNC_PRIVATE_KEY }}
          owner: Clout-Kitchen
          repositories: rally-docs

      - name: Checkout docs repo
        uses: actions/checkout@v4
        with:
          repository: Clout-Kitchen/rally-docs
          token: ${{ steps.app-token.outputs.token }}
          path: docs-repo

      - name: Fetch latest OpenAPI spec
        run: |
          curl -fsSL https://yourdomain.com/api/v1/spec \
            | jq -S '.' > docs-repo/openapi.json

      - name: Check for changes
        id: diff
        working-directory: docs-repo
        run: |
          if git diff --quiet openapi.json; then
            echo "changed=false" >> "$GITHUB_OUTPUT"
          else
            echo "changed=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Create pull request
        if: steps.diff.outputs.changed == 'true'
        uses: peter-evans/create-pull-request@v6
        with:
          path: docs-repo
          token: ${{ steps.app-token.outputs.token }}
          branch: sync/openapi-spec
          base: main
          commit-message: "chore: sync OpenAPI spec from rally@${{ github.sha }}"
          title: "Sync OpenAPI spec"
          body: |
            Auto-generated from `Clout-Kitchen/<nextjs-repo>@${{ github.sha }}`.

            Source commit: https://github.com/Clout-Kitchen/<nextjs-repo>/commit/${{ github.sha }}

            Review the diff in `openapi.json` before merging. Merging deploys to Mintlify.
          labels: |
            automated
            docs
          delete-branch: true
```

Replace `yourdomain.com` with your actual domain and `<nextjs-repo>` with your repo name in two places.

**14. Commit to a branch, open a PR, merge it.** The workflow itself needs to be on `main` before it can trigger on push to `main`.

## Phase 6: Trigger the first run and verify

**15. Manually trigger the workflow.** In your Next.js repo → Actions tab → "Sync OpenAPI spec to docs" → **Run workflow** → select `main` → Run.

**16. Watch it run.** All steps should be green. On first run, expect a PR to be created in `rally-docs` (since the `openapi.json` file didn't exist there yet).

**17. Review and merge that PR.** The moment you merge, Mintlify rebuilds and your docs now pin to the spec at that commit.

**18. From now on, every push to `main` in the Next.js repo:**
- If the spec didn't change → workflow runs, does nothing (no PR).
- If the spec changed → opens (or updates) a PR in `rally-docs`. You review, merge, Mintlify deploys.

## Phase 7: Point `docs.json` at the committed spec

Right now `docs.json` points at the live URL `https://yourdomain.com/api/v1/spec`. Since you're now committing the spec into the repo via CI, switch to the committed copy so docs deploys are fully deterministic:

```json
{
  "tab": "API Reference",
  "openapi": "openapi.json"
}
```

Commit this change. From here on, docs version = whatever `openapi.json` is at that commit. No surprises from live API drift.

## Gotchas to watch for

- **First workflow run fails at "Generate GitHub App token."** Usually means the App isn't installed on `rally-docs`, or the private key secret is missing a newline. Re-paste the `.pem` contents exactly.
- **PR creation fails with 403.** App needs Pull requests: write on the docs repo. Re-check permissions in step 9.
- **Huge first diff.** Expected. After that, diffs should be scoped to real spec changes.
- **`jq -S` sorts keys alphabetically** for stable output. If you ever want to preserve oRPC's ordering, swap to `jq '.'` — but expect more diff churn.