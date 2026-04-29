# yourOS — Redesign Brief

A system reframe of the current MVP into a behavioral operating system.

The current MVP is a productivity app: tasks, escalating notifications, a discipline score, a dashboard. That product is opened, used, closed. yourOS is the inverse — it runs against the user from wake to sleep, treats every commitment as a contract, and treats execution failure as the only signal that matters.

This document defines what to build, in what order, and what to throw away.

---

## 1. System Overview

**What yourOS is:**
A continuously-running behavioral runtime that converts a user's stated identity into time-bound commitments, observes whether they execute on those commitments using passive sensors, and applies graduated consequences when they drift. It does not coach. It does not motivate. It enforces the contract the user signed with themselves.

**Inversion from the MVP:**

| MVP (productivity app) | yourOS (behavioral OS) |
|---|---|
| Task | Commitment (time-bound contract with a stake) |
| Notification | Escalation chain across channels |
| Discipline Score | Identity Score per declared vector |
| Dashboard | Drift Report (gap between stated and observed self) |
| Session-based (open the app) | Always-on (the app runs against you) |
| Tracks productivity | Reshapes identity through forced execution |
| Reminds | Enforces |
| Asks | Escalates |

**The unit of work is the Commitment**, not the task. A commitment has: an identity vector it belongs to, a time window, a verification method, and a consequence on miss. No stake, no commitment. No verification, no commitment. No identity vector, no commitment. This is the only object the user creates.

**The single failure mode the system exists to prevent:** drift between who the user has declared they are becoming and who their behavior shows they actually are. Everything else is in service of closing that gap.

---

## 2. Architecture

Six modules. Each owns one responsibility. They communicate over an internal event bus so the system can run while the app is closed.

```
                       ┌────────────────────────────┐
                       │     IDENTITY ENGINE        │
                       │  declared vectors,         │
                       │  identity scores,          │
                       │  drift detection           │
                       └────────────┬───────────────┘
                                    │ policy
                                    ▼
                       ┌────────────────────────────┐
                       │    COMMITMENT LAYER        │
   user input ───────▶ │  contracts, stakes,        │
                       │  windows, verification     │
                       └─────┬───────────────┬──────┘
                             │ active        │ outcomes
                             │ window        │
                             ▼               │
   ┌─────────────────┐   ┌────────────────┐  │
   │  SENSOR MESH    │──▶│  INFERENCE     │  │
   │  phone, band,   │   │  CORE          │  │
   │  calendar, GPS, │   │  rule + adapt. │  │
   │  Screen Time    │   │  classifier    │  │
   └─────────────────┘   └────────┬───────┘  │
                                  │ verdict  │
                                  ▼          │
                       ┌────────────────────────────┐
                       │   ENFORCEMENT BUS          │
                       │  escalation policies,      │
                       │  channel routing:          │
                       │  haptic / lockout /        │
                       │  social / financial        │
                       └────────────┬───────────────┘
                                    │ event
                                    ▼
                       ┌────────────────────────────┐
                       │        LEDGER              │
                       │  immutable record,         │
                       │  feeds Identity Engine,    │
                       │  source of truth           │
                       └────────────┬───────────────┘
                                    │ updates
                                    └──▶ back to Identity Engine
```

**Module responsibilities:**

- **Identity Engine** — owns declared identity vectors (e.g., "trains 5 days a week", "ships code daily", "in bed by 23:00"). Maintains an Identity Score per vector. Sets policy strictness based on current drift. Generates the Drift Report.
- **Commitment Layer** — converts identity vectors into concrete time-bound commitments. Holds the contract: window, stake, verification method, escalation policy. No commitment exists without all four.
- **Sensor Mesh** — passive data collection. Phone foreground apps (Screen Time API), location, motion, calendar events, wearable biometrics (HRV, motion, skin conductance). On-device. Continuous.
- **Inference Core** — answers one question per active commitment window: *is the user complying right now?* Rule-based for clear cases, adaptive classifier for ambiguous ones (the "stuck in traffic vs watching YouTube" call).
- **Enforcement Bus** — receives verdicts, applies the escalation policy on the contract, routes interventions to the right channel. Stateful — knows where in the escalation chain a given commitment is.
- **Ledger** — append-only record of every commitment, every intervention, every outcome. Cannot be edited or deleted. Feeds Identity Engine and is the only source of truth for Identity Scores.

---

## 3. Core Loops

### Primary loop: Commitment Execution (runs per active window)

```
1. TRIGGER         Commitment window opens (clock or context, e.g. "arrive home → 30min training")
2. SENSE           Sensor Mesh polls relevant signals (location, foreground app, motion)
3. INFER           Inference Core emits verdict: COMPLIANT | DRIFTING | NON_COMPLIANT
4. ENFORCE         Enforcement Bus applies next step in escalation policy
                   - DRIFTING → ambient haptic, glance UI
                   - NON_COMPLIANT → app lockout, escalating haptic, partner ping
                   - REPEATED NON_COMPLIANT → stake forfeiture, hardware aversive (opt-in tier)
5. RESOLVE         Window closes with outcome: HIT | MISS | PARTIAL
6. LEDGER          Outcome written, immutable
7. RECALC          Identity Engine updates the relevant vector's score and policy strictness
8. POLICY DRIFT    If score on a vector falls below threshold, future commitments in that vector
                   default to stricter stakes and tighter verification
```

