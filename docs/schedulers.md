# Schedulers

Both client frontends use the same account pool, leases and quota accounting.
The Claude Agent SDK additionally supplies typed account-restriction errors to
this lifecycle; see [SDK error mapping](provider-compatibility.md#claude-agent-sdk-account-errors).

## What you experience

- **Selection stays within a provider group.** A Claude model is served by one of your Claude accounts, a `gpt-*` model by one of your Codex accounts. llmux never substitutes a different provider when one is exhausted; crossing providers is your explicit choice (`/model`, or a per-agent `model:` — see [multi-model-agents.md](multi-model-agents.md)).
- **Quota is finite and perishable.** Each account has a 5-hour and a 7-day window that reset on fixed timestamps; quota left unspent at a reset is gone.
- **`default` spends perishable quota first.** It prefers an eligible account whose soon-to-reset quota would otherwise expire unused, but stays on the current account unless another scores clearly higher (>25%), so prompt caches survive. This is a scoring preference, not a promise to always use the soonest-expiring account.
- **`round-robin` never switches proactively.** Fewest switches and best cache locality, at the cost of letting other accounts' quota expire.
- **`priority-drain` empties accounts in a fixed order.** Give accounts a tier; the lowest tier with an eligible account serves everything, and higher tiers are overflow. Use it to keep a bigger account untouched by automation while a smaller one is drained first.
- **You can pin or pause an account manually** (`p` in the TUI switcher; context menu in llmux-islands). Paused accounts are never selected.

## How selection works

Which account serves the next request is decided by the scheduler. Three algorithms ship; cycle live with `S` in the TUI (persisted to `scheduler.mode`), or `POST /llmux/scheduler-mode {"mode": "default" | "round-robin" | "priority-drain"}`.

**Why switching matters:** the upstream prompt cache is scoped per account — every account switch invalidates it, and the next request re-reads the full conversation context uncached (token cost + latency). Both schedulers are therefore sticky on the current account; they differ in *when* they move and *who* is next.

## Eligibility (both modes)

An account can be picked only when ALL of these hold — the same pure gate drives selection, the TUI status column, and `/llmux/status`, so they can never disagree:

- auth healthy, not operator-**paused** (`p` in the TUI switcher, context menu in llmux-islands)
- not cooling down (429 `retry-after` park)
- 5h utilization ≤ `scheduler.five_hour_max` (default **0.90**)
- 7d utilization ≤ `scheduler.seven_day_max` (default **0.99**)
- usage data fresh (≤ `usage_max_age_secs`; if ALL accounts are stale, the gate degrades to headers-only mode instead of stalling)

Per-account overrides: `account_limits` in the config (TUI: `L` in the switcher, `"90,98,98"` = 5h,7d,fbl percents) replace any of the three ceilings for that account.

**Fable scope:** Fable-family requests are additionally refused an account whose Fable weekly bucket is constraining (≥ `scheduler.fable_weekly_max`, default **0.98**, reset-aware) or Fable-cooling. Non-Fable traffic ignores Fable state entirely — a Fable-exhausted account still serves everything else. Full Fable lane mechanics: [fable-scheduling.md](fable-scheduling.md).

## `default` — quota-maximizing

Ranks eligible accounts by `score = servable_now × urgency`: `servable_now` = min(5h, 7d headroom below the ceilings) — the binding limit wins; `urgency` = 1–4× as the 7d reset approaches — soon-to-reset budget is perishable, so it burns first, while long-runway accounts are preserved as reservoirs. Sticky on the current account, but proactively switches when another account scores >25% higher (`SWITCH_MARGIN`) — it trades some cache locality for not letting quota expire unused.

## `round-robin` — sequential exhaust (fewest switches)

Stays on the current account until it is **hard ineligible** (ceiling hit, cooldown, auth, pause) — never a proactive switch — then moves to the **next account in roster order**, wrapping. Deterministic, score-free, and the minimum possible number of switches, at the cost of letting other accounts' soon-to-reset quota expire unused. Pick this when prompt-cache locality (long agent sessions) matters more than squeezing every window.

## `priority-drain` — tiered overflow

`scheduler.mode = "priority-drain"` adds a drain order on top of `default`. Each account may carry `account_limits.<name>.priority` (integer, **lower drains first**; unset = after every configured tier). Existing configs are unchanged: the key is ignored by the other modes, and with no priorities set every account shares one tier, so decisions equal `default`.

- **Within a tier** the `default` perishability score and 25% stickiness margin apply unchanged.
- **Across tiers** a higher tier is a candidate only when every account of every lower tier is ineligible: weekly ceiling hit, paused, auth-unhealthy, stale usage, or in a 429 cooldown. Per-account `seven_day_max` still applies on top, so a ceiling on the overflow account is a hard floor.
- **5h-only gating** (`scheduler.spill_on_five_hour`, default `true`): a lower-tier account that is blocked only by its 5h ceiling (weekly capacity left) lets the next tier serve until the window clears, so the pool never 429s while any tier is eligible. Set `false` for strict drain: the higher tier is then untouched until the lower tier is out for the week, and requests get a `429` whose `retry-after` is the held account's own 5h reset (up to ~5h; clients stall for that long).
- **Return is immediate.** When a lower tier becomes eligible again, new requests move back with no switch margin: the sticky lease on the overflow account is refused as soon as a better tier is serving, which triggers an evaluation on the spot (not at the next 60s tick). Requests already in flight on the overflow account finish there.
- **All tiers out:** the usual `429` with the soonest `retry-after`.
- **A manual pin** (`s` in the TUI) outranks tiers, like every other automatic policy.
- **Visibility:** `limits.priority` is in `/llmux/status` and the dashboard document; an account that passes its gates but is not being served shows `blocked: "tier 2: lower tier tech serving"` (or `... 5h-held` under strict drain).

Example: drain a Max 5x account before a Max 20x one, never letting automation take the 20x account past 95% weekly:

```json
{
  "scheduler": { "mode": "priority-drain", "spill_on_five_hour": true, "overlap_promote": true },
  "account_limits": {
    "claude:tech@bethelchurchak.org": { "priority": 1, "seven_day_max": 0.999 },
    "claude:andrew@alaskaautorental.com": { "priority": 2, "seven_day_max": 0.95 }
  }
}
```

### Overlap promotion (`scheduler.overlap_promote`, `priority-drain` only)

Accounts' weekly windows roll at different times (tech@ Sat 15:59, andrew@ Sat 22:00). Between a peer's rollover and an account's own, that account holds an *expiring* remainder while the peer has a fresh week, so burning the stale one first loses nothing. With `overlap_promote: true` this is derived from the pool's own data, with no hour constants:

- `window_start(X) = X.seven_day.resets_at - 7d`. Account A is in **overlap** when a same-backend-group peer has a newer `window_start` (rolled by more than 60s, start not in the future) while A's window is still open. The **overlap start** is the newest such peer start. The freshest-rolled account has no newer peer and is never in overlap. A missing or already-expired `resets_at` on A or the peer means no overlap (normal order). Codex accounts are never promoted (they stay the last-resort overflow pool).
- An eligible account in overlap is **promoted ahead of every tier**; among several, the earliest `resets_at` first. The usual gates still apply: `seven_day_max` (incl. the ramp), `five_hour_max`, paused, stale, cooldown. A promoted account that is gated falls through to the normal tier order (it never starts a strict 5h hold); a promotion is picked up on the next evaluation with no 25% switch margin, and traffic returns to the normal order the same way when the overlap ends.
- **Ceiling ramp.** Across `[overlap start, own resets_at]` the account's effective weekly ceiling (its `account_limits.<name>.seven_day_max`, else `scheduler.seven_day_max`) rises linearly to `seven_day_max_end` (`scheduler.seven_day_max_end`, default `0.99`; per-account `account_limits.<name>.seven_day_max_end` overrides; capped at 1.0). Outside overlap the base ceiling applies; the gate, score and status text all use the ramped value. An end not above the base is inert, so tech@'s `0.999` never ramps.
- **Consequence for tiers.** With the switch on, accounts drain **soonest-reset-first** (every account except the freshest is promoted, earliest reset first); `priority` is the tiebreak and the overflow order once the promoted accounts are out. With two accounts the older-window one (tech@, most of the week) is promoted too, which matches tier order; andrew@ is promoted only in the Sat 15:59-22:00 window. Off (default) = exactly the behaviour above. The switch only acts in `priority-drain`.

A ceiling of `1.0` is allowed (`utilization <= 1.0` stays eligible); a real upstream 429 at 100% parks the account through the normal cooldown path.

Hand edits to the config file take effect on `llmux restart`; the `S` toggle and TUI/endpoint limit edits apply live. Editing ceilings from the TUI (`L`) keeps an existing `priority`; the tier itself is set in the config file.

## Adding a scheduler

The selection logic is pure and lives in `src/scheduler/select.rs` (`pick_scoped`, `rank`, `round_robin_next`) — deterministic functions of `(PoolSnapshot, SelectParams, now)`, unit-tested without IO. To add a mode: extend `SchedulerMode` in `src/config/schema.rs`, branch in `pick_scoped`/`next_in_line`/`selection_order`, and document it here.

The full derivation, edge cases, and the wasted-quota simulation behind the `default` policy live in [`.prd/09-scheduler-perishability.md`](../.prd/09-scheduler-perishability.md).
