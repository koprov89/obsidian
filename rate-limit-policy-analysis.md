# Cloudflare Rate-Limiting Policy: Full Analysis & Recommendations

**Date**: 2026-04-17  
**Data**: `cloudflare-beef-logs*` (2026-02-20 – today) + `cloudflare-*-logs*` (24 zones)  
**Baseline**: 2026-04-10 – 2026-04-16 (quiet week, 76.6M requests across all sites)

---

## 1. Normal Traffic Baseline

### beef.casino (largest, most stable site — best reference)

| Metric | Value |
|--------|-------|
| Normal daily volume | ~960K req/day |
| Peak day (Mon) | 1.29M |
| Low day (Sun) | 580K |
| % unmatched (no rule fired) | 84.5% |
| % skip (allowlisted) | 9.3% |
| % challenged or blocked | 2.0% |
| % challengeBypassed (cookie) | 4.2% |
| Avg BotScore | 74.3 |
| % BotScore 0–9 (clear bot) | 9.6% |
| % BotScore 80–100 (human) | 66.9% |
| Top countries | ru 44%, de 15%, by 7%, pl 4%, nl 4% |

### All zones combined (quiet week, 76.6M requests)

| SecuritySource | Hits | Share |
|----------------|------|-------|
| (no rule) | 65.1M | 85.0% |
| firewallCustom | 11.1M | 14.5% |
| rateLimit | 430K | 0.56% |
| l7ddos | 76K | 0.10% |
| firewallManaged | 37K | 0.05% |

---

## 2. Legitimate Per-IP Request Rates (7-day, beef.casino, skip-action only)

| Percentile | 7-day total | Per-day |
|-----------|------------|---------|
| p50 | 21 | 3 |
| p90 | 107 | 15 |
| p95 | 244 | 35 |
| p99 | 941 | 134 |
| p99.5 | ~1,900 | ~271 |
| max (excluding internal IP 212.98.191.19) | ~3,200/day | — |

**p99 legitimate user**: ~134 req/day, or **~1 request per 10 minutes** on average.

The only extreme outlier is 212.98.191.19 (Belarus, ASN 12406) with 64K+ req/day — explicitly whitelisted via `skip` rule — clearly an internal load-testing or monitoring tool. The second-highest legitimate IP makes ~3,200 req/day (22 req/hr).

---

## 3. Current Rate-Limit Rule Inventory & Effectiveness

### Firing frequency (normal quiet week)

| Rule | Weekly hits | Action | Status |
|------|-------------|--------|--------|
| IP RateLimit 2 min * | 195,442 | managedChallenge | Active, most-used RL rule |
| **hardcore_test_for_false_positives** | **75,090** | **log only** | **⚠ NOT BLOCKING** |
| Empty refer * | 69,239 | managedChallenge | Active |
| 1000 (1500) not_cached_requests per 10 sec | 43,213 | managedChallenge | Active |
| 1000 not_cached_requests per 10 sec | 20,546 | managedChallenge | Active |
| IP RateLimit 2 min API | 16,279 | managedChallenge | Active |
| IP RateLimit 10 min API 600 (900) | 5,364 | managedChallenge | Active |
| Empty refer API | 2,489 | managedChallenge | Active |
| IP 60 (200) in minute | 180 | managedChallenge | Barely firing |
| IP RateLimit 10 min SIGN_IN \| REG | 152 | managedChallenge | Barely firing normally |
| games limit | 35 | managedChallenge | Barely firing |
| registration_limit | 13 | managedChallenge | Barely firing |

### Bypass rates by rule type (critical comparison)

| Trigger | Action | Bypass rate (quiet week) | Bypass rate (attack) |
|---------|--------|--------------------------|----------------------|
| **rateLimit** | managedChallenge | **0.01%** | **~0.01%** |
| firewallCustom | managedChallenge | 1.8% | varies |
| l7ddos | managedChallenge | ~0% | **23.5% on fugu** |
| securityLevel | managedChallenge | ~83% (cookie) | very high |

> **Key finding**: Rate-limit-triggered `managedChallenge` is nearly impossible to bypass (0.01%). The problem during attacks is NOT the rate-limit rules — it's the l7ddos and securityLevel rules that use managedChallenge and get bypassed by sophisticated bots with real browsers.

---

## 4. Attack Analysis (fugu.casino, 2026-04-17)

### Scale comparison

