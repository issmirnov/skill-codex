# Auto-update workflow: Codex model list

This document captures everything non-obvious about
`.github/workflows/update-models.yml` — the scheduled job that keeps
`SKILL.md`'s model list in sync with the slugs the Codex CLI reports.
Read this before modifying the workflow; several of the design choices
exist because of bugs we already hit.

## What the workflow does

Every Monday at 13:00 UTC (and on `workflow_dispatch`):

1. Installs the Codex CLI (`@openai/codex`).
2. Runs `codex debug models` to enumerate available models.
3. Filters to entries whose `visibility == "list"`.
4. Patches the parenthesised model list on the *"Ask the user ... which
   model to run"* line in `plugins/skill-codex/skills/codex/SKILL.md`.
5. If `SKILL.md` actually changed, bumps the patch version in three
   files (`plugin.json`, `openpackage.yml`, `marketplace.json`) so
   `claude plugins update` will deliver the change to installed users
   (see #15 — version-stuck-on-disk problem).
6. Opens or updates a PR on the `bot/update-models` branch.

The PR is intentionally **not** auto-merged — a human reviews each
refresh.

---

## Setup requirements

### Repository setting — required, easy to miss

The workflow's `permissions:` block (`pull-requests: write`) is
**necessary but not sufficient**. GitHub enforces a repo-level policy
that blocks `GITHUB_TOKEN` from creating PRs by default, regardless of
what the workflow asks for. The first run on a new repo will fail with:

```
Error: GitHub Actions is not permitted to create or approve pull requests.
```

Enable once via UI (Settings → Actions → General → Workflow permissions
→ "Allow GitHub Actions to create and approve pull requests") or:

```bash
gh api repos/<owner>/<repo>/actions/permissions/workflow \
  --method PUT \
  --field default_workflow_permissions=write \
  --field can_approve_pull_request_reviews=true
```

This same caveat is documented in a header comment in the workflow file
itself, so anyone reviewing the YAML will see it.

---

## Pitfalls when modifying the workflow

### Bash command substitution at the `${{ }}` boundary

GitHub Actions interpolates `${{ steps.xxx.outputs.yyy }}` as **raw
text at parse time**, before bash sees the line. If the value contains
backticks and gets dropped into a double-quoted bash string, bash will
treat each backtick pair as command substitution and silently mangle
the value.

We hit this in v1.2.1's first attempt, where:

```yaml
python3 - "$skill" "${{ steps.models.outputs.slugs }}" <<'PY'
```

…with `slugs = "gpt-5.5\`, \`gpt-5.4\`, ..."` collapsed every separator
to empty, producing `gpt-5.5gpt-5.4gpt-5.4-mini...` in `SKILL.md`. Fix
in #5: stop emitting backticks from `jq`, pass the value via `env:` not
`${{ }}`. Either fix alone works; both together is defense-in-depth.

**Rule of thumb:** any time a step's output flows into another step's
shell context, prefer `env:` over `${{ }}` interpolation. Env-var
values are not re-parsed by bash for command substitution.

### Variable expansion inside the same step is *not* affected

Bash's `$slugs` (parameter expansion) does not re-interpret backticks
in expanded values — only `${{ }}` (template substitution at parse
time) does. That's why `echo "slugs=$slugs"` in the Extract step works
fine even though the value contains backticks (when we still emitted
them).

---

## Known gap: `gpt-5.3-codex-spark` is missing from auto-updates

The workflow's `codex debug models` runs **unauthenticated** on the
runner (no `~/.codex/auth.json`, no env vars). OpenAI's model
directory filters out models with `supported_in_api: false` for
unauthenticated callers. As of this writing, that filter excludes
`gpt-5.3-codex-spark` — the only "CLI-only" model in the listed set.

This is a *visibility gap*, not an *availability gap*. Users running
the skill DO go through Codex CLI, which has its own privileged path
to spark, so anyone with the slug will be able to invoke it.

If you want spark in the auto-generated list, the cleanest fix is to
hardcode it in the Patch step:

```python
slugs = raw.split(",")
if "gpt-5.3-codex-spark" not in slugs:
    slugs.append("gpt-5.3-codex-spark")  # CLI-only model, not in unauth directory response
```

Trade-off: hardcoding undermines the "fully dynamic discovery" goal,
but the set of `supported_in_api: false` slugs is small and stable.

---

## Reference: the Codex model directory

Discovered by reverse-engineering the `codex` binary's strings. Worth
documenting since it's not in OpenAI's public docs:

| | |
|---|---|
| **Endpoint** | `https://chatgpt.com/backend-api/codex/models` |
| **Required query param** | `client_version=<semver>` (e.g. `0.124.0`) |
| **Auth** | `Authorization: Bearer <ChatGPT OAuth access_token>` |
| **Token source** | `~/.codex/auth.json` (`.tokens.access_token`) |
| **Public alternative** | `https://developers.openai.com/api/docs/guides/latest-model.md` (no auth, but only names *current* latest model, not the full list) |

Manual probe:
```bash
TOKEN=$(jq -r '.tokens.access_token' ~/.codex/auth.json)
curl -H "Authorization: Bearer $TOKEN" \
  "https://chatgpt.com/backend-api/codex/models?client_version=0.124.0" \
  | jq '[.models[] | select(.visibility == "list") | .slug]'
```

### Authentication surfaces — easy to confuse

Codex CLI uses **ChatGPT account OAuth**, not the standard OpenAI
API. The two are wholly separate:

| Endpoint | Auth | Notes |
|---|---|---|
| `chatgpt.com/backend-api/codex/*` | ChatGPT OAuth (`access_token`) | Codex CLI's own backend |
| `api.openai.com/v1/*` | API key (`sk-...`) | Standard OpenAI API |

A regular API key on the Codex endpoint returns:
```
HTTP 401 — "Could not parse your authentication token"
```

This is why no amount of `OPENAI_API_KEY` secret-juggling in CI will
get you authenticated model-directory access on the runner.

---

## Things that look like good ideas but aren't

- **Adding `OPENAI_API_KEY` as a workflow secret to expose more models.**
  Doesn't work — wrong auth surface (see above).
- **Adding the OAuth `access_token` as a workflow secret.** Tokens are
  short-lived (rotate within hours via refresh flow). The weekly cron
  would break almost immediately.
- **Embedding the `refresh_token` and implementing OAuth refresh in the
  workflow.** Possible but fragile — refresh tokens themselves rotate,
  and the workflow would need to write the new token back somewhere
  (which is a chicken-and-egg with secret storage).
- **Switching to `developers.openai.com/.../latest-model.md` for
  discovery.** Public, but it only names *one* model (the current
  latest). Would regress the workflow.
- **Switching from `peter-evans/create-pull-request` to a different
  action to bypass the repo-permission block.** Doesn't help — the
  block is on `GITHUB_TOKEN`, not the action. All actions hit the same
  wall.

---

## Future work / open questions

- **Spark inclusion.** See "Known gap" above. Awaiting a maintainer
  call on whether to hardcode-as-fallback or accept the unauth subset.
- **Natural-language formatting.** The hand-written list said
  ``(`A`, `B`, or `C`)``; the bot writes ``(`A`, `B`, `C`)``. Cosmetic
  but mildly worse prose. Trivial to add an "or" before the last item
  if anyone cares.
- **Node 20 deprecation.** `actions/checkout@v4`, `actions/setup-node@v4`,
  and `peter-evans/create-pull-request@v6` will be forced to Node 24
  starting June 2026. Not blocking now, but worth a refresh PR
  sometime.
- **Upstream contribution.** Issue #21 in `skills-directory/skill-codex`
  proposes runtime model discovery instead of the hardcoded SKILL.md
  list. If that lands upstream, this entire workflow becomes
  redundant — the SKILL.md itself would tell Claude to read
  `~/.codex/models_cache.json` at invocation time.

---

## History

- **#1, #2** — initial workflow scaffolding, switched from cache-file
  read to `codex debug models` for cross-platform robustness.
- **#3** — first auto-PR, broken slug rendering (closed, not merged).
- **#4** — header comment documenting the repo-permission requirement.
- **#5** — fix for the slug-rendering bug (env var + jq cleanup).
- **#6** — first clean auto-PR.
