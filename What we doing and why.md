# PROJECT METHODOLOGY & DECISION LOG

> This file is for humans only — it is never pasted into Amazon Q, Copilot, or any AI tool. Its only job is to explain *why* the project is structured the way it is, so that anyone reading the repo (a mentor, a viva panel, or future-you six months from now) understands the reasoning, not just the output. The `architecture-vX.md` files explain *what* to build; this file explains *why we're building it this way at all*.

---

## 1. The constraint that shaped everything

This project was built during a fixed-length internal training program (Capgemini, prompt-engineering track, ending 22 July), on a company laptop with no ability to install or use Claude, Gemini, or any general-purpose AI tool. The only two tools available were:

- **GitHub Copilot Business**, metered on a fixed monthly token-based credit allowance
- **Amazon Q Developer (Builder free tier)**, metered on a fixed count of 50 total prompts, not tokens

Every methodological decision below exists because of this specific pairing: one tool that's expensive per unit of *work generated* (Copilot), and one that's expensive per unit of *conversation turn* (Amazon Q), with a hard 10-day deadline sitting on top of both. A different tool combination, or unlimited credits, would not have produced this same structure — it's worth stating that plainly so the reasoning doesn't get mistaken for "best practice in general" rather than "the right call under these specific limits."

## 2. Why the project is versioned instead of built as one continuous build

The single biggest risk in a fixed-credit, fixed-deadline solo project is running out of resources midway through an ambitious architecture and being left with a half-finished codebase — too complex to explain, and with nothing demonstrable to show for the credits already spent.

The response to that risk was to structure the project as five deliberately incremental versions (v1 through v5), each one a **complete, working, demoable state on its own**:

- v1 is not a stub or a skeleton — it's a real, working, end-to-end e-commerce flow (register, browse, cart, checkout, order history) across five services. It's just *architecturally minimal*: no gateway, no real auth, no recommendations.
- Every version after it adds one coherent theme on top of a system that already works, rather than adding many things at once.

The practical consequence: if credits run out after v2, v3, or v4, there is always a fully working, testable, explainable project to submit — just a smaller one than hoped for, not a broken one. The versioning isn't about following an agile methodology for its own sake; it's a direct hedge against the specific failure mode of a resource-constrained solo build.

## 3. Why five versions specifically, and why this particular grouping

Three was rejected as too coarse — it would force v1 to contain too much at once (the whole bedrock plus at least one advanced feature), which defeats the purpose of having a "safe, demoable" early stopping point. Eight was rejected as too granular — more version boundaries means more overhead (more architecture files, more "what changed" transitions to explain), for versions thin enough that some wouldn't deserve a dedicated slot of their own.

The five landed on this way:

| Version | Theme | Why this grouping |
|---|---|---|
| v1 | Working bedrock | The absolute minimum for a real user journey — proves the five core services and their one inter-service call pattern work before anything else is layered on |
| v2 | Gateway + real auth + inventory | These three are grouped together because they all change *how requests reach services and what's trusted*, not what the services do — a single conceptual theme (the request path), not three unrelated features |
| v3 | Recommendation v1 | The first AI-flavored feature, deliberately isolated to its own version and kept rule-based rather than combined with v4's smarter version — this lets "does a recommendation engine exist at all" be proven before "is it any good" is attempted |
| v4 | Recommendation v2 + payment + notifications | Grouped because all three only make sense *after* v1–v3 have generated real order/view history and a gateway to route through — none of them are meaningful in isolation earlier |
| v5 | Resilience + polish | Deliberately contains zero new business features — everything here hardens or documents what already exists, which is why it's the version most acceptable to leave partially done if time runs out |

The ordering itself is a decision worth naming: recommendations (v3) come before payment (v4) even though a "real" e-commerce project might prioritize payment first, because payment and notifications only become interesting once there's an existing checkout flow to attach them to, and the recommendation engine was the actual differentiating/AI-flavored feature this project needed to demonstrate — so it was pulled earlier rather than risk it being the first thing cut if time ran short.

## 4. Why rule-based recommendations before "smart" recommendations, rather than building the smart version once

v3 uses simple co-occurrence counting and category fallback — no machine learning, no similarity metrics, no external libraries. v4 upgrades this to weighted scoring using purchase and view counts, still with no ML.

This was a deliberate two-step approach rather than jumping straight to a more sophisticated algorithm, for three reasons:

1. **Data volume.** At the point v3 is built, the project has only whatever test orders were manually created — nowhere near enough volume for a collaborative-filtering or similarity-based approach to produce meaningful output rather than noise.
2. **Explainability.** A viva question like "walk me through exactly how this recommendation was produced" needs an answer that can be done on paper. A hand-counted co-occurrence tally and a small weighted-sum formula both satisfy that; a trained model or a similarity matrix does not, without a much deeper explanation of the underlying math.
3. **Credit risk.** A "smart" approach that doesn't work first time is much harder to debug than a rule-based one, because there's more surface area for something to go subtly wrong (data shape, normalization, library integration) versus a bug in a `Map<Long, Integer>` tally, which is usually visible by inspection.

## 5. Why the three-tier documentation system exists

The core problem this system solves: an AI coding assistant needs *just enough* context to work correctly on the current task, without re-reading (and re-paying token cost for) everything that's ever been decided in the whole project. Three tiers, each solving a different piece of that:

