# Benchmarks — summary

Source: [`docs/benchmarks.md`](https://github.com/bejiihiu/node-contracts/blob/main/docs/benchmarks.md) in the reference repository ([`node-contracts`](https://github.com/bejiihiu/node-contracts)). Methodology below is copied/condensed from that document; numbers are the repo's own measurements, reproduced here for reference.

## Methodology

- **Tooling**: vitest 5's built-in benchmark mode (tinybench underneath). Scenarios are trivial standalone functions, so the measured delta is the wrapper + predicate cost, not domain logic.
- **Environments**: Node 24.2.0 (`npx vitest bench`) and Bun 1.4.2 (`bun run bench`).
- **Dist build not benchmarked.** Numbers below are against sources loaded through the vitest module-runner, which adds cross-module getter overhead (proxy getters on every module-export access). Treat all absolute figures as **pessimistic upper bounds**; a compiled/bundled `dist` is materially cheaper. Relative orderings are meaningful.
- **Scenarios**: plain call · 1 precondition · 3 preconditions · pre + post ("typical") · pre + post + `old` snapshot · `off` · `silent` · `warn` · async baseline vs async contract · `contractObject` invariant vs plain method · throwing function with and without `onError`.

## Results — Node 24.2.0

| Scenario                                | ops/sec | vs plain     |
| --------------------------------------- | ------- | ------------ |
| plain call                              | 12.35M  | 1x           |
| contract, 1 pre                         | 565.4k  | ~22x slower  |
| contract, 3 pre                         | 388.3k  | ~32x slower  |
| contract, pre + post                    | 433.7k  | ~29x slower  |
| contract, pre + post + old (array form) | 101.3k  | ~122x slower |
| contract, `off`                         | 1.217M  | ~10x slower  |
| contract, `warn`                        | 387.2k  | ~32x slower  |
| contract, `silent`                      | 1.336M  | ~9x slower   |
| async plain                             | 1.77M   | 1x           |
| async contract, pre + post              | 238.0k  | ~7.4x slower |
| plain method call                       | 6.38M   | 1x           |
| contractObject + invariant              | 254.7k  | ~25x slower  |
| plain throw                             | 29.4k   | 1x           |
| contract + onError (throwing fn)        | 16.0k   | ~1.8x slower |

## Results — Bun 1.4.2

| Scenario                                | ops/sec | vs plain     |
| --------------------------------------- | ------- | ------------ |
| plain call                              | 11.90M  | 1x           |
| contract, 1 pre                         | 497.5k  | ~24x slower  |
| contract, 3 pre                         | 352.2k  | ~34x slower  |
| contract, pre + post                    | 403.7k  | ~30x slower  |
| contract, pre + post + old (array form) | 100.2k  | ~119x slower |
| contract, `off`                         | 1.039M  | ~11x slower  |
| contract, `warn`                        | 336.3k  | ~35x slower  |
| contract, `silent`                      | 1.054M  | ~11x slower  |
| async plain                             | 1.80M   | 1x           |
| async contract, pre + post              | 218.7k  | ~8.2x slower |
| plain method call                       | 6.42M   | 1x           |
| contractObject + invariant              | 233.3k  | ~27x slower  |
| plain throw                             | 26.6k   | 1x           |
| contract + onError (throwing fn)        | 15.5k   | ~1.7x slower |

## Reading these numbers honestly

1. **Upper bounds, not predictions.** The vitest module-runner inflates cross-module costs; a `dist`/bundled build will be cheaper in absolute terms.
2. **Relative orderings are meaningful.** `off` ≈ `silent` < `warn` ≈ `strict`-like typical; `old` dominates everything (a `structuredClone` per call); async contracts cost proportionally less because promise machinery dwarfs the checks.
3. **`off` is not free.** A "≤ 5x overhead" goal for runtime `off` was **not** reached — ~10–11x remains under the module-runner (one config read, one flag read, a few `Set.has` before the bare call). Physical removal requires the compile-time define plus a minifier that inlines the flag check at the call site (e.g. terser); esbuild `define` + treeshake folds the branch but leaves the machinery unreachable-but-present.
4. **`warn` ≈ `strict`.** Same predicate cost; the difference is only what happens on violation.
5. **`onError` is free on the happy path.** Within noise of having none.
6. **Synthetic functions.** Real contracts on real functions are dominated by the function's own work; ratios on trivial functions are worst-case by construction.

Practical guidance: wrap domain/business logic, not loops; use `audit` in production where you want telemetry (same price as `strict` — spend it where you get the data); prefer `silent`/`off` or the compile-time define for release builds; avoid `old` on hot paths.
