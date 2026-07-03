# Prompt: Extract API Response Types from a React Repo for Mock Generation

You are building the extraction engine behind a mock-generation app. Given any React/TypeScript repository, the engine detects every API call the app makes and extracts the TypeScript type of each response payload, so that realistic mocks can be generated automatically.

The app's UI lists every detected endpoint, and **each endpoint has a Regenerate button**. The engine must therefore support fast, targeted, per-endpoint regeneration — not just one-shot whole-repo extraction. Design everything below with that in mind.

## Existing system context — read this first

This is **not a greenfield build**. The application already works today using a HAR-based pipeline: recorded HTTP traffic is parsed into endpoint records and mock responses. That pipeline stays. Your job is to reorganize the project so static type extraction becomes a **second schema provider alongside HAR**, behind a shared interface — not a replacement engine.

Note that the HAR pipeline is already the "runtime capture" half of the two-pass strategy described under Clever Extras below. You are adding the static half and a reconciliation layer, nothing more.

Hard constraints:

- Do not rewrite or degrade the HAR pipeline. It remains the source of truth for runtime-observed data and the only path for untyped code.
- Every phase below must be independently shippable, and the static provider must be behind a feature flag until Phase 2 is proven.

Phased plan — do these in order, stop and ship after each:

- **Phase 0 — shared model, no behavior change.** Define a provider-agnostic endpoint record (the JSON shape below) and refactor the HAR pipeline to emit it. Before touching anything, write golden tests over existing HAR fixtures so the refactor is provably behavior-preserving.
- **Phase 1 — static provider as one-shot enrichment.** A CLI (no service, no watcher, no warm compiler) that scans the repo and emits the same records. Merge with HAR records by endpoint ID; in the UI, static types appear as annotations on existing endpoints plus any endpoints HAR never saw.
- **Phase 2 — reconciliation + Regenerate integration.** Implement the conflict policy (below) and let the Regenerate button use the merged schema.
- **Phase 3 — warm compiler service.** Long-lived program, file watcher, deep-regenerate mode. Only build this once Phases 1–2 have proven value; reroll mode needs none of it.

Reconciliation rules (the genuinely new logic):

- HAR gives concrete URLs (`/api/users/42`); static gives patterns (`/api/users/:id`). Match by method + route-pattern matching; when a static pattern matches multiple HAR entries, they are the same endpoint.
- Where both provide a schema and they disagree, do not silently pick a winner. Surface the disagreement in the record (`"schemaConflict": {...}`). Default policy: HAR wins for field presence and example values (it is ground truth of what actually flew); static wins for optionality, unions, and fields HAR happened not to capture.
- Where static extraction yields `any`/unknown, fall back to the HAR-inferred schema — this is the existing behavior, unchanged.

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

## Clever extras — implement these too

- **Untyped calls fallback (two-pass strategy).** `fetch(...).then(r => r.json())` returns `any`; static analysis is a dead end there. Fall back to runtime capture: run the app with MSW in passthrough-record mode, capture real responses, and infer schemas from the captured JSON with `quicktype`. Use static extraction where the code is typed, runtime capture where it is not, and merge the results.

- **Synthetic type trick for stubborn cases.** When unwrapping fails (deeply generic wrappers, conditional types), create an in-memory source file containing `type __Extract = Awaited<ReturnType<typeof theCall>>` and ask the checker to expand `__Extract`. The compiler performs all generic resolution for free.

- **tRPC / GraphQL short-circuit.** Detect these frameworks first — the router or schema *is* the complete type inventory. Extract types directly from the tRPC router definition or the GraphQL schema/codegen output and skip AST walking entirely for those calls. Cheap, complete win.

- **Reuse `ts-json-schema-generator`** for named/exported types — it already does the type→JSON Schema conversion well. Hand-roll the structural walker only for anonymous inferred types it cannot handle.

## Deliverables

1. A long-lived extraction service (library + small HTTP API) that:
   - performs the initial whole-repo scan and emits the JSON records described above,
   - keeps the compiler program warm and serves `POST /endpoints/:id/regenerate` with both `reroll` and `deep` modes,
   - watches source files and marks affected endpoints stale.
2. A CLI wrapper (`extract-api-types <repo-path>`) over the same library for one-shot use.
3. A generator step that turns those records into an MSW handler file plus mock fixtures.
4. Tests against a fixture repo covering: raw fetch, axios with generics, a wrapped client, react-query hooks, an untyped call (verifying the fallback path is reported), a cyclic type, and regeneration (reroll changes data but not schema; deep regenerate after editing a type changes `schemaHash` and reports a diff; endpoint IDs stay stable across unrelated edits).

Key principle throughout: symbol resolution plus type-checker interrogation is the core mechanism. Regex finds roughly 60% of call sites; the compiler finds all of them, with types nobody had to write.
