# Prompt: Extract API Response Types from a React Repo for Mock Generation

You are building the extraction engine behind a mock-generation app. Given any React/TypeScript repository, the engine detects every API call the app makes and extracts the TypeScript type of each response payload, so that realistic mocks can be generated automatically.

The app's UI lists every detected endpoint, and **each endpoint has a Regenerate button**. The engine must therefore support fast, targeted, per-endpoint regeneration — not just one-shot whole-repo extraction. Design everything below with that in mind.

## Existing system context — read this first

This is **not a greenfield build**. The application already works today using a HAR-based pipeline: recorded HTTP traffic is parsed into endpoint records and mock responses. That pipeline stays. Your job is to reorganize the project so static type extraction becomes a **second schema provider alongside HAR**, behind a shared interface — not a replacement engine.

Note that the HAR pipeline is already the "runtime capture" half of the two-pass strategy described under Clever Extras below. You are adding the static half and a reconciliation layer, nothing more.

Hard constraints:

- Do not rewrite or degrade the HAR pipeline. It remains the source of truth for runtime-observed data and the only path for untyped code.
- Every phase below must be independently shippable, and the static provider must be behind a feature flag until Phase 2 is proven.

Feature flag behavior:

- This is a local dev tool — use a plain config flag in the app's existing settings store (`staticExtraction.enabled`, default `false`), not a remote flag service. CLI equivalent: `--static` flag or `MOCKGEN_STATIC=1` env var.
- Gate at **provider registration**: when the flag is off, the static provider is never instantiated and `ts-morph`/`typescript` are never loaded (lazy `import()` inside the provider — the dependency is heavy).
- Tag every endpoint record with `source: "har" | "static" | "merged"`. Flag off ⇒ static and merged records are excluded from the UI and from mock serving; the HAR path behaves byte-for-byte as it does today. Turning the flag off must fully restore current behavior with no data migration — static records become inert, never destructive.
- Phase 2 reconciliation sits behind a sub-flag (`staticExtraction.reconcile`) so enrichment-only mode (Phase 1) can ship and be evaluated independently.
- CI runs the HAR golden tests in **both** flag states; flag-on with no static data must be indistinguishable from flag-off.
- Flipping the default to `true` is its own release, after Phase 2 is validated — never bundled with feature work.

Phased plan — do these in order, stop and ship after each:

- **Phase 0 — shared model, no behavior change.** Define a provider-agnostic endpoint record (the JSON shape below) and refactor the HAR pipeline to emit it. Before touching anything, write golden tests over existing HAR fixtures so the refactor is provably behavior-preserving.
- **Phase 1 — static provider as one-shot enrichment.** A CLI (no service, no watcher, no warm compiler) that scans the repo and emits the same records. Merge with HAR records by endpoint ID; in the UI, static types appear as annotations on existing endpoints plus any endpoints HAR never saw.
- **Phase 2 — reconciliation + Regenerate integration.** Implement the conflict policy (below) and let the Regenerate button use the merged schema.
- **Phase 3 — warm compiler service.** Long-lived program, file watcher, deep-regenerate mode. Only build this once Phases 1–2 have proven value; reroll mode needs none of it.

Where the code lives:

- The extraction engine is a **new package in the existing monorepo** (e.g. `packages/extraction`), alongside `cli`, `desktop`, and `shared`. Not a separate repo, and not a rewrite of any HAR code.
- The provider-agnostic endpoint record and the `SchemaProvider` interface (Phase 0) go in `shared`. Both the HAR pipeline and the new engine implement/emit against those types.
- `cli` gains a thin command wrapping the engine for Phase 1 one-shot scans. `desktop` consumes it through the provider interface from Phase 2 on.
- Keep `typescript`/`ts-morph` as dependencies of `packages/extraction` only — never in `shared`, and never loaded in the desktop renderer. If the desktop app is Electron, run extraction in the main process or a child/worker process; the warm compiler program (Phase 3) especially must not live in the renderer.
- Reuse the existing mock-gen pipeline pieces (schema store, faker/mock emission, fixture writing) rather than duplicating them; the engine's job ends at producing endpoint records with schemas.

Reconciliation rules (the genuinely new logic):

- HAR gives concrete URLs (`/api/users/42`); static gives patterns (`/api/users/:id`). Match by method + route-pattern matching; when a static pattern matches multiple HAR entries, they are the same endpoint.
- Where both provide a schema and they disagree, do not silently pick a winner. Surface the disagreement in the record (`"schemaConflict": {...}`). Default policy: HAR wins for field presence and example values (it is ground truth of what actually flew); static wins for optionality, unions, and fields HAR happened not to capture.
- Where static extraction yields `any`/unknown, fall back to the HAR-inferred schema — this is the existing behavior, unchanged.

### Route pattern matching