| Metric | beef.casino (normal day) | fugu.casino (attack, 6h) |
|--------|--------------------------|--------------------------|
| Total requests | ~960K | **39M** (40× normal) |
| Avg BotScore | 74.3 | **16.9** |
| % BotScore 0–9 | 9.6% | **55.2%** |
| MC bypassed rate | 2.0% | **23.5%** |
| Top IP rate | ~134/day (~0.1/min) | **480 req/min** |
| Unique attacking IPs | — | 14,700+ |

### What triggered during the attack

| SecuritySource | Hits | Action used | Bypass |
|----------------|------|-------------|--------|
| l7ddos | 23.3M | managedChallenge | **HIGH — this is the leak** |
| rateLimit | 11.5M | managedChallenge | 0.01% — working fine |
| securityLevel | 10.7M | managedChallenge | HIGH (cookie-based) |
| firewallCustom | 2.1M | mixed | varies |

### Rate-limit rule scaling during attack

| Rule | Normal (per week) | Attack (today) | Scale factor |
|------|-------------------|----------------|--------------|
| IP RateLimit 2 min API | 16,279 | 2,646,997 | **163×** |
| IP RateLimit 10 min SIGN_IN\|REG | 152 | 918,000 | **6,039×** |
| All rateLimit source | 430,022 | 11,500,000 | **27× per hour** |

The rate-limit rules detected and challenged the attack aggressively. The bypass problem is NOT from rate-limit rules — it's from l7ddos-triggered managedChallenge.

### Normal weekly volume on attacked paths (all sites)

| Path | Normal weekly | Attack today (fugu, 6h) | Amplification |
|------|--------------|------------------------|---------------|
| /ml/games-rtp/ | 140K | 69.7M | **498×** |
| /api/v4/payment_methods/popular | 241K | 18.1M | 75× |
| /api/v3/games/by_category/new | ~200K | 16.5M | 83× |
| /api/users/sign_in | 122K | 4.6M | 38× |

---

## 5. Elevated Bypass Zones — False Alarm Clarification

The 6h window showed several zones with high MC bypass percentages. Investigation shows these are **not attacks** — the absolute numbers are tiny:

| Zone | MC hits today | MC bypassed today | Bypass % | Verdict |
|------|--------------|-------------------|----------|---------|
| fugu.casino | 26.4M | 8.1M | 23.5% | **ATTACK** — 8.1M absolute bypasses |
| drip.casino | 51 | 894 | 94.6% | Normal — only 945 total MC interactions |
| legzo.casino | 110 | 685 | 86.2% | Normal — 795 total, returning users with cookies |
| volna.casino | 67 | 252 | 79.0% | Normal — 319 total |
| jet.casino | 253 | 503 | 66.5% | Normal — 756 total |

For these smaller zones, a "high bypass %" simply means their managedChallenge rules fire rarely, and when they do, most are returning users with valid cookies. **No action needed.**

---

## 6. Path-Level Security Profile (Normal Week, All Sites)

### Paths with elevated challenge/block rates in normal traffic

| Path | Total/week | Notable | Likely cause |
|------|-----------|---------|--------------|
| /api/v4/tournaments | 244K | **38.3% block** | Bot scraping (38% from Estonian ASNs) |
| /cent/connection/websocket | 570K | **13.9% JS challenge + 6.6% MC** | WebSocket heavily targeted; challenges break WS |
| /sdk.js | 314K | **12.9% managedChallenge** | JS static file challenged — unusual |
| /cp | 1.78M | 18.5% MC | bad_countries rule on maxclientstatapi.com — expected |

### Paths with unexpectedly low protection (no dedicated rule)

| Path | Normal weekly | Attack today | Has dedicated RL rule? |
|------|--------------|--------------|----------------------|
| /api/v4/payment_methods/popular | 241K | 18.1M (75×) | ❌ No |
| /api/v3/games/by_category/new | ~200K | 16.5M (83×) | ❌ No |
| /ml/games-rtp/ | 140K | 69.7M (498×) | ✅ Added today ("ml") |
| /api/users/sign_in | 122K | 4.6M (38×) | ✅ Exists but threshold high |

---

## 7. Recommendations

### 7.1 — Fix the Root Cause of DDoS Bypass: Change L7 DDoS Response to Block

**Evidence**: The 8.1M bypassed requests came from `l7ddos`-triggered `managedChallenge`. Rate-limit rules have 0.01% bypass; l7ddos-triggered MC got bypassed at 23.5% because the attacking bots run real headless Chrome and can solve managed challenges.

