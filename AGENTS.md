# AGENTS.md

Contributor guidance for AI coding tools (Cursor, Codex, Aider, etc.) and humans working in this fork.

**The complete checklist lives in [`CLAUDE.md`](./CLAUDE.md)** — read it before editing `clis/*` or `extension/*` or opening a PR.

The conventions were derived from observing how the upstream maintainer (`jackwener/OpenCLI`) amends PRs before merging. Following them upfront shortens review cycles.

Quick summary of what's in `CLAUDE.md`:

1. Use typed errors from `@jackwener/opencli/errors` — never `throw new Error()`
2. Validate `--limit` / `--page` fail-fast with `readPositiveInteger`, before `page.goto`
3. Prefer imperative `func:` over declarative `pipeline:`
4. Type-check after `JSON.parse` — `"null"` and `42` are valid JSON
5. Multi-source merge: merge first, filter last
6. Replace `Array.find` with scored sort when multiple matches are possible
7. Re-check socket / instance identity after every `await`
8. Test error paths, not just happy paths
9. Docs: name every lifecycle subtype when introducing limits
10. Rebuild `cli-manifest.json` after adapter arg changes
