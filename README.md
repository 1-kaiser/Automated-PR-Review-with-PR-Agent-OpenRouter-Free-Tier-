# Automated PR Review with PR-Agent + OpenRouter (Free Tier)

A complete setup guide for running AI-powered pull request reviews on GitHub using [PR-Agent](https://github.com/qodo-ai/pr-agent) and free models via [OpenRouter](https://openrouter.ai).

---

## 1. Get an OpenRouter API key

1. Go to [openrouter.ai](https://openrouter.ai) and sign up.
2. Navigate to **Keys** → **Create Key**.
3. Copy the key — it starts with `sk-or-v1-...`.

---

## 2. Store the key as a GitHub secret

1. In your repo: **Settings → Secrets and variables → Actions → New repository secret**.
2. Name: `OPENROUTER_KEY`
3. Value: paste the raw key (no quotes, no extra spaces/newlines).
4. Save.

> ⚠️ Repository secrets, not Environment secrets — unless your workflow explicitly declares an `environment:`, environment-scoped secrets won't be visible to the job.

---

## 3. Pick a free model

Free models on OpenRouter are tagged with `:free` in their slug, and the free lineup **rotates** — sometimes weekly. Always double-check [openrouter.ai/models](https://openrouter.ai/models) (filter by "Free") before trusting a hardcoded model name long-term.

Known-good options as of writing:
- `openai/gpt-oss-120b:free` — strong all-round coding/review quality
- `qwen/qwen3-next-80b-a3b-instruct:free` — good multi-turn reasoning
- `nvidia/nemotron-3-super-120b-a12b:free` — large context window for big diffs

---

## 4. Create `.pr_agent.toml`

In your repo's **root folder**, create `.pr_agent.toml`:

```toml
[config]
model="openrouter/openai/gpt-oss-120b:free"
fallback_models=["openrouter/nvidia/nemotron-3-super-120b-a12b:free"]
custom_model_max_tokens=127000
```

- `model` — primary model PR-Agent uses.
- `fallback_models` — backups if the primary is rate-limited or down.
- `custom_model_max_tokens` — **required** for models PR-Agent doesn't recognize internally (most free OpenRouter models). Without this, PR-Agent refuses to even attempt the API call, failing with a `MAX_TOKENS` error before reaching OpenRouter.

---

## 5. Create the workflow file

Create `.github/workflows/pr-agent.yml`:

```yaml
name: Automated PR Review and Testing
on:
  pull_request:
    types: [opened, reopened, ready_for_review, synchronize]
  issue_comment:
jobs:
  pr_agent_job:
    if: ${{ github.event.sender.type != 'Bot' }}
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
      contents: write
      checks: write
    name: Run pr agent on every pull request, respond to user comments
    steps:
      - name: PR Agent action step
        id: pragent
        uses: the-pr-agent/pr-agent@main
        env:
          OPENAI_KEY: ${{ secrets.OPENROUTER_KEY }}
          OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Both env vars matter:**
- `OPENAI_KEY` — some internal PR-Agent code paths still check this.
- `OPENROUTER_API_KEY` — this is the one litellm (PR-Agent's model router) actually uses to authenticate calls to any `openrouter/...` model. **Without this specific variable name, requests are sent with no Authorization header at all**, and OpenRouter returns a confusing `401: No cookie auth credentials found` error — not a normal "invalid key" error.

---

## 6. Commit and push

```bash
git add .pr_agent.toml .github/workflows/pr-agent.yml
git commit -m "Add automated PR review with PR-Agent + OpenRouter"
git push
```

---

## 7. Test it

- **Open a brand-new pull request.** `opened` is one of the default trigger events PR-Agent auto-runs on.
- Wait 1–2 minutes, then check the PR's **Conversation** tab for the review comment. Inline suggestions (if any) appear under **Files changed**.

> Note: pushing a new commit to an *already-open* PR fires a `synchronize` event, which PR-Agent **skips by default** (to avoid re-reviewing on every commit and burning through free-tier rate limits). You'll see `Skipping action: synchronize` in the logs — that's expected, not an error.

### To manually trigger a review on an existing PR
Comment directly on the PR:
```
/review
```
or `/describe`, `/improve` — PR-Agent responds within a minute or two.

### To make it auto-run on every new commit too
Add this to `.pr_agent.toml`:
```toml
[github_action_config]
pr_actions = ["opened", "reopened", "synchronize"]
```

---

## Troubleshooting reference

| Symptom in logs | Cause | Fix |
|---|---|---|
| `Model ... is not defined in MAX_TOKENS ... and no custom_model_max_tokens is set` | PR-Agent doesn't recognize the model internally | Add `custom_model_max_tokens` to `.pr_agent.toml` |
| `OpenrouterException - No cookie auth credentials found` (401) | No API key was actually sent — empty Authorization header | Add `OPENROUTER_API_KEY` env var (not just `OPENAI_KEY`) |
| `Incorrect API key provided` | A key was sent but to the wrong endpoint/format | Check the key value itself for typos, extra spaces, or quotes |
| `Skipping action: synchronize` | Normal — synchronize isn't auto-reviewed by default | Comment `/review` manually, or enable it in `pr_actions` |
