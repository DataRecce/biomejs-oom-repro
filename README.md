# biome stack overflow on Turbopack-generated SSR chunk

Minimal reproducer for a Biome stack overflow that occurs when any of the
TypeScript-aware nursery rules (`noFloatingPromises`, `noMisusedPromises`,
`noUnnecessaryConditions`) is enabled and a Turbopack-generated SSR chunk
exists in the working tree.

Filed as a follow-up to biomejs/biome#10411 per @siketyan's request.

## Reproduce

```bash
npm install
npx biome check
```

## Expected

A clean run (the linter does not lint `.js` files by default and there are no
`.ts`/`.tsx` files in this repo).

## Actual

```
thread 'biome::workspace_worker_N' (NNNNNNN) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

## Trigger

A single Turbopack-generated chunk under
`.next/dev/server/chunks/ssr/05o5_yaml_dist_0y-df2-._.js` (~310 KB,
turbopack bundle of the `yaml` package, captured from a Next.js 16 + React 19
dev build).

The crash happens even though:

- the file is not a TypeScript file
- the file lives under `.next/`, which most users assume Biome ignores
- adding `"!**/.next"` to `linter.includes` in `biome.json` does **not**
  prevent the crash

## Version bracket

Tested on macOS `darwin-arm64`, Node 24:

| Biome version | Result |
| --- | --- |
| 2.4.11 | ok |
| 2.4.12 | **stack overflow** |
| 2.4.13 | stack overflow |
| 2.4.14 | stack overflow |
| 2.4.15 | stack overflow |
| 2.4.16 | stack overflow |

The regression was introduced in **2.4.12** for this trigger file.

## Files

- `package.json` — pins `@biomejs/biome@2.4.16`
- `biome.json` — enables one type-aware nursery rule (`noFloatingPromises`)
- `.next/dev/server/chunks/ssr/05o5_yaml_dist_0y-df2-._.js` — the trigger,
  unmodified Turbopack output

## Rule sensitivity

Any one of these nursery rules is enough to trigger the crash, independently:

- `noFloatingPromises`
- `noMisusedPromises`
- `noUnnecessaryConditions`

With none of them enabled, the run completes cleanly.

## Environment captured for the original report

- macOS, `darwin-arm64`
- `@biomejs/biome@2.4.16` (latest at time of filing)
- Node 24.16.0
- Original codebase: Next.js 16 + React 19 + Turbopack dev build

## Bisection notes

Truncating the 7426-line trigger file to its first or second half alone does
not crash; the crash requires a span across the middle of the file, so the
overflow is not produced by a single isolated construct. The file is the
unmodified Turbopack output for `yaml@2.8.3` and is included verbatim.
