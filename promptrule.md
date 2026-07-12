# PROMPT RULES

> This file is a rulebook, not a per-service spec. `architecture-vX.md` tells the AI *what* to build for the current version; `services-index.md` tells it *what else exists*; this file tells it *how to behave regardless of what it's building*.
>
> Important: Amazon Q and Copilot chats don't automatically read files from your repo unless your IDE session has them open in a way the tool's context window picks up. Don't assume pasting `architecture-vX.md` and `services-index.md` is enough on its own — paste the relevant rules below into the prompt itself too (or at minimum the ones marked **[always paste]**), especially for fix prompts where scope discipline matters most.

---

## 1. Scope discipline — the most important section

- **[always paste]** Do not modify any file belonging to a different service. Every prompt targets exactly one service (or, for a fix prompt, exactly one file within one service). If a change genuinely requires touching two services, that will be stated explicitly as two separate prompts — never assume it's implied.
- **[always paste]** Only output the files explicitly listed in the prompt's "output every file in full" instruction (or, for a fix prompt, only the specific broken file). Do not regenerate, restate, or "helpfully" touch any file not named.
- Do not add endpoints, fields, DTOs, dependencies, or configuration beyond what's explicitly specified in the current prompt — even if it seems like an obvious or small addition. If something looks missing, flag it back to me in your response rather than adding it silently.
- Do not implement a feature from a later version ahead of schedule (e.g. don't add JWT while building v1's user-service just because "it's more secure" — v1's user-service is intentionally using a simple in-memory session, and that's a stated, deliberate choice, not an oversight to correct).

## 2. Consistency rules

- Package names, class names, port numbers, and application.yml values must match exactly what's specified in the architecture file — do not rename anything for "clarity" or convention reasons.
- Once an enum-like status value (e.g. `Order.status` values `CART`, `PLACED`, `PENDING_PAYMENT`, `PAID`, `PAYMENT_FAILED`, `PAYMENT_PENDING_RETRY`) has been introduced in an earlier version, all of its existing values must be preserved when a later version adds new ones. Never drop or rename a previously-existing value while adding new ones — check `services-index.md` and the relevant earlier `architecture-vX.md` if unsure what already exists.
- Entity field names and types, once established, do not change in a later version unless that version's architecture file explicitly says so. If a later version needs a new field, it's additive, not a rename or a type change to an existing field.
- Follow whatever pattern an earlier service already established for equivalent situations (e.g. the `GlobalExceptionHandler` shape, the custom-exception-per-error-case pattern, the DTO-not-raw-entity response pattern) rather than inventing a new pattern for the same kind of problem.

## 3. Resilience and error-handling boundaries

- Do not add retry logic, circuit breakers, timeouts, or fallback behavior on any inter-service call unless the current version's architecture file explicitly introduces it. Plain, unguarded calls are correct for early versions on purpose — resilience is deliberately deferred to v5, not an oversight to "fix" early.
- Do not add caching anywhere unless explicitly instructed. Recomputing from live calls is the deliberate v1–v4 default.
- Every new custom exception must be wired into that service's own `GlobalExceptionHandler` (or `WebExceptionHandler` for frontend-service) with a specific HTTP status — never let a business-logic error fall through to the generic 500 handler if a more specific one is called for.

## 4. Fix-prompt specific rules (Copilot error resolution)

- **[always paste]** When fixing an error, only the file(s) actually causing the error should be touched. Do not "clean up" or refactor adjacent working code while fixing an unrelated bug.
- Include the exact error message/stack trace, the specific broken file's current content, and nothing else from the rest of the codebase — do not paste the whole service's source as context for a fix unless the error genuinely can't be understood without it.
- If a fix requires understanding another service's contract (e.g. a deserialization mismatch with product-service's response shape), pull that contract from `services-index.md` or the relevant `architecture-vX.md`'s endpoint table — never paste the other service's actual source code as context for a fix in a different service.
- After a fix, re-run only the specific test case(s) from the version's testing checklist that relate to the broken behavior — don't treat every fix as a reason to re-test the entire service from scratch, unless the fix touched genuinely shared/core logic (e.g. the exception handler itself).

## 5. Testing gate rule

- **[always paste]** Do not consider a service "done" until every item in its version's testing checklist (in `architecture-vX.md`) passes. Do not begin the next service's construction prompt until the current one is marked `Built & Tested` in `services-index.md`.
- If a test fails, the fix belongs to the service that failed — do not "fix" a failing order-service test by changing product-service's behavior unless the architecture file's contract for product-service was itself wrong (in which case, flag that explicitly rather than quietly patching around it).

## 6. Output style rules

- Prefer being told "the response was cut off after file X" over receiving a shortened or summarized version of a file — never truncate or abbreviate a file's content with a comment like "// rest of the code is similar." Full files only, every time.
- If a prompt's scope is genuinely too large for one response, say so explicitly and generate as much as fits cleanly (a complete file, not a partial one) rather than silently truncating mid-file.
