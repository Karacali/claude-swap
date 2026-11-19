# Our changes to claude-swap (fork Karacali/claude-swap)

Inventory of every local improvement, where it lives, and what happened to it in the
2026-11-19 rebase onto upstream `realiti4/claude-swap` `origin/main` `7fecb86` (1.2.0b1).
The live install (`~/.local/bin/cswap`, `cswap-auto.service`) is built from branch
**`feat/hard-threshold-rebase-20261119`** (this worktree); `~/.local/bin/cswap-guard` re-installs it
from here if an upgrade replaces it. The pre-rebase install is backed up in
`~/.local/share/cswap-backup-20261119/`.

## Sources

| Source | Base | Contents |
|---|---|---|
| `fix/user-scoped-api-keys` `1bcdb40` | upstream (243 behind) | sk-ant-usr keys |
| `fix/api-key-last-resort-at-limit` `7c38045` | upstream (243 behind) | API-key last resort only at limit |
| `feat/api-key-credit-budgets` `b7482a8` | upstream (243 behind) | prepaid-credit budgets |
| `feat/hard-threshold` | `7187ce8` (= upstream, no own commits) | the work was only uncommitted in the main checkout |
| `wip/uncommitted-main-checkout-20261119` `1e611a9` | `7187ce8` | that uncommitted tree, committed as-is: all four items above combined (what the old 0.27.0b1 install ran) |

## Improvements

| # | Improvement | Origin | What it does | Status in the rebase | Upstream PR candidate? (generic) |
|---|---|---|---|---|---|
| 1 | Hard threshold (`autoswitch.hardThreshold`, `--hard-threshold`) | WIP `1e611a9` (autoswitch/cli/settings + tests) | Opt-in second line on raw utilization. At or above it the engine takes a `hard-limit` trigger: it escapes to the roomiest account (accounts still under their lines first) if that account beats the active one by `hysteresisPct` raw points. The escape ignores the cooldown and the no-return bar. When nothing qualifies it falls back to the proactive rules, cooldown included. Live: 98. | **In rebase branch**, `b054f23` (re-implemented on upstream's per-window lines and per-account thresholds; not used by the priority strategies). The motivating case, an all-over-threshold fleet held by soonest-reset ranking, is now **superseded by upstream `2179636`** (branch `fix/auto-rank-headroom-all-above`, merged in `d16135f`: rank by most room when every account is over its lines). What still differs is escaping inside the cooldown or past the no-return bar. | Yes, generic and opt-in, off by default. Its value is narrower now that upstream ranks all-above by room. |
| 2 | Hard-limit may use the API-key last resort | WIP `1e611a9` (`must_move_trigger`) | A `hard-limit` move may also fall back to API-key accounts, like at-limit. | **In rebase branch**, `b054f23` | Only together with #1 |
| 3 | API-key last resort only at the limit | `7c38045` (+ the same code in WIP) | Metered API-key accounts are used only when the active subscription is at its limit. Never for a proactive move (quota still left) and never for a usage-unknown failover (e.g. the usage endpoint returning 429). | **In rebase branch**, `5b1fd6f` (re-applied on upstream's priority-strategy condition) | Yes: upstream still uses API keys on proactive and failover moves, which spends money while subscription quota is left |
| 4 | Prepaid-credit budgets for API-key accounts (`api_budget.py`, `api-budgets.json`) | `b7482a8` (WIP has the same logic with a real org name in the docstring example; the commit uses `acme`) | Per-organization prepaid budget (`budgetUsd`, `since`, `expiresAt`, `capPct`). Keys over the cap or past expiry are held out of rotation. `cswap list`/`status` show a budget line under API-key rows. | **In rebase branch**, `92c4e86` (`_usage_entry_lines` gained a `budget_line` argument beside upstream's `login_expires_at`) | Maybe: generic opt-in, but it needs an admin/cost key per org, and upstream may not want cost tracking |
| 5 | User-scoped `sk-ant-usr` Console keys recognised as API keys | `1bcdb40` (+ WIP `credentials.py`, `test_api_key_accounts.py`) | `looks_like_api_key` accepts `sk-ant-usr…` keys as well as `sk-ant-api…`; TUI hint text updated. | **Superseded by upstream** `9f72f07` (#465, merged in `71871ba`): any bare `sk-ant-…` key without an OAuth prefix counts as an API key. Dropped our version. | No (already upstream) |
| 6 | WIP test tweak: `test_api_key_not_used_while_subscription_quota_left` param 99 → 94, and the usage-unknown-failover test replaced by a hard-threshold test | WIP `1e611a9` | Test-only. | **Dropped as unnecessary**: the rebase keeps upstream's 99 case and the failover test, both still valid, and adds `test_api_key_used_past_hard_threshold` | No |

## Hunk coverage: WIP `1e611a9` vs `7187ce8`

| File / hunk | Verdict |
|---|---|
| `api_budget.py` (new), `tests/test_api_budget.py` (new) | ported `92c4e86` (sanitized docstring example) |
| `autoswitch.py` SwitchEvent trigger comment; hard-limit classification; `must_move_trigger` + hard→proactive fallback; API-key last-resort condition + `trigger = must_move_trigger`; `hard-limit` candidate filter | ported `5b1fd6f` + `b054f23` |
| `cli.py` example line, `--hard-threshold` flag, banner `hard N%` | ported `b054f23` |
| `cli.py` `--include-api-key-accounts` help text | ported `5b1fd6f` |
| `credentials.py` `API_KEY_PREFIXES` / `looks_like_api_key` (3 hunks) | superseded by upstream `9f72f07` (#465) |
| `settings.py` docstring, `hard_threshold` field, `hardThreshold` spec, `merged_with_cli` | ported `b054f23` |
| `switcher.py` `_usage_entry_lines` budget arg, budget child line, `switchable_account_numbers` budget filter, `api_budgets`/`api_budget_blocked`/`_budget_line`, `list`/`status` callers | ported `92c4e86` |
| `tests/test_api_key_accounts.py` sk-ant-usr test | superseded by upstream (#465 `PERSONAL_KEY` tests) |
| `tests/test_autoswitch.py` API-key at-limit tests | ported `5b1fd6f`; hard-threshold API test ported `b054f23` |
| `tests/test_autoswitch.py` `TestHardThreshold` (6 tests) | ported `b054f23` (7 tests). Two were adapted: past the cooldown the WIP's "holds" shape now switches on upstream alone (`2179636`), so the off/below-hard-line cases assert the cooldown hold instead. |
| `tests/test_config_cli.py` counts, `tests/test_settings.py` round-trip | ported `b054f23` |

**Missing: none.**

## Ops (outside this repo)
- `~/.local/bin/cswap-guard`: `CLONE` points at this worktree (`CSWAP_CLONE` overrides it). It checks these markers: api_budget import, sk-ant-usr detection, `must_move_trigger`, and the `hard_threshold` field. If this worktree is removed, point `CLONE` somewhere else first.
- `claude-acc-*` helpers (incl. `claude-acc-smart-rotate`) come from `~/Desktop/projeler/claude-account-rotation`, not this repo.
