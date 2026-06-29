# Veritas Universal Rigor Guard — V15 Terminal

A deterministic, registry-based LLM response guard with multi-agent refinement, blind
third-party judge ensemble, and pluggable domain packs.

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  executeUniversalRigorPipeline()                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │ Auto-Fix     │ →  │ Flaw Scan    │ →  │ Critique → Editor    │  │
│  │ (per-pass)   │    │ + Score      │    │ (LLM, remediation    │  │
│  └──────────────┘    └──────────────┘    │ hints injected)      │  │
│       ↑                                  └──────────────────────┘  │
│       │ (loop up to maxDepth, early-exit on stable)                │
│       │                                                            │
│       ▼                                                            │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │ Final        │ →  │ Re-score &   │ →  │ Parallel Blind Judge │  │
│  │ Auto-Fix     │    │ Stable       │    │ (3 samples, optional │  │
│  │ (post-loop)  │    │ promotion    │    │ cross-model rotation)│  │
│  └──────────────┘    └──────────────┘    └──────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

## Registry Layout

| Pack                  | Detectors | Description                                |
|-----------------------|-----------|--------------------------------------------|
| `builtin`             | ~38       | Universal output-surface, formatting, citation, numeric, epistemic |
| `statistics`          | ~9        | Core statistical methodology (catalog #1-680)|
| `statistics-advanced` | ~6        | Frontier statistical methodology (#801-875)|
| `software-extended`   | ~40       | Node/TS/Python/Go/C++/Swift/Tailwind       |
| `software-rn-webgl`   | ~25       | React Native + WebGL/WebGPU                |
| `medical`             | ~13       | HIPAA PHI + clinical safety (NEW)          |
| `legal`               | ~11       | Legal advisory + citation integrity (NEW)  |
| `finance`             | ~13       | Investment/tax/securities (NEW)            |

**Total: ~155 detectors + 9 deterministic auto-fixers**

## File Index

```
src/lib/
  flaw-registry.ts             ← Core registry + shared toolkit
  universal-rigor-guard.ts     ← Pipeline orchestration
  flaws/
    builtins.ts                ← Universal detectors
    fixers.ts                  ← Deterministic auto-fixers
    statistics.ts              ← Statistical methodology
    statistics-advanced.ts     ← Statistical frontier
    software-extended.ts       ← Cross-stack software
    software-rn-webgl.ts       ← React Native + WebGL
    medical.ts                 ← Medical/HIPAA (NEW)
    legal.ts                   ← Legal/UPL (NEW)
    finance.ts                 ← Finance/SEC (NEW)
    _template.ts               ← Pack template
    index.ts                   ← Pack registration
    selftest.ts                ← Assertions harness
```

## API

```typescript
import { ensureFlawsLoaded, listPacks, setPackEnabled } from "./lib/flaws";
import { executeUniversalRigorPipeline } from "./lib/universal-rigor-guard";

ensureFlawsLoaded();          // Idempotent — call once at app start
listPacks();                  // [{ pack, total, enabled }, ...]
setPackEnabled("medical", false);  // Runtime disable

const result = await executeUniversalRigorPipeline({
  initialDraft, userQuery, baseParams,
  computeRecords, constraints,
  maxDepth: 3,
  judgeModels: [modelA, modelB, modelC],  // optional cross-model rotation
  onDebug: msg => console.log(msg),
});
```

## What Universal Layer Cannot Cover

Without external oracles or domain packs:

| Failure Class                | Required Oracle                    |
|------------------------------|------------------------------------|
| Type errors / concurrency    | Compiler/linter harness (tsc, rustc, go vet) |
| Memory safety bugs           | Sanitizer harness (ASAN/TSAN)      |
| Dataflow taint (SQLi, XSS)   | SAST (Semgrep, CodeQL)             |
| Runtime invariants           | Log/trace analyzer (OpenTelemetry) |
| Database query plans         | EXPLAIN analyzer                   |
| Known-vulnerable dependencies| CVE/SBOM scanner (Syft + OSV)      |
| License conflicts            | License compliance scanner         |
| Complex semantic NLI         | Dedicated NLI model                |
| Code plagiarism / hallucination | Code similarity engine          |
| Formal safety properties     | Bounded model checker (CBMC, KLEE) |
| Sandboxed execution effects  | Process/filesystem monitor         |
| Network egress validation    | Network traffic analyzer           |
| External fact verification   | Retrieval-grounding + claim extraction |
| Source-quote fabrication     | Span-match against retrieved docs  |

## Declarative Pack Loading

Domain packs can also be loaded from JSON without writing TypeScript:

```typescript
import { loadDeclarativePack } from "./lib/flaws";

loadDeclarativePack({
  id: "my-org-policy",
  version: "1.0.0",
  rules: [{
    id: "myorg.no-internal-codename",
    domain: "structural",
    severity: "major",
    code: "MYORG_INTERNAL_LEAK",
    message: "Internal codename leaked.",
    remediation: "Replace internal codename with public name.",
    appliesWhenAny: ["external", "public"],
    scanRegex: "\\b(Project Bluebird|Operation Phoenix)\\b",
    scanFlags: "i",
  }],
});
```
