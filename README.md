# Design by Contract (DbC) for ECMAScript

[![npm](https://img.shields.io/npm/v/design-contracts)](https://www.npmjs.com/package/design-contracts)
[![license](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)
[![stage](https://img.shields.io/badge/stage-0-lightgrey.svg)](https://github.com/bejiihiu/proposal-design-by-contract/issues)

**Stage: 0 — seeking a champion**

|                          |                                                        |
| ------------------------ | ------------------------------------------------------- |
| Stage                    | 0 (strawperson / exploration)                           |
| Author                   | bejiihiu                                                |
| Champion                 | **seeking** — see [How to engage](#how-to-engage)       |
| Reference implementation | [`design-contracts`](https://www.npmjs.com/package/design-contracts) on npm |
| Source                   | [github.com/bejiihiu/node-contracts](https://github.com/bejiihiu/node-contracts) |
| This repository          | [github.com/bejiihiu/proposal-design-by-contract](https://github.com/bejiihiu/proposal-design-by-contract) |
| License                  | MIT                                                     |

Like every Stage 0 document, this proposal is a starting point for discussion, not a commitment. Nothing here is normative yet; the API sketch is informative and will change — possibly radically — if the committee engages. Sections marked *informative* describe the current shape of the reference implementation, not a requirement.

---

## Abstract

Design by Contract (DbC) is a discipline — pioneered by Eiffel — in which a unit of code declares the promises that hold around its execution, and the runtime enforces them: **preconditions** (what the caller must ensure), **postconditions** (what the callee guarantees), and **invariants** (what must remain true throughout). This proposal explores adding first-class Design by Contract support to ECMAScript, initially as a library-shaped, spec-aligned API surface rather than new syntax. It does **not** propose a `require`/`ensure` keyword at this stage.

The reference implementation, [`design-contracts`](https://www.npmjs.com/package/design-contracts) (MIT, zero dependencies, dual ESM/CJS, TypeScript-first), demonstrates that the semantics are implementable today in plain ECMAScript with acceptable and honestly measured overhead, and that the hardest questions — failure behavior, inheritance semantics, old-value snapshots, async boundaries, compile-time elimination — have workable, tested answers.

## Motivation & use cases

### Why types are not enough

TypeScript and JSDoc types prove that a value has a *shape*. They cannot prove that:

- `withdraw(account, amount)` never returns a negative balance;
- `charge(card, amountCents)` is only ever called with a positive integral amount;
- a subclass override of `save()` does not silently weaken the guarantees callers rely on;
- a data structure's internal invariant ("no element appears twice") still holds after this method returns;
- a function that fails fails *in the documented way* ("throws `GatewayError` on decline").

These are behavioral promises. Types are erased or structural; promises are dynamic. A contract violation is a **runtime event** precisely where a wrong answer is worse than an exception — types can't see it, tests only see the scenarios they cover.

### Use cases

1. **Critical cores** — financial arithmetic, permission checks, state machines, ledger writes, protocol handling: anywhere a silent wrong answer is worse than a loud exception.
2. **Internal library guarantees** — a library can state and enforce its own pre/postconditions instead of trusting consumers to read the docs.
3. **Team boundaries inside a codebase** — consumers of a module should be able to trust it more than the type system alone allows.
4. **Gradual correctness rollout in production** — an `audit` mode records violations into telemetry without changing control flow, so contracts can be enabled on real traffic before being enforced.
5. **AI-generated and junior-written code** — types check out but behavior drifts; contracts ratchet behavior against the stated intent.

### Contrast with adjacent tools

|                       | TypeScript (types)            | Zod / schema validation        | Node `--experimental-*` flags | This proposal                              |
| --------------------- | ----------------------------- | ------------------------------ | ---------------------------- | ------------------------------------------ |
| Direction             | static, compile time          | edge → inside (untrusted input) | enable experimental features today | code making promises to **itself**          |
| When it runs          | never (erased)                | once, at the boundary          | —                            | every call, while enabled                  |
| Failure means         | "this doesn't type-check"     | "this input is unacceptable"   | —                            | "this part of the system broke its promise" |
| Cost model            | zero at runtime               | paid at the edge               | —                            | paid per call (see [Performance](#performance)) |

Zod answers "can I trust this value from outside?" Contracts answer "is this part of my code healthy?" You want both, at different layers. Contracts are **not** a validation library and are not positioned as one — see the [FAQ](#faq).

## Prior art

Design by Contract has a long, well-trodden history. This section is deliberately honest about both successes and failures.

### Eiffel (Meyer, 1986)

The origin. `require` / `ensure` / `invariant`, the `old` expression (pre-call value of an expression), **assertion levels** (`check`, `require`, `ensure`, `invariant`, controllable at runtime via `ISE_STATE`/`Eiffel assertion levels`), and **Liskov-compatible inheritance semantics built into the language**: a subclass may weaken preconditions (disjunction with the base) and must strengthen postconditions (conjunction/union), and class invariants of the base survive in subclasses. This proposal's semantics are explicitly Eiffel-derived.

### D

D has built-in contracts: `in` / `out` / `body` (historically), class invariants, and `assert`. `out` gives the function a name for the result so postconditions can read it. D demonstrates that contracts can live as first-class language members without a new keyword universe.

### Dart

Dart once shipped built-in contract support (`@contract`, pre/postconditions) in early SDKs and later **removed** it — a cautionary tale: contract features that are always-on, or whose semantics fork the type system, tend not to survive. This proposal learns from that: opt-in, zero-config-off, no type-system fork.

### .NET Code Contracts (Microsoft.Contracts)

`[ContractPrecondition]` / `Contract.Requires` / `Contract.Ensures`, with static analysis (the "cloudify"/abstract-interpretation checker) and runtime rewriting via IL instrumentation. It worked and was genuinely useful, and it was ultimately **deprecated/removed from the framework** — tooling burden, IL rewriting complexity, and mediocre diagnostics killed it. Lesson: a contract system that requires a special toolchain for basic operation will not reach mainstream adoption. This proposal keeps runtime semantics self-contained.

### Java: iContract, OVal, JML

- **iContract** — doc-comment style contracts; compiles via a bytecode rewriter.
- **OVal** — annotation-based (`@AssertTrue` etc.), reflection-heavy.
- **JML (Java Modeling Language)** — specification language with a checker (OpenJML); powerful, heavyweight, never mainstream.

### SPARK / Ada

Formal contracts with proof support — beyond the scope of ECMAScript, but the reference for "contracts as specification, not just runtime checks".

### Python: `icontract`

A strong, mature library (not language-integrated): `@icontract.require`, `@icontract.ensure`, `old`-style snapshots via `snapshot` param, inheritance semantics, SMT-backed static checking available. Evidence that a *library-shaped* DbC can thrive without new syntax.

### Rust: `contract` crate, `contracts` crate, `prusti`

The `contract`/`contracts` crates provide attribute macros (`#[requires]`, `#[ensures]`) — compile-time macro expansion, zero runtime cost when disabled. The ecosystem treats contracts as a library concern. This proposal is in the same family.

### C++: P0380R0 / P0542R0 (Contracts, failed)

The most important cautionary tale in this document. Contracts were proposed repeatedly; the papers accumulated semantic disputes (should contract violation be `[[noreturn]]`? is continued execution after a failed contract well-defined?), implementation complexity, and a genuinely unresolved question: **what does it cost when disabled?** The C++ debacle was not about whether contracts are useful — nobody disagreed — but about unchecked assumptions concerning performance, terminology, and what should happen at the exact moment of violation. WG21 ultimately failed to ship them (C++20) and reworked the design over years (P2388R*, Contracts in C++26).

This proposal addresses the C++ pitfalls explicitly, as first-class design constraints:

1. **No ambiguity about violation semantics.** A violation in `strict` mode throws a typed error (`ContractViolationError` subclasses); it does not `abort`, does not invoke UB, and does not fork the language. The C++ "what is the meaning of a failed contract" question has a boring, safe, typed answer here.
2. **Compile-time elimination is a real, tested path**, not an aspiration — see [Compile-time elimination](#compile-time-elimination) and [Performance](#performance). Modes make cost predictable at build time.
3. **No global behavior fork.** Contract checks are opt-in and local; a library that ships contracts does not change the meaning of code for its consumers.
4. **The violation is observable and telemetry-friendly** (`audit` mode), so "turn them on in production gradually" is a supported workflow, not a hope.

### TypeScript

`asserts` functions and assertion signatures (`asserts condition`) are the closest static cousin. They narrow types at a call site but check nothing at runtime and cannot express old-values, invariants, or Liskov rules. Complementary, not competitive.

### Node.js `diagnostics_channel`

A stable (node ≥ 15.1) pub/sub channel for diagnostics events. The reference implementation already publishes violations/audit records on `diagnostics_channel` (`design-contracts:violation` / `design-contracts:audit`) with zero subscription cost — evidence that contracts integrate with existing observability machinery without new engine support.

### Julia

Julia supports runtime contracts through libraries (`@contract`-style packages) and has a strong culture of checkable specifications. No language-integrated contracts; used here as evidence that the library path is viable across ecosystems.

### Relationship to V8 / TC39

This proposal is a **plain JavaScript normative change** (or, in Stage 0 form, library-shaped guidance + conventions). **No V8 internals are required**; everything in the reference implementation runs today in stock engines. The interesting future question — whether anything belongs *in* the engine (e.g. a `%Contract%` intrinsic, `Function.prototype.contract`, or syntax) — is explicitly deferred to later stages. See [Open questions](#open-questions).

## Proposed API sketch (informative, not normative)

Not normative. The names will change. What must survive is the *semantic core*: preconditions, postconditions, invariants, error/on-error conditions, `old` values, modes, Liskov rules. The sketch ships as the npm package [`design-contracts`](https://www.npmjs.com/package/design-contracts) v1.0.0 (MIT, zero dependencies, dual ESM/CJS, Node ≥ 18, Bun supported).

### The semantic core, in six bullets

- `contract(fn, options)` wraps `fn`; returns a function with the **same runtime shape and TypeScript signature**. No wrapper types leak into call sites.
- Options: `pre`, `post`, `onError`, `old`, `name`, `mode`, `captureStack`, `message`.
- Conditions are **values**, not strings or closures-only: `condition(predicate, { name, message })`, composable via `all` / `any` / `not`; `isNamedCondition` guards them.
- `assert(cond, message?, context?)` / `assert.fail` — free-standing assertions with the same enforcement machinery (no-op under `off`/`silent`/`disabled: ['assertion']`).
- `contractObject(target, { invariant })` — invariants on a `Proxy` wrapper: checked at creation, after every method settles, and after every direct property write.
- `configure({ mode, onViolation, onAudit, disabled, ... })`, scoped via `withConfig()` (AsyncLocalStorage), overridable per contract; resolution order **per-contract mode > scope > global > environment** (`NODE_CONTRACTS_MODE`, else `NODE_ENV === 'production' ? 'off' : 'strict'`).

### Code sketches (appendix)

```ts
import { contract, condition, contractObject, configure } from 'design-contracts';

const positiveAmount = condition((n: number) => Number.isInteger(n) && n > 0, {
  name: 'positive amount',
});

const charge = contract(
  async function charge(cardId: string, amountCents: number) {
    return gateway.charge(cardId, amountCents);
  },
  {
    name: 'charge',
    pre: [positiveAmount],
    post: [(receipt) => receipt.status === 'captured'],
    onError: [(error) => error instanceof GatewayError],
    old: ['amountCents'], // snapshot args; visible as context.old in post
  },
);
```

Named conditions and composition:

```ts
import { all, any, condition } from 'design-contracts';

const notBlank = condition((s: string) => s.trim().length > 0, { name: 'not blank' });
const noThrowawayDomain = condition((s: string) => !s.includes('mailinator'), { name: 'no throwaway domain' });
const signupEmail = all(notBlank, noThrowawayDomain); // composite reads 'all(not blank, no throwaway domain)'
```

Object invariants:

```ts
const balanceNonNegative = condition((self) => self.balance >= 0, {
  name: 'balance non-negative',
});
const account = contractObject(new BankAccount('Ada', 100), {
  name: 'ada account',
  invariant: [balanceNonNegative],
});
```

Stage 3 decorator surface (shipped in the reference implementation):

```ts
class Bank {
  @contractMethod({ pre: [(amount: number) => amount > 0] })
  deposit(amount: number) { /* ... */ return amount; }
}
```

Modes and telemetry (see [Performance](#performance) and [docs/benchmarks-summary.md](./docs/benchmarks-summary.md)):

```ts
configure({
  mode: 'audit',                                  // predicates run, violations → records, control flow unchanged
  onAudit: (record) => otelOrPinoLogger.emit(record),
  onViolation: (error) => alerting.capture(error),
});
```

More complete example gallery: [`examples/`](https://github.com/bejiihiu/node-contracts/tree/main/examples) in the reference repository.

### Errors (informative)

```
Error
└── ContractViolationError        kind: precondition | postcondition | invariant | assertion | on-error
    ├── PreconditionViolationError
    ├── PostconditionViolationError
    ├── InvariantViolationError
    ├── AssertionViolationError
    └── OnErrorViolationError
```

Each carries structured readonly fields (`kind`, `contractName?`, `functionName?`, `conditionIndex?`, `conditionName?`, `context?`, `mode`) and a log-safe `toJSON()`. `context` only carries what the caller explicitly passes, sanitized; the library never serializes raw arguments on its own.

### Modes (informative)

| Mode     | Predicates executed? | On violation                              | Use for                         |
| -------- | -------------------- | ----------------------------------------- | ------------------------------- |
| `strict` | yes                  | throws; `fn` not called on pre-failure    | development, CI, critical paths |
| `warn`   | yes                  | reports (`onViolation` / one-line warn), continues | gradual rollout, staging        |
| `silent` | **no**               | ignored                                   | zero-predicate-cost kill switch |
| `audit`  | yes                  | emits `AuditRecord`, continues            | production telemetry            |
| `off`    | **no**               | nothing — bare `fn.apply`                 | production with stripping       |

`silent`/`off` short-circuit **before any predicate runs** — not "evaluate and swallow". `disabled: ViolationCategory[]` unions across all layers and wins over per-contract `mode`. `warn` ≈ `strict` in cost; the difference is only the landing, not the price.

## Semantics details

### Liskov substitution and inheritance

The reference implementation takes Eiffel's rules literally:

- **Effective precondition = `ALL(base.pre) OR ALL(derived.pre)`** — disjunction of the two groups. A derived contract may relax, never strengthen, what base-typed callers rely on.
- **Effective postcondition = `base.post ∪ derived.post`** (base first).
- **Effective `onError` = `base.onError ∪ derived.onError`** (base first).
- **`old` / `mode` / `name` / `captureStack` / `message`** = `derived ?? base`.

Invariants from the base class are unioned into the derived object's invariant set (no weakening in subclasses). This is the part static type systems get subtly wrong in generics-heavy code; here it is one sentence and it is tested.

### Failure semantics (what happens at the exact moment of violation)

The most-litigated question in the prior art. Here:

- `strict` throws the violation error from the wrapper. Precondition failure → `fn` is not called. Postcondition failure → the call already happened; the error replaces the (derived) promise's resolution or throws on the sync path.
- `warn` calls `onViolation(error)` (exceptions in the handler are caught and logged; control flow is never changed by telemetry); execution continues.
- `audit` builds an `AuditRecord` (`kind`, `message`, `timestamp`, `mode: 'audit'`, `contractName?`, `functionName?`, `conditionIndex?`, `durationMs?`), hands it to `onAudit` and `onViolation` if set; execution continues. Without handlers the record is dropped — there is **no hidden global log**.
- `silent`/`off` — predicates are not executed at all.
- Predicate exceptions propagate unwrapped; they are userland code. A violation of `onError` semantics throws `OnErrorViolationError`; otherwise the original error propagates untouched.

### Invariants on objects

`contractObject(target, { invariant })` returns a `Proxy` typed as `T`. Invariants run: (a) at creation, (b) after every wrapped method settles (sync return, sync throw, promise fulfill **and** reject), and (c) after every direct property write (`<set>`). Writes *inside* a method are not re-checked at the moment of assignment — one check per method settle. Invariants therefore tolerate transiently inconsistent intermediate states inside a method body, which is the correct transactionality model for real code.

### `old` values — explicit snapshots

`old` is never an implicit string/closure capture. It is an **explicit snapshot request**: either an array of parameter names (via best-effort `fn.toString()` introspection — broken by minification, documented loudly) or a function `(args) => Record<string, unknown>` (exact, minification-proof). Capture strategy is documented per value shape: primitives by value; plain objects/arrays/Map/Set/Date via `structuredClone`; class instances containing functions and non-cloneable graphs fall back to live references flagged by `isReferenceCapture()`. `old` is only taken when configured **and** `post` is non-empty **and** postconditions are not skipped.

### Async functions

Postconditions see the **resolved** value, never a thenable. Async predicates are awaited on the async path; an async predicate in a synchronous call path throws `TypeError` rather than vacuously passing. Composites (`all`/`any`/`not`) await nested async predicates on the async path. `withConfig` scope survives `await` via AsyncLocalStorage. A rejected promise skips postconditions and routes to `onError`; a violation of `onError` throws `OnErrorViolationError` (strict) and otherwise the original rejection propagates untouched.

### `diagnostics_channel` integration

Violations and audit records are published on stable `node:diagnostics_channel` channels (`design-contracts:violation` / `design-contracts:audit`). Subscribing costs nothing when nobody listens; producers pay only the publish.

## Performance

Honest, measured, upper bounds. Source: `benchmarks/contract.bench.ts` in the reference repository, run with `vitest bench` (tinybench). Numbers below are **Node 24.2.0**; the full table (including Bun 1.4.2) is in [docs/benchmarks-summary.md](./docs/benchmarks-summary.md) and in the reference repository's [docs/benchmarks.md](https://github.com/bejiihiu/node-contracts/blob/main/docs/benchmarks.md).

| Scenario                            | ops/sec | vs plain      |
| ----------------------------------- | ------- | ------------- |
| plain call                          | 12.35M  | 1x            |
| contract, 1 precondition            | 565k    | ~22x slower   |
| contract, pre + post ("typical")    | 434k    | ~29x slower   |
| contract, `off` mode                | 1.22M   | ~10x slower   |
| contract, `warn` mode               | 387k    | ~32x slower   |
| async contract, pre + post          | 238k    | ~7.4x slower* |
| `contractObject` + invariant        | 255k    | ~25x slower   |
| contract, `pre + post + old`        | 101k    | ~122x slower  |

\* vs async baseline (1.77M ops/s).

**Read these numbers the honest way:**

1. They are measured through the vitest module-runner, which interposes proxy getters on module-export accesses. A compiled `dist` build resolves these once and is materially cheaper; treat these as pessimistic ceilings, not predictions.
2. `old` is the most expensive feature (a `structuredClone` per call) and dominated by design — don't put it on hot paths.
3. `warn` ≈ `strict`; the only difference is the dispatch on violation. `silent` ≈ `off` — predicates are not executed at all.
4. Synthetic trivial functions exaggerate the ratio; a contract on a 200 µs database call is unmeasurable.

### Compile-time elimination

`design-contracts` reads a global constant `__DESIGN_CONTRACTS_ENABLED__` through a `typeof` guard. When the bundler defines it `false`:

- **Behavior (always guaranteed):** predicates never execute; observable behavior does not depend on checks.
- **Physical removal (minifier-dependent):** physically deleting the machinery requires a minifier that inlines the flag check at the call site (e.g. terser's `compress`). esbuild `define` + treeshake folds the branch but leaves the (unreachable) machinery in the bundle.

Define recipes exist for esbuild, tsup, Vite, webpack (`DefinePlugin`), and Rollup (`@rollup/plugin-replace`); the exact wiring is documented in the reference repository. The compile-time switch is the supported production path for "zero cost", not a benchmark trick.

## Open questions

Stage 0 exists to ask these. None of them are settled:

1. **Syntax vs library.** The strongest path may land as *guidance + conventions + potential built-ins later* (e.g. a `%Contract%` intrinsic or `Function.prototype.contract`), not as new syntax. Is syntax ever justified, or is the library-plus-intrinsic shape the end state?
2. **What, if anything, goes in the engine.** Candidates discussed: an intrinsic for contract wrapping, `Function.prototype.contract(options)`, `diagnostics_channel`-adjacent standard events. Nothing is committed; all are premature before Stage 1.
3. **Assertion configuration model.** `strict`/`warn`/`silent`/`audit`/`off` + per-contract override + per-category disable works as a library. Should the platform bless an environment-variable convention (`NODE_CONTRACTS_MODE`-style) as the interoperable baseline, or stay library-defined?
4. **Naming.** `contract`/`pre`/`post`/`invariant` are Eiffel heritage, not ECMAScript style. Alternatives (`assertThat`, `requires`/`ensures`) were rejected as gratuitous divergence from 40 years of literature — but naming bikeshedding is expected and welcome in Stage 0.
5. **Interaction with `Promise` subclassing and custom thenables** in postcondition phases — edge cases remain.
6. **`assert` under `off` in production.** Current semantics: `assert` is a no-op under `off`/`silent`. Is that the right default, or should `assert` survive stripping (Eiffel keeps `check` semantics distinct)?
7. **Spec text vs implementation.** If this advances to Stage 1, the first deliverable is a spec for the *semantics* (modes, failure dispatch, Liskov rules, `old` semantics), with the reference implementation as a test oracle — not a rewrite.
8. **Decorators.** `@contractMethod` ships as a Stage 3-compatible decorator. Does it remain library surface, or does the committee see a path to standard integration?
9. **Error classes in the language.** Should violations be a distinct `ContractViolationError` brand in the realm (like `AggregateError`) or stay library classes?

If you think any of these have obvious answers, that is exactly what the Stage 0 discussion is for — see [How to engage](#how-to-engage).

## FAQ

**Why not just Zod?**
Zod validates untrusted input once at the boundary: "can I trust this value from outside?" Contracts verify your own code's promises on every call: "is this part of my code healthy?" A Zod schema keeps malformed HTTP bodies out; a contract on `CreditLedger.post` proves the ledger never double-spends. Different layers, both needed.

**Why not TypeScript?**
Types are static, erased, and structural. They cannot see runtime values, invariant drift inside an object, or Liskov violations across dynamic dispatch. `asserts` narrows a type at a call site; it checks nothing at runtime.

**Does it work in production?**
Yes — that is the point of `audit` mode (violations recorded, control flow unchanged) and `off`/silent (zero predicate cost) and the compile-time define (zero runtime cost). Ship with `audit`, watch telemetry, flip to `strict` on critical cores when the signal is clean.

**What does a contract cost on a hot path?**
Measured, pessimistically: single-digit nanoseconds to a few hundred nanoseconds per call depending on predicates; `old` dominates. Don't put contracts under million-iteration inner loops. Measure with your own workload.

**What happens on violation — does the process crash?**
In `strict`, the wrapper throws. Nothing crashes by default. In `warn`/`audit`, control flow is untouched and telemetry handlers are failure-isolated (their own exceptions cannot break your business logic).

**Is this compatible with bundlers?**
Yes. Dual ESM/CJS package, `sideEffects: false`, no import-time side effects, no hidden globals beyond what you configure, and an explicit define-based stripping path for release builds.

**Why should TC39 take this seriously?**
Because the alternative is every critical codebase re-learning Eiffel badly on its own: ad-hoc `assert` sprinkles, silent invariant decay, and "types compiled, production broke". One honest, spec-worthy semantics beats a hundred private ones. And because the reference implementation already exists, is MIT, is tested (500+ tests), and publishes honest benchmarks and failure semantics — the usual Stage 0 objection that "nobody knows what this means in practice" does not apply here.

## How to engage

This proposal is **Stage 0, seeking a champion**. To get it there:

1. **Read the reference material**: [proposal repo README](https://github.com/bejiihiu/proposal-design-by-contract), [reference implementation](https://github.com/bejiihiu/node-contracts), [npm package](https://www.npmjs.com/package/design-contracts), [docs & benchmarks](https://github.com/bejiihiu/node-contracts/tree/main/docs).
2. **Discuss**: open an issue in [this repository's issue tracker](https://github.com/bejiihiu/proposal-design-by-contract/issues) or on the reference repository. Critique is the entire point of Stage 0 — vague enthusiasm is less useful than a hostile review of the failure semantics.
3. **Committee visibility**: if you are a TC39 member or contribute to the ecosystem (bundlers, ORMs, runtimes, APM vendors), the next milestone is a post on [forum.tc39.es](https://forum.tc39.es) asking for a champion and initial feedback. That step is owned by the repository owner; watch this repository's issues for the announcement.
4. **Champion welcome**: ECMAScript needs a champion — a committee member willing to carry this through Stages 1–4. If you are that person, or you know that person, reach out via an issue here.

Concrete asks for reviewers, in order of value: (1) attack the failure semantics in [Semantics details](#semantics-details); (2) attack the performance claims in [Performance](#performance) — the benchmark file is public; (3) tell us which of the [open questions](#open-questions) blocks Stage 1 for you.

## License

[MIT](./LICENSE) © bejiihiu and contributors.

---

*Stage 0 means exploration, not commitment. This document will evolve with the discussion; material changes will be reflected here and in the issue tracker.*
