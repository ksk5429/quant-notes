---
title: Korean legal status
description: Summary of the Korean legal analysis for the hypekr-bot component
---

# Korean legal status of the Telegram bot

!!! danger "Not legal advice"
    This page is a **research summary**, not legal advice. The detailed
    due-diligence brief (with full statute citations and attorney
    questions) lives in the private `kfish` repo at
    `runbooks/kr-legal-brief.md`. If you're building something similar,
    you need your own Korean counsel — don't rely on this.

## TL;DR posture

The Korean Telegram bot (`hypekr-bot`) is architecturally non-custodial
and fee-collecting, routing orders to the offshore Hyperliquid DEX. This
sits inside **three genuinely disputed legal zones** under Korean law:

1. **VASP status under 특금법** (Act on Reporting and Using Specified
   Financial Transaction Information). Non-custodial wallets *alone*
   don't trigger registration, but **fee-based intermediation can**.
   KoFIU's 2025-12-02 press release explicitly named "stablecoin
   exchanges conducted through Telegram or open chat rooms" as illegal.
2. **Gambling under Criminal Code Article 246 / 247.** Prediction-
   market-style contracts (HIP-4 binary outcomes) are a gray zone.
   Supreme Court doctrine defines gambling as property wagered on a
   chance-based outcome. Offshore venue doesn't obviously exempt Korean
   users.
3. **Derivatives / financial promotion under Capital Markets Act.**
   Marketing binary event contracts to KR retail could be construed as
   unlicensed foreign-derivative promotion.

**Current stance:** paper trading only, no KR marketing, no KRW on/off-ramp,
no sports/election markets, 19+ gate, explicit disclaimer on `/start`.
Every item in the blueprint's §3.9 "failure modes" remains current.

## Laws in scope (with citations)

These are the statutes a Korean attorney will actually care about.

| Law | Korean | Effect on the bot |
|---|---|---|
| **Virtual Asset User Protection Act** (effective 2024-07-19) | 가상자산이용자보호법 | Not primarily binding — bot doesn't hold user VA or KRW. But VASP-definition overlap is possible. |
| **Specified Financial Transaction Information Act** (amended 2021) | 특정금융거래정보의 보고 및 이용 등에 관한 법률 | **Central concern.** VASPs targeting Korean users must register with KoFIU. 2025-12-02 enforcement statement targets Telegram-based intermediaries. |
| **Travel Rule** (in force since 2022-03) | — | ≥ ₩1M per transfer threshold. Being expanded downward in 2026. |
| **Criminal Code Article 246** | 형법 제246조 | Individual gambling: fine up to ₩10M; habitual up to ₩20M + 3 years. |
| **Criminal Code Article 247** | 형법 제247조 | Operating gambling for profit: fine up to ₩30M + 5 years. |
| **Capital Markets Act** | 자본시장과 금융투자업에 관한 법률 | Expansive "derivatives" definition — HIP-4 event contracts could fall inside. |
| **Personal Information Protection Act** (updated 2024) | 개인정보보호법 | Cross-border data transfer requires granular, non-bundled consent. Applies to our Mini App sign flow. |
| **Electronic Financial Transactions Act** | 전자금융거래법 | Triggered only if the bot ever handles KRW. Currently out of scope by design. |

## The three questions that need a real attorney

These are the exact questions on the attorney-meeting agenda:

1. **Does a non-custodial, fee-collecting Telegram bot routing to an
   offshore DEX qualify as a VASP under 특금법 Art. 2?** If yes, what's
   the narrowest registration path? If no, do we still need KoFIU
   pre-notification because we're "directed to the Korean market"?
2. **Are HIP-4 binary event contracts "gambling" under Criminal Code
   Art. 246 when accessed from Korea?** What category distinctions
   matter — sports markets vs non-sports?
3. **If the above resolve favorably, what minimum ToS / privacy policy
   / disclaimer language does PIPA and 전자상거래법 require for
   targeting KR retail?**

## Current enforcement posture (what's actually happening)

The single most important document is KoFIU's **2025-12-02 press
release**, which names three categories of illegal virtual-asset
activity:

1. **Stablecoin exchanges conducted through Telegram or open chat rooms**
2. Blog / social-media promotion or referral for unregistered VASPs
3. Illegal VA exchange via currency-exchange businesses

hypekr-bot is architecturally adjacent to category 1, which is why the
paper-only posture matters.

Other active moves:

- **2026-01-28:** Google Play Korea requires all crypto apps to hold
  VASP registration — Binance, Bybit, OKX effectively blocked from the
  KR store.
- **2024 precedent:** blocks on 16 offshore VASPs that didn't register
  with KoFIU.

## Recommended firms

Chambers-ranked Band 1 FinTech Legal (South Korea), 2025/2026:

- [**Kim & Chang** (김&장)](https://www.kimchang.com/en/) — largest firm, strongest regulatory bench
- [**Lee & Ko** (이앤코)](https://www.leeko.com/) — Band 1 FinTech 2026, individually-ranked crypto partners
- [**Yulchon LLC** (율촌)](https://www.yulchon.com/) — fintech-forward

Band 2:

- [**Bae, Kim & Lee LLC**](https://www.bkl.co.kr/) — strong blockchain team under Jongbaek Park
- [**Shin & Kim**](https://www.shinkim.com/) — first-VAUPA-case commentary

Specialist boutiques worth a call: **D'Light Law Group**, **SEUM Law**,
**Decent Law**.

## Budget (informational)

Based on blueprint §3.9 and current rate cards, a full written opinion
on Q1 (VASP status) + Q2 (gambling) runs ~₩5–13M. Korean ToS + privacy-
policy drafting adds another ~₩1.5–3M. Ongoing compliance retainers
start ~₩500K/month at Band-2 firms.

Timeline: 2-4 weeks from retainer to written opinion on Q1/Q2. ToS
drafts typically 2-3 weeks after that.

## What this page deliberately does NOT say

- Whether the bot **is** legal in Korea. That takes a written opinion
  from licensed counsel.
- Specific PIPA-compliant privacy-policy text.
- A guarantee that the KoFIU position won't move — re-run the research
  every quarter through Q2 2026.

## Related

- [Open Questions](../reviews/open-questions.md) — the collaborator
  discussion version of these items
- [Architecture: Telegram bot](../architecture/hypekr-bot.md) — the
  technical design the legal analysis is reasoning about
- [Build log: live-hardening](../reviews/2026-04-19-live-hardening.md)
  — current status of the bot's paper-trading readiness
