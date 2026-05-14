# Contributing to GreyC/OpenCLI

This fork's PRs are eventually upstreamed to `jackwener/OpenCLI`. The checklist below captures conventions the upstream maintainer reliably enforces by amending PRs before merging (observed across PRs #1538–#1542). Front-loading them shortens review.

This file is auto-loaded by Claude Code and read by most AI coding tools (Cursor, Codex, Aider, etc.). Human contributors should read it before opening a PR.

## Before submitting a PR — checklist

### 1. Use typed errors, never `throw new Error(...)`

In `clis/*` and `src/*`, import from `@jackwener/opencli/errors`:

| Error | When |
|-------|------|
| `ArgumentError` | Invalid CLI arg (range, format, required). Throw **before** `page.goto`. |
| `AuthRequiredError` | Cookie expired / user not logged in (maps to exit 77). |
| `EmptyResultError` | Valid request, empty result set. |
| `CommandExecutionError` | Unexpected API shape, missing field, malformed payload. |

Untyped `Error` collapses to a generic `COMMAND_EXEC` exit code and removes the caller's ability to differentiate "retry" vs "ask user to log in" vs "no data".

### 2. Validate args fail-fast, before any navigation

```js
import { readPositiveInteger } from './utils.js'; // boss adapter; similar helpers exist per-site

const limit = readPositiveInteger(kwargs.limit, 'mycmd --limit', 20, 100);
const pageNum = readPositiveInteger(kwargs.page, 'mycmd --page', 1);
```

Reject bogus inputs before `page.goto` so they don't waste a browser context.

### 3. Prefer imperative `func:` over declarative `pipeline:`

Declarative `pipeline: [{navigate}, {evaluate}]` is fine for trivial scrapers but **cannot express error classification** (auth vs empty vs malformed all collapse). Use `func: async (page, kwargs) => { ... }` whenever you need to:

- Distinguish error categories
- Branch on intermediate state
- Call multiple endpoints in sequence

### 4. After `JSON.parse`, type-check the parsed value

`JSON.parse` succeeding does not mean you got an object. `"null"`, `"true"`, `42`, `"[1,2]"` are all valid JSON that break `data.code` lookups.

```js
const data = JSON.parse(text);
if (!data || typeof data !== 'object') {
    throw new CommandExecutionError('API returned malformed response');
}
if (!Array.isArray(data.zpData?.messages)) {
    throw new CommandExecutionError('Expected messages array');
}
```

### 5. Multi-source merge — merge first, filter last

When combining a "primary" source with a "fallback" source, **do not gate the fallback on primary's count**. Primary may pass the count check but fail downstream filtering, leaving you short.

```js
// Bad: skips fallback when primary has enough raw nodes
const combined = primary.length >= limit ? primary : [...primary, ...fallback];

// Good: always merge, let the row-level filter be the final gate
const combined = dedupe([...primary, ...fallback]);
```

### 6. `Array.find` smell — explicit scored selection

When you write `xs.find(x => ...)`, ask: "if multiple match, which is correct?"

- If answer is "any" → `find` is fine.
- If answer is "the focused one" / "the newest" / "the one with X" → write an explicit scored sort and pick `[0]`. Otherwise behaviour is order-dependent and surprising.

### 7. Long-lived sockets + async handlers — guard stale messages after every await

If a handler crosses an `await` boundary, the global state it depends on may have changed by the time it returns. Re-check identity **after every await**, not just at entry.

```js
thisWs.onmessage = async (event) => {
    if (ws !== thisWs) return;              // entry guard
    const result = await handleCommand(...); // long await
    if (ws !== thisWs) return;              // re-check
    safeSend(thisWs, result);
};
```

Also: `safeSend` checks `readyState === OPEN`, not "not CLOSING/CLOSED" — `CONNECTING` is also not sendable.

### 8. Tests cover error paths, not just happy paths

Every typed error you throw should have a test:

```js
await expect(command.func(page, badInput)).rejects.toBeInstanceOf(ArgumentError);
```

Use the substring-dispatch mock pattern from `clis/boss/*.test.js`:

```js
page.evaluate.mockImplementation(async (script) => {
    if (script.includes('myEndpoint')) return mockPayload;
    if (script.includes('document.cookie')) return 'fake-cookie';
    return {};
});
```

### 9. Docs — when introducing limits, name every lifecycle subtype

If a concept has multiple lifecycles (owned vs bound, persistent vs ephemeral) and you mention a timeout/quota/limit, **specify which subtype it applies to in the same sentence**. Readers default to assuming the strictest variant applies to all.

- Bad: "Browser sessions use a 10-minute idle timeout."
- Good: "**Owned** browser sessions use a 10-minute idle timeout. **Bound** sessions have no idle timer; they stay attached until …"

### 10. After adapter changes, rebuild the manifest

```bash
npm run build-manifest
git add cli-manifest.json
```

CLI option parsing reads from the **committed** `cli-manifest.json`, not from the adapter source at runtime. If you add `--side`, `--limit`, or any new option and forget to rebuild, the new flag is silently ignored when users run the CLI.

## Verification before opening PR

```bash
npm run build-manifest                          # if you touched clis/<site>/*.js or args
npx vitest run clis/<site>/                     # adapter tests
npx vitest run --project unit                   # unit tests
npm run typecheck -- --pretty false             # type check
git diff --check                                # whitespace
```

## Related files

- `clis/boss/utils.js` — reference for adapter helpers (`bossFetch`, `readPositiveInteger`, `assertOk`, `checkAuth`)
- `src/cli.ts` — the single-file CLI definition. Browser commands at lines 666+.
- `extension/src/background.ts` — Chrome extension service worker. Workspace lease policy at lines 209–261.
- `docs/guide/browser-bridge.md` — browser session lifecycle docs (owned vs bound)
- Memory: `~/.claude/projects/-Users-gangchen-Documents-Projects-OpenCLI/memory/feedback_opencli-pr-checklist.md` (Claude Code only — same content as this file, with owner-commit SHAs for each pattern)