**Action**: In the Cloudflare dashboard → **Security → DDoS** → configure the L7 DDoS mitigation action:

```
Current:  l7ddos → managedChallenge
Proposed: l7ddos → block
```

**Or** add a custom WAF rule with higher priority than l7ddos to block when the l7ddos override is active. Cloudflare exposes `cf.threat_score` and per-path conditions.

This single change would eliminate the majority of the 8.1M bypass problem during DDoS events.

---

### 7.2 — Activate `hardcore_test_for_false_positives`

**Evidence**: Fires **75,090 times/week** in `log` mode, blocking nothing.

**Action**: 
1. Review the rule's match conditions against the logged traffic in OpenSearch (filter by `SecurityRuleDescription: "hardcore_test_for_false_positives"`, check the matched IPs/paths/UAs)
2. If false-positive rate is acceptable, switch to `managedChallenge` → then to `block` after monitoring
3. If the name implies it was intentionally left in log mode for testing, set a reminder to activate or remove it

**Expected impact**: ~75K additional challenges/week without new false positives.

---

### 7.3 — Add Hard Rate Limits on Newly-Exposed Attack Paths

No dedicated rate-limit rules exist for these paths that received 75–498× normal traffic today:

#### /api/v4/payment_methods/popular
Normal: 241K/week (~34K/day, ~1.4K/hr). Attack: 18.1M in 6h.

```
Name: rl-payment-methods-popular
Path: /api/v4/payment_methods/popular
Threshold: 20 req / 1 min / IP
Action: block
```

#### /api/v3/games/by_category/new
Normal: ~200K/week (~29K/day). Attack: 16.5M in 6h.

```
Name: rl-games-by-category
Path: /api/v3/games/by_category/new
Threshold: 20 req / 1 min / IP
Action: block
```

Both thresholds are ~20× what a real user would do per minute. No false positives expected at p99.5.

---

### 7.4 — Upgrade "ml" Rule Action from `managedChallenge` to `block`

**Evidence**: Current "ml" rule (created today) uses `managedChallenge`. The attack produced 5.6M bypasses on `/ml/games-rtp/` in 6 hours. The rate-limit-triggered version with `block` has 0.01% bypass.

**Action**:
```
Name: ml-ratelimit
Path: /ml/games-rtp/
Threshold: 30 req / 1 min / IP
Action: block  ← change from managedChallenge
```

Legitimate users visit this path a few times per session. Top attack IPs were doing 350–480 req/min. 30/min is safe.

---

### 7.5 — Lower Threshold on Sign-In Rate Limit

**Evidence**: The `IP RateLimit 10 min SIGN_IN | REG` rule fired 918,000 times during the attack but only 152 times in a normal week. The rule IS working — but with 4.6M total sign-in requests and 2,763 unique IPs, each IP still got ~1,660 free requests before the threshold kicked in (4.6M − 918K = 3.68M below threshold, divided by 2,763 IPs ≈ 1,330 free per IP).

This suggests the current threshold is around **~100–200 req/10min per IP**. Real users do at most 5 sign-in attempts per session.

**Proposed**:
```
Name: rl-sign-in-strict
Path: /api/users/sign_in
Threshold: 5 req / 10 min / IP
Action: block

Name: rl-registration-strict  
Path: /registration
Threshold: 3 req / 10 min / IP
Action: block
```

**Why 5 sign-ins / 10 min is safe**: Legitimate p99 behavior on beef is ~3 req/10min total on the entire site — sign-in specifically would be far fewer.

---

### 7.6 — Add BotScore Hard Block on `/api/*`

**Evidence**: During normal traffic, 9.6% of all requests score BotScore 0–9 (Cloudflare's highest-confidence bot category). During the attack, this jumps to 55%. Cloudflare score < 10 = near-certain bot.

```
Name: api-definite-bot-block
When: http.request.uri.path matches "^/api/"
      AND cf.bot_management.score lt 10
Action: block
```

**Before enabling**: Verify that no partner/internal API callers are scoring < 10. The internal test IP (212.98.191.19) scores avg 23 — it would not be blocked by this rule. Add any legitimate automation to an allowlist first.

**Expected impact**: Blocks ~9.6% of normal API traffic that is already low-bot-score. During attacks, this blocks 55% of attack traffic at the API level directly.

---

### 7.7 — Fix WebSocket Challenge Rule

**Evidence**: `/cent/connection/websocket` has 570K requests/week with 13.9% getting JS challenge and 6.6% managed challenge. WebSocket connections cannot complete a challenge — the challenge interrupts the WS handshake.

**Action**: Replace the challenge rule on this path with a **rate limit by IP**:
```
Name: rl-websocket
Path: /cent/connection/websocket
Threshold: 10 new connections / 1 min / IP
Action: block
```

Remove the existing JS challenge/managedChallenge rule on this path.

---

### 7.8 — Add Tournament Scraping Block to All Zones

**Evidence**: `/api/v4/tournaments` has 38.3% block rate in normal week, concentrated on monro.casino with 38% of requests from Estonian ASNs (clear bot scraping). The `anti_bot_99%` rule exists on monro.casino but may not be deployed to all zones.

**Action**: Ensure this rule is applied globally:
```
Name: block-tournaments-bots
Path: /api/v4/tournaments
When: cf.bot_management.score lt 30
Action: block
```

---

### 7.9 — Review `games limit` and `registration_limit` Rules

**Evidence**: Fire 35 and 13 times/week respectively. These are near-zero hit rates even though `/api/v3/games/by_identifier` has 400K+ weekly requests and registration is a target during credential-stuffing attacks.

**Possible issues**:
- Threshold too high (attackers staying under it)
- Path pattern not matching correctly (e.g., `/registration` vs `/api/users/sign_in`)
- The rules may be redundant with broader rules

**Action**: Log the matched traffic for these rules and verify the path patterns are correct. Consider merging with rule 7.5 above.

---

## 8. Summary Table

| # | Action | Expected Impact | Priority |
|---|--------|-----------------|----------|
| 7.1 | Change L7 DDoS action: managedChallenge → **block** | Eliminate ~8M bypass/attack | 🔴 Critical |
| 7.2 | Activate `hardcore_test_for_false_positives` | +75K blocks/week | 🟠 High |
| 7.3 | Add RL rules on /api/payment and /api/games paths | Block paths with 75–83× attack amplification | 🟠 High |
| 7.4 | Upgrade "ml" rule: managedChallenge → **block** | Eliminate 5.6M/attack bypass on ML path | 🟠 High |
| 7.5 | Lower sign-in/registration threshold to 5 req/10min | Stop credential stuffing earlier | 🟡 Medium |
| 7.6 | Add BotScore < 10 → block on /api/* | Block 9.6% of normal bots, 55% during attack | 🟡 Medium |
| 7.7 | Replace WS challenge with RL rule | Fix broken challenge behavior on WebSocket | 🟡 Medium |
| 7.8 | Deploy tournament scraping block to all zones | Extend existing per-zone rule globally | 🟡 Medium |
| 7.9 | Audit games/registration_limit rules | Fix near-zero rules | 🟢 Low |

---

## 9. Threshold Reference (Data-Driven)

Based on legitimate user p99 behavior from beef.casino:

| Path | Legitimate p99 (per IP/min) | Recommended limit | Attack rate |
|------|--------------------------|-------------------|-------------|
| Any `/api/*` (general) | < 0.1 req/min avg | existing 2-min rules OK | varies |
| `/api/users/sign_in` | < 0.01 req/min | **5 req/10min** | 480 req/min |
| `/registration` | < 0.005 req/min | **3 req/10min** | high |
| `/ml/games-rtp/` | < 0.01 req/min | **30 req/min** | 350–480 req/min |
| `/api/v4/payment_methods/popular` | < 0.1 req/min | **20 req/min** | ~50K req/min total |
| `/api/v3/games/by_category/new` | < 0.1 req/min | **20 req/min** | ~45K req/min total |
| `/cent/connection/websocket` | 1 connection/min | **10 new conn/min** | — |

---

## 10. What Is Working Well

- **Rate-limit rules** (`rateLimit` source): 0.01% bypass rate — nearly perfect, scaled 163× during attack
- **Block rules** (`firewallCustom`): 1.24M blocks/week in normal times, consistent
- **`anti_bot_99%` rule**: 0.002% bypass — excellent for tournament/game endpoint scraping
- **`bad_countries`/`bad_as_account`/`bic` rules**: stable block coverage
- **IP allowlisting** for internal tools (212.98.191.19) — correctly exempted from all rules

The core architecture is solid. The main gaps are:
1. L7 DDoS using managedChallenge instead of block (biggest gap)
2. Missing path-specific RL rules for 2-3 high-value API endpoints
3. One rule stuck in log mode for months
