This service is now fully built and tested per its version's checklist. Before moving on, generate a compact operational reference for it — NOT more code, NOT a restatement of the architecture or design rationale. Just a cheat-sheet I can save and use later to manually exercise this service's endpoints without re-reading the source.

Base everything strictly on the actual code already generated in this chat for this service. Do not guess, do not invent an endpoint, header, or field that isn't literally present in the code. If something about a header or precondition is genuinely ambiguous from the code as written, say so in one line rather than assuming.

For EVERY REST endpoint this service exposes (skip nothing, don't list anything twice), output one compact block in exactly this shape:

### METHOD /path — one-line purpose

- Direct call (this service's own port): state whether this is even possible given any InternalSecretFilter present in this version's code, and if so, the exact header(s) with placeholder values needed to pass it — e.g. X-Internal-Secret: <value>
- Via api-gateway (if this version has one): the exact header(s) an external caller must set — e.g. Authorization: Bearer <jwt> — or "none, on the gateway's public whitelist" — and note if this endpoint even routes through the gateway in this version
- Request body (if any): a minimal realistic JSON example with plausible values, not a schema description
- Success response: status code + a minimal realistic JSON example
- Notable error responses: status code — one-line trigger condition, for each custom exception this endpoint can throw (skip the generic 500 fallback, that's identical everywhere)
- Preconditions / call order: anything that MUST already exist or MUST be called first — including data that must already exist in a DIFFERENT service. Infer this from any Feign client calls in this service's own code (e.g. "a category and product must already exist in product-service" if this service calls product-service's GET /api/products/{id}) — don't fabricate the other service's endpoint shape, just name what's needed. Write "none" if there are genuinely no preconditions.

After all endpoints, add one final section titled "Known gotchas for manual testing" — 3 to 6 bullet points max, only things that would trip someone up calling this service in isolation via Postman (e.g. "userId in the path isn't validated against anything in this version — any Long works" or "the internal-secret filter rejects even a valid JWT if X-Internal-Secret is also missing").

Constraints:
- No prose outside the structure above. No code blocks except the JSON examples.
- Keep every JSON example to the minimum fields needed to show the shape — don't pad with every optional field.
- Total output should be scannable in under a minute per service — favor brevity over completeness of prose if the service has many endpoints.