- **`architecture.md`** (this file, plus a fuller narrative version) — human-only, unlimited length, never read by any AI. Exists so the *reasoning* behind decisions is preserved somewhere, since none of the AI-facing files need to (or should) carry that reasoning at the depth a person might want.
- **`services-index.md`** — a permanent, tiny (~30–40 line) file that's present in every single AI prompt regardless of which version or service is being worked on. Exists to solve the specific problem of: "a service being built in v3 needs to know a service built in v1 exists and what it exposes, without reading v1's full architecture file or its code." It's the cheapest possible way to carry cross-service awareness forward indefinitely.
- **`architecture-vX.md`** — one deep-dive file per version, swapped out as work moves from version to version. Exists to carry the *depth* (exact entity fields, validation rules, endpoint contracts) needed to prompt an AI precisely, scoped only to what's new or changed in that version, so it doesn't have to re-explain everything from every prior version each time.

The alternative to this system would be either (a) re-explaining relevant context by hand in every new AI chat, which is slow and error-prone, or (b) keeping one continuously growing conversation thread, which — since these chat-based tools resend the entire conversation history as input on every turn — becomes progressively more expensive per message the longer the conversation runs. The three-tier file system is the fix for both problems at once: bounded, explicit, versioned context instead of either manual re-explanation or an ever-growing thread.

## 6. Why one Amazon Q prompt per service, in a new chat each time

This follows directly from the point above about growing conversation threads. Since Copilot's cost is now token-metered (input + output), and every additional turn in the same thread re-pays for the entire prior conversation, the single highest-leverage habit available is: **start a new chat per service**, so that building service #5 doesn't carry the accumulated token cost of the four services built before it.

This is also why the architecture files are split *per version* rather than *per service* — splitting per service would mean a service built in isolation has no cheap way to know what else exists in the project (the exact problem `services-index.md` is designed to solve), whereas splitting per version keeps enough surrounding context (what else this version adds, why) without needing the full history of every prior version.

## 7. Why the prompts are written in dense, spec-style language instead of natural, conversational language

Every v1 prompt follows the same shape: exact package names, exact class names, exact field names/types/annotations, exact endpoint contracts, then an explicit list of what *not* to add. This is a deliberate trade against natural, conversational prompting, for one reason: **the more a prompt leaves to the model's judgment, the more surface area there is for an unexpected addition, omission, or guess.**

Concretely, this specificity does two things at once:
1. It reduces the chance the generated code diverges from the architecture file's spec in a way that breaks a later version's assumptions (e.g. a differently-named field, an extra endpoint that wasn't asked for)
2. It reduces output token usage, since the model doesn't need to "invent" structure, DTOs, or additional validation on its own — everything it needs to decide has already been decided for it

The explicit "do not add X" constraints in every prompt exist specifically to counter AI models' tendency to be "helpfully" over-generous — adding extra fields, extra endpoints, or extra validation that wasn't requested, which costs credits and adds untracked surface area to a project that's supposed to match its architecture file exactly.

## 8. Why prompts are split by natural service boundaries, with explicit overflow contingencies rather than pre-guessed splits

Each version's build is broken into prompts along service boundaries (one service = one prompt, as a default), rather than pre-emptively splitting every service into multiple smaller prompts. The reasoning: splitting too early wastes prompts on services that would have fit in one shot anyway (costly on Amazon Q's fixed 50-prompt budget), while not splitting at all risks a single prompt's output getting cut off mid-generation with no easy way to resume cleanly.

The compromise: build one prompt per service by default, but have an explicit, pre-planned overflow split ready (e.g. "entities+repos" vs "controllers+exception handling") for any service where the output volume is a real risk (product-service, order-service) — decided in advance so that if truncation happens, the fix is "run the pre-planned second half," not "figure out on the fly where the code got cut off."

The one genuine exception to "one service = one prompt" is frontend-service, split into Part A (browsing) and Part B (auth/cart/checkout/history) — not because either half is individually complex, but because the *combined* file/page count was large enough that generating it all in one response risked exactly the kind of truncation the overflow strategy above is meant to avoid, and splitting cleanly along a "read-only" vs "stateful" boundary was a natural, pre-plannable seam rather than a guess.

## 9. Why every service must be independently testable before the next one is started

Each version's architecture file ends with a testing checklist, and the working agreement is: don't start building the next service until the current one passes its checklist. This exists to solve the exact failure mode described at the very start of this project's planning — reaching a point mid-project where something is broken and it's unclear whether the cause is the service currently being built or something several services back that was never actually verified.

Testing gate-by-gate turns "the whole system either works or it doesn't" into "each service is confirmed correct in isolation, so if something breaks later, the search space for the cause is small" — which matters more under credit constraints than it would with unlimited iteration budget, since diagnosing a bug against an unverified foundation costs the same credits as diagnosing it against a verified one, but takes much longer and is much more likely to send you chasing the wrong layer.

## 10. Why some deliberately "textbook-incorrect" choices were made, and why that's stated openly rather than hidden

Several decisions in this project intentionally diverge from how a production system would actually be built:

- Direct synchronous REST calls between services instead of an event-driven architecture with a message broker (no Docker available, so no Kafka/RabbitMQ)
- A separate database per service, but all hosted on one shared Postgres server instance, instead of genuinely independent database servers per service
- Manual, synchronous compensation logic (restock-on-payment-failure) instead of a proper saga/distributed-transaction framework
- No refresh tokens, no role-based authorization enforcement despite a role claim existing in the JWT
- Downstream services trusting the gateway's JWT validation instead of independently re-validating

None of these are oversights — each is a stated trade-off, made consciously against the project's real constraints (no Docker, limited time, limited credits, solo build). The decision to document them explicitly (both here and in each version's "Known Limitations" section) rather than leave them unstated is itself a deliberate choice: being able to say "I knew the textbook-correct approach and chose not to take it, for this specific reason" is a stronger position in a technical review or viva than a project that either pretends every trade-off was accidental, or over-engineers a "correct" solution to a problem the project's actual constraints didn't require solving.