This loop runs continuously. There is no "open the app and start a session." The app is the runtime; the user lives inside it.

### Secondary loops

**Adaptation loop (weekly cadence)**
Inference Core re-fits the user's personal drift fingerprint: when do they typically fail (Sunday evenings, post-meal, after social events, on travel days). System raises preemptive friction during predicted high-drift windows next week. No user input required.

**Punishment loop (per miss)**
On MISS: stake forfeits (held via payment provider on commitment creation), Ledger logs it, accountability partner receives auto-generated miss report, Identity Score drops, next-N commitments in same vector escalate stake or shrink verification window. Punishment is automatic and unappealable.

**Reward loop (per streak)**
The only positive reinforcement: streaks reduce friction. After 14 consecutive HITs on a vector, stake requirement drops; verification window widens; ambient nudges replace hard lockouts. The carrot is *less of the stick*. There are no badges.

**Identity drift loop (continuous, surfaced weekly)**
Identity Engine compares declared vectors against observed behavior in the Ledger. If a user declared "trains 5 days a week" and the rolling four-week observation shows three, the Drift Report surfaces the gap and forces a choice: tighten enforcement on that vector, or amend the declared identity. There is no third option. The system refuses to let the lie persist.

---

## 4. Control Layers

Five layers, ordered by intrusiveness. Every commitment runs through them in sequence as drift accumulates.

| Layer | Name | Mechanism | When it fires |
|---|---|---|---|
| **L0** | Passive monitoring | Sensor Mesh always on | Continuously |
| **L1** | Ambient nudge | Live Activity / lock screen / single haptic | Window opens, user is compliant or pre-compliant |
| **L2** | Active intervention | Escalating haptic, full-screen takeover, app lockout via Screen Time API | DRIFTING verdict |
| **L3** | Hard enforcement | Stake forfeiture, accountability partner auto-ping, cross-device lockout, identity-vector flagged | NON_COMPLIANT verdict |
| **L4** | Aversive (opt-in) | yourBand electrical stimulus | REPEATED non-compliance on a contract the user explicitly signed at this tier |

L0–L3 are software-only and shippable on iOS today. L4 is hardware-gated and ships when the band is in market. Every layer is mandatory for users at that tier — there is no per-event opt-out, only per-contract opt-in at signup.

---

## 5. Identity System

**Declaration.** On onboarding, the user picks 3–5 identity vectors from a constrained list (extensible later). Examples: *Trains daily. Sleeps before 23:00. Ships code Monday–Friday. Calls family weekly. Reads 30min/day.* Custom vectors require the user to define a verification method — without one, the vector cannot be declared.

**Mapping.** Every commitment is tagged to one or more vectors. A 06:00 gym block maps to *Trains daily*. A 22:30 phone-down block maps to *Sleeps before 23:00*. The user does not create commitments in a vacuum — they choose a vector, and the system suggests commitments that satisfy it.

**Identity Score per vector** (0–100). Computed from the rolling four-week Ledger for that vector's commitments. Weighted by recency. Cannot be inflated by completing easy commitments — Inference Core flags trivial commitments and excludes them from the score.

**Real-time updates.** Every commitment outcome immediately moves the relevant vector's score. The user sees this in the Drift Report.

**Punishment for misalignment.** If a vector's Identity Score falls below 60, the system enters *enforcement mode* on that vector: stricter stakes, tighter verification, narrower windows, more escalation steps. Below 40: the system requires the user to either amend the declared identity (admit who they actually are) or sign a recovery contract with significantly higher stakes. The system will not let a user maintain an identity their behavior contradicts.

---

## 6. AI Role (Realistic)

Most decisions are **rule-based**. The adaptive layer is narrow and earns its place.

**Rule-based (deterministic, transparent, auditable):**
- Escalation timing and channel selection
- Lockout policies during commitment windows
- Identity Score arithmetic
- Stake-forfeiture trigger conditions
- Streak detection and reward thresholds
- Drift classification thresholds

**Adaptive (model-based, on-device):**
- *Compliance classification under ambiguity* — distinguishing "stuck in traffic on the way to the gym" from "scrolling Instagram instead of the gym." Inputs: location trajectory, motion signature, foreground-app history, calendar context, time-of-day prior. Output: COMPLIANT / DRIFTING / NON_COMPLIANT.
- *Personal drift fingerprint* — per-user model of when and where this user typically fails. Drives preemptive friction in the adaptation loop.
- *Trivial-commitment detection* — flags commitments that exist solely to game the score (e.g., "drink water" with no real stake).