Use the existing `path-to-regexp` dependency as the matching primitive — do not write a custom matcher. The library only solves pattern → regex → test; the algorithm around it is:

1. **Normalize both sides.** Strip origin/base URL from HAR entries (configurable base-URL list), drop query strings (store them separately on the record), normalize trailing slashes and case. On the static side, convert template literals (`` `/api/users/${id}` ``) to `:param` patterns before feeding path-to-regexp.
2. **Direction:** for each concrete HAR URL, test against all static patterns with the same HTTP method.
3. **Specificity ranking — the only custom code (~20 lines).** `/api/users/new` matches both the literal `/api/users/new` and `/api/users/:id`. Rank like a router: more literal segments wins, then fewer params, then longer pattern. Deterministic tie → pick first and emit a warning on the record so the ambiguity is visible, never silent.
4. Unmatched HAR URLs remain concrete-URL endpoints (current behavior). Unreducible static call sites remain file+symbol orphans. Both are expected outcomes, not errors.

### Schema conflict record shape

Conflicts are per-field and JSON-pointer keyed — a whole-schema blob is useless to the UI. Compute them with a structural walk of both JSON Schemas, comparing at each pointer:

```json
{
  "responseSchema": { "...merged result — what mock generation actually uses..." },
  "schemaConflict": {
    "detectedAt": "2026-07-03T12:00:00Z",
    "harSampleCount": 14,
    "conflicts": [
      {
        "path": "/properties/user/properties/age",
        "kind": "type-mismatch",
        "static": { "type": "number" },
        "har": { "type": "string" },
        "resolution": "har",
        "overridden": false
      }
    ]
  }
}
```

`kind` values and default resolution:

| kind | meaning | default winner |
| --- | --- | --- |
| `missing-in-static` | HAR saw the field, types don't declare it | HAR (types are stale) |
| `missing-in-har` | declared, never observed | static (HAR sample bias is not proof of absence) |
| `type-mismatch` | both present, types differ | HAR (ground truth) |
| `optionality` | required vs sometimes-omitted disagreement | static |
| `union-narrowing` | HAR only ever saw one union branch | static (keep the full union) |

Rules: `responseSchema` is always the merged schema, so mock generation never interprets conflicts. `resolution: "manual"` marks a user override from the UI; overrides persist keyed by `endpointId + path` so deep regeneration never clobbers a user decision. `harSampleCount` lets the UI qualify confidence ("1 sample" vs "200 samples").

## Core insight

Do not parse or infer types yourself, and do not rely on regex to find API calls. Make the TypeScript compiler do both jobs — it already knows every inferred type in the project. Use `ts-morph` (a wrapper around the TypeScript compiler API), load the repo via its own `tsconfig.json`, and interrogate the type checker at each API call site.

## Pipeline to implement

1. **Find call sites by symbol resolution, not text matching.**
   Grepping for `fetch` or `axios` breaks on wrappers and re-exports. Instead, walk every `CallExpression` in the AST and resolve the called expression's symbol back to its original declaration. This correctly catches:
   - `import { api } from './client'` style wrappers around fetch/axios
   - aliased or re-exported HTTP clients
   - data-fetching hooks (`useQuery`, `useSWR`, `useMutation`, etc.)

2. **Ask the type checker for the return type, then unwrap containers.**
   Call `checker.getTypeAtLocation(callExpression)` and peel wrapper types until the payload type remains:
   - `Promise<T>` → `T` (use the awaited type)
   - `AxiosResponse<T>` → `T`
   - `UseQueryResult<T>` / `{ data: T }` shapes → `T`

   This works even when the type is fully *inferred* and never written down anywhere in the source.

3. **Serialize the structural type to JSON Schema.**
   Recursively walk `type.getProperties()`, handling unions, intersections, arrays, literal types, optionals, and enums. Track already-visited types to survive cycles (they exist in real codebases). JSON Schema is the target format because:
   - `json-schema-faker` turns it directly into realistic mock data
   - it is language-agnostic and easy to diff/version

4. **Extract the request metadata alongside the type.**
   For each call site, pull the URL argument (the checker can evaluate template literals to string literal types), the HTTP method, and the request body type if present. Emit one record per call site:

   ```json
   {
     "id": "GET /api/users/:id",
     "file": "src/api/users.ts",
     "line": 42,
     "method": "GET",
     "url": "/api/users/:id",
     "responseSchema": { ... },
     "requestBodySchema": { ... },
     "schemaHash": "sha256:...",
     "seed": 12345
   }
   ```

5. **Generate MSW handlers + mock data** from those records: one `http.get/post/...` handler per unique endpoint, with `json-schema-faker` producing the response body.

## Per-endpoint regeneration (drives the Regenerate button)