**Data that actually matters:**
Foreground app events, location, motion, time-of-day, calendar entries, prior-compliance pattern by hour-of-week, biometrics from band (when present). On-device. Never leaves the phone.

**Data to ignore:**
Self-reported mood, journaling content, sentiment, "wellness" metrics. None of these predict execution. All of them invite users to feel productive instead of being productive.

**No LLM coaching chat.** A chat interface would let the user talk their way out of consequences. The system has no opinion the user can negotiate with.

---

## 7. Transition Plan

Four phases. Each phase ships a usable product. Each phase moves the system one layer deeper into the OS reframe.

### Phase 0 — Reframe (2–3 weeks)

The current MVP, relabeled and rewired around the Commitment object. Mostly a data-model and UX change; minimal new infrastructure.

**Build:**
- Rename `Task` → `Commitment`. Required fields: identity vector, time window, verification method, stake (can be $0 in this phase but field must exist), escalation policy.
- Replace the freeform task creator with a vector-first flow: pick vector → system suggests commitment templates → user customizes.
- Rename `Discipline Score` → `Identity Score`, split per declared vector.
- Replace `Dashboard` with `Drift Report`: shows declared vectors, observed compliance, the gap.
- Lock the Ledger: outcomes cannot be edited or deleted.

**Ignore:**
- Social features, badges, leaderboards, mood tracking, journaling, LLM chat.

**Why first:** Changes the product's meaning before changing its surface area. After Phase 0, the team and the user both stop thinking in tasks.

### Phase 1 — Always-On Runtime (4–6 weeks)

The app stops being session-based. It runs continuously and intervenes without being opened.

**Build:**
- iOS BackgroundTasks + significant-location-change triggers → Sensor Mesh runs while app is closed.
- Live Activity + lock-screen widget showing the next commitment window, time remaining, current verdict.
- Screen Time API integration → app lockouts during commitment windows for non-compliance.
- Stake holding: Stripe (or equivalent) authorizes a hold on commitment creation; forfeits on MISS, releases on HIT.
- Escalation chains as first-class objects (not hardcoded notification sequences).

**Ignore:**
- Adaptive inference (still rule-based in this phase).
- Wearable integration.
- Social/accountability features.

**Why second:** "Always running" is the single largest gap between the MVP and an OS. Without this, nothing else is real.

### Phase 2 — Identity Engine + Adaptive Inference (6–8 weeks)

The system stops treating all commitments equally and starts having an opinion about who the user is becoming.

**Build:**
- Identity vector declaration flow on onboarding (3–5 vectors, constrained list + custom with required verification).
- Per-vector Identity Score with weighted-recency computation.
- Drift Report: weekly auto-generated, surfaced as a required-acknowledgement screen the user cannot dismiss without action (tighten or amend).
- On-device classifier for the ambiguous-compliance call. Train on existing Ledger data + a labeled bootstrap set.
- Personal drift fingerprint: per-user model of failure timing. Drives preemptive friction.
- Enforcement-mode policy: vectors below score 60 auto-escalate stakes and tighten windows.

**Ignore:**
- Hardware. Cross-device. Network filtering.

**Why third:** Identity reframe only matters once the runtime is real. Built earlier, it'd be a marketing layer over a tracker.

### Phase 3 — Hard Enforcement Mesh (8–12 weeks)

The system reaches outside the phone. Consequences become unavoidable.

**Build:**
- Accountability Partner: invite flow, auto-generated miss reports sent via SMS/email on MISS. Partner cannot be removed mid-commitment.
- yourBand integration: BLE pairing, haptic escalation tiers, opt-in aversive stimulus tier (L4) gated behind a separate signed contract.
- Recovery contracts: when an Identity Score falls below 40, system offers a single recovery path with significantly higher stakes and tighter verification. Decline = vector demoted; the user must amend their declared identity.
- Cross-device lockout: macOS companion app for users on desktop during commitment windows.

**Ignore:**
- Network-level filtering. Calendar takeover. Anything that requires a separate platform team.

**Why last:** Hard enforcement is what users will resist hardest at signup. It only earns credibility once the earlier phases have demonstrated the system works.

### Phase 4 — OS-Level (later, post-PMF)

Calendar takeover (system writes blocks, user does not). Network-level filtering. Desktop runtime. Multi-user accountability circles. Public commitment APIs. None of this ships until Phases 0–3 have proven retention and identity-shift outcomes.

### What to ignore, permanently

- Gamification (badges, levels, leaderboards). Undermines the seriousness of the contract.
- Open-ended LLM chat coach. Gives the user something to argue with.
- Mood / wellness / journaling. Invites the feeling of progress without progress.
- Social feed of any kind. The system has no audience layer.
- Per-event opt-outs from enforcement. Defeats the entire premise.

---

## Closing principle

Every feature decision passes through one filter: *does this make the gap between who the user said they are and what they did today smaller, or does it let the gap persist comfortably?* If it makes the gap comfortable, cut it. The OS exists to make that gap unlivable.