Every endpoint record needs a **stable identity** so the UI button can target it. Derive the ID from `method + normalized URL pattern` (fall back to `file + symbol name` for dynamic URLs the checker can't reduce to a literal). The ID must survive unrelated edits to the file — never key on line numbers alone.

The Regenerate button should support two modes:

1. **Reroll (default, instant).** Keep the cached `responseSchema`, feed `json-schema-faker` a new seed, emit fresh mock data. No compiler involvement — this must return in milliseconds. Persist the seed per endpoint so a given mock is reproducible until the next reroll.

2. **Deep regenerate (on demand / on file change).** Re-run type extraction for that endpoint only:
   - Keep the `ts-morph` `Project` alive in memory (creating a program is the expensive part; re-checking one file is cheap).
   - Refresh only the source file containing the call site (`sourceFile.refreshFromFileSystem()`), re-resolve the call expression, re-extract the schema.
   - Compare `schemaHash` before/after — if unchanged, tell the UI "schema identical, data rerolled"; if changed, surface a schema diff so the user sees what their code change did.

Expose this as a small API the app calls, e.g. `POST /endpoints/:id/regenerate { mode: "reroll" | "deep" }`, returning the updated record + mock payload. A file watcher can invalidate affected endpoints and mark them stale in the UI (badge on the button) rather than auto-regenerating.

## Reference skeleton

```ts
import { Project, Node } from "ts-morph";

const project = new Project({ tsConfigFilePath: "tsconfig.json" });
const checker = project.getTypeChecker();

for (const sf of project.getSourceFiles("src/**/*.{ts,tsx}")) {
  sf.forEachDescendant((node) => {
    if (!Node.isCallExpression(node)) return;

    const decl = node.getExpression().getSymbol()?.getDeclarations()?.[0];
    if (!isApiCall(decl)) return; // does the symbol resolve to fetch/axios/known client?

    // The compiler already inferred this — even with zero annotations
    let t = checker.getTypeAtLocation(node);
    while (t.getTypeArguments().length && isWrapper(t)) {
      t = t.getTypeArguments()[0]; // unwrap Promise<T>, AxiosResponse<T>, { data: T }...
    }

    const schema = typeToJsonSchema(t, checker); // recursive getProperties() walk
    const url = extractUrlArg(node, checker);    // checker evaluates template literals too
  });
}
```

## Clever extras — with the phase each belongs to

- **Untyped calls fallback (Phase 2 — already exists, do not build).** `fetch(...).then(r => r.json())` returns `any`; static analysis is a dead end there. The HAR pipeline *is* the runtime-capture half of this strategy — do **not** build an MSW passthrough-record mode, that would duplicate it. This extra reduces entirely to the Phase 2 reconciliation rule: where static yields `any`/unknown, the HAR-inferred schema wins. Zero new capture code.

- **Synthetic type trick for stubborn cases (Phase 1, fallback branch only).** When unwrapping fails (deeply generic wrappers, conditional types), create an in-memory source file containing `type __Extract = Awaited<ReturnType<typeof theCall>>` and ask the checker to expand `__Extract`. The compiler performs all generic resolution for free. Works fine in the one-shot CLI — no warm program needed. Don't build it preemptively; add it when the fixture repo shows unwrap failures, and until then leave a reported "couldn't unwrap" branch.

- **tRPC / GraphQL short-circuit (conditional — outside the phase ladder).** The router or schema *is* the complete type inventory, so extracting from it beats AST walking. But only build it if target repos actually use tRPC or GraphQL codegen. Phase 1 ships **detection only**: flag the dependency and report "N calls skipped (tRPC)". Build the real extractor as a follow-up only if that count is nonzero and users care.

- **Reuse `ts-json-schema-generator` (Phase 1, day one).** Not an add-on — it is *how* the type→JSON Schema step is done for named/exported types. Hand-roll the structural walker only for anonymous inferred types it cannot handle.

## Deliverables

1. A long-lived extraction service (library + small HTTP API) that:
   - performs the initial whole-repo scan and emits the JSON records described above,
   - keeps the compiler program warm and serves `POST /endpoints/:id/regenerate` with both `reroll` and `deep` modes,
   - watches source files and marks affected endpoints stale.
2. A CLI wrapper (`extract-api-types <repo-path>`) over the same library for one-shot use.
3. A generator step that turns those records into an MSW handler file plus mock fixtures.
4. Tests against a fixture repo covering: raw fetch, axios with generics, a wrapped client, react-query hooks, an untyped call (verifying the fallback path is reported), a cyclic type, and regeneration (reroll changes data but not schema; deep regenerate after editing a type changes `schemaHash` and reports a diff; endpoint IDs stay stable across unrelated edits).

Key principle throughout: symbol resolution plus type-checker interrogation is the core mechanism. Regex finds roughly 60% of call sites; the compiler finds all of them, with types nobody had to write.
