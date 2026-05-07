# OMO Permission Redesign

| Field | Value |
|---|---|
| **Date** | 2026-05-05 |
| **Status** | Approved (pending user review of this spec) |
| **Scope** | `oh-my-opencode-slim` agent ecosystem — tools, MCPs, skills, task delegation |
| **Predecessors** | omo-slim-superpowers-patch-kit v1.3.0 (current) |
| **Successor target** | omo-slim-superpowers-patch-kit v1.4.0 |

---

## 1. Overview

Redesign per-agent permission allocation across the 31-agent OMO ecosystem to:

1. Resolve current asymmetries between built-in agents and best-of-N variants
2. Tighten "read-only" semantics by adding `write` and `todowrite` denies (item 6)
3. Replace ad-hoc per-agent third-party MCP whitelisting with a **closed-set blacklist** for known restricted MCPs — future-proof against new MCPs landing in the OpenCode ecosystem
4. Lock down `wildcard` and other utility agents from loading wide skill sets via `non-SP * deny`
5. Reserve a permission slot for the upcoming `update-memory` skill (orchestrator-only) — skill being built in another session
6. Sweep up misplaced skills (`simplify`, `verification-before-completion`, `receiving-code-review`) from agents that don't structurally need them

---

## 2. Background

### 2.1 Current state (pre-redesign)

The OMO permission stack today resolves to a **3-layer overlay** at runtime:

```
+----------------------------------------------------------------+
| Layer 1 | OMO patch code (this kit)                            |
|         | - skills allow/deny + suffix-strip inheritance       |
|         | - manages 3 MCPs only: websearch, context7, grep_app |
|         | - custom-agent registration + applyDefaultPermissions|
+----------------------------------------------------------------+
| Layer 2 | ~/.config/opencode/agents/*.md                       |
|         | - real self-prompt                                   |
|         | - explicit tool denies in frontmatter                |
|         | - shallow-merge wins over plugin-generated fields    |
+----------------------------------------------------------------+
| Layer 3 | oh-my-opencode-slim.jsonc / opencode.json            |
|         | - models, mcps allowlist, skills:[] override         |
|         | - global denies (windows-mcp_PowerShell/FileSystem)  |
+----------------------------------------------------------------+
```

### 2.2 Issues found

A two-pass codebase audit (`@explorer` rounds 1 + 2 with patch layer included) surfaced:

1. **Asymmetric inheritance**: variants inherit `skills` (suffix-strip), but NOT `mcps`/`tools`/`prompt`. Variant MCPs must be hand-copied in `oh-my-opencode-slim.jsonc`.
2. **Read-only semantic gap**: agents marked "read-only" in prose deny only `edit/bash/task` — `write` and `todowrite` slip through unenforced.
3. **`wildcard` non-SP = `* allow`**: a temperature-1.0 ideation agent can load any non-Superpowers skill, including mutation skills (`simplify`, `agent-browser`, etc.).
4. **Built-in `oracle` accidentally has broad tools**: `oracle.ts` self-prompt declares `READ-ONLY: You advise, you don't implement` (line 21-24), and `orchestrator.ts:46-53` describes `@oracle` as `Permissions: Read files`, but the code never adds `edit/bash/write` denies — so `@oracle` can in practice mutate state.
5. **Misplaced skills**:
   - `simplify` (custom OMO skill, says "Make the change + Run tests"): currently allowed on `oracle`, who structurally cannot land changes under the new design.
   - `verification-before-completion` (VBC): currently allowed on `oracle*`, but VBC's contract is "evidence before claiming completion" — `oracle` reviews, doesn't claim work-completion.
   - `receiving-code-review` (RCR): currently allowed on `oracle*`, but RCR is for the agent *receiving* review feedback (i.e. the implementer). Oracle *gives* reviews.
6. **MCP whitelisting friction**: every operator-class agent (fixer*/orchestrator*/laborer/designer*/librarian*) needs each restricted third-party MCP hand-listed in jsonc. Adding a new MCP later requires touching every agent.

### 2.3 Brainstorming fan-out outcome

Three independent specialist lenses (`@oracle-alpha` architecture, `@librarian-alpha` prior art, `@explorer-alpha` codebase impact) plus `@wildcard` contrarian fan-out converged on:

- Built-in `oracle` should be locked down (deny `edit/write/bash/task/todowrite`)
- `oracle-alpha/beta/gamma/delta` should remain locked down (already are, except for `write/todowrite` slip-through)
- `librarian-alpha` cited Anthropic's canonical `code-reviewer` archetype: `tools: Read, Grep, Glob, Bash` — but for **single-call reviewers**, not best-of-N rubric judges. Anthropic's own multi-agent research post documents the rubric judge as pure-text.

User extended this convergence to **Approach D**: full symmetry — even built-in `oracle` loses `bash`, on the grounds that:
- mental load of "which oracle has bash, which doesn't?"
- tournament fairness requires N judges see equivalent input — independent bash from one oracle violates this
- prompt-vs-code consistency (orchestrator + oracle both already say read-only)
- orchestrator can pre-fetch `gh`/`git` data and feed it in (best-of-N skill already does this)

---

## 3. Design Principles

### 3.1 Capability classes (3 tiers)

```
+============================================================+
| Tier 1: Coordinator                                        |
|   orchestrator, orchestrator-beta                          |
|   - Full tool access (incl. task delegation)               |
|   - Wide MCP set                                           |
|   - All Superpowers + reserved orchestrator-only skills    |
+============================================================+
| Tier 2: Implementer (write+edit+bash, no delegation)       |
|   fixer / fixer-{alpha,beta,gamma,delta}                   |
|   designer / designer-{alpha,beta,gamma,delta}             |
|   laborer                                                  |
|   - Tools: read, write, edit, bash, todowrite              |
|   - Tool deny: task (no further delegation)                |
|   - MCPs: windows-mcp + chrome-devtools + playwright OK    |
+============================================================+
| Tier 3: Read-only analyzer / researcher / utility          |
|   oracle / oracle-{alpha,beta,gamma,delta}                 |
|   explorer / explorer-{alpha,beta}                         |
|   librarian / librarian-{alpha,beta}                       |
|   observer, council, councillor                            |
|   scout, validator, gist, wildcard                         |
|   - Tool deny: edit, write, bash, task, todowrite          |
|   - MCPs: closed-set blacklist removes restricted ones     |
|   - Skills: per-agent narrow allowlist; non-SP * deny      |
+============================================================+
```

### 3.2 MCP allocation: hybrid whitelist + closed-set blacklist

```
+============================================================+
| OMO 3 MCPs (websearch / context7 / grep_app)               |
|   Mechanism: whitelist via jsonc `mcps` field              |
|   Why: introduced by OMO, OMO patch handles permission gen |
|   Future: same — patch-managed                             |
+============================================================+
| Restricted 3 MCPs (windows-mcp / chrome-devtools / playwright) |
|   Mechanism: closed-set blacklist in patch                 |
|   Patch knows operator agent set per MCP, emits deny rules |
|   for non-operators                                        |
|   Future-safe: new restricted MCP = single line in patch   |
+============================================================+
| Other third-party MCPs (gemini-tools, gmail, future)       |
|   Mechanism: untouched                                     |
|   Default: implicit allow for all agents                   |
|   Future: stays untouched unless we add to blacklist       |
+============================================================+
```

**Operator sets (closed-set):**

```ts
windows-mcp     -> fixer*, orchestrator*, laborer, designer*
chrome-devtools -> fixer*, orchestrator*, laborer, designer*, librarian*
playwright      -> fixer*, orchestrator*, laborer, designer*, librarian*
```

### 3.3 Symmetric oracle family

`oracle` + all `oracle-*` variants share identical capabilities:

- Tool deny: `edit, write, bash, task, todowrite`
- MCP: all restricted (per blacklist) + no OMO 3
- SP allow: only `systematic-debugging` (DBG)
- non-SP: `* deny`

Trade-off accepted:

- Pro: zero mental load, prompt-vs-code consistent, tournament-fairness preserved, no edge case "is this oracle the bash one or the no-bash one?"
- Con: orchestrator must pre-fetch `gh`/`git` data when oracle needs it (~5 sec extra per such turn)

### 3.4 Read-only agents: `write` + `todowrite` MUST be denied

Item 6 from user. Agents prose-marked as read-only currently leak `write` and `todowrite`. Spec:

```
Read-only agents add to their tool deny set:
  - edit, write, bash, task, todowrite
```

Affected: `oracle*`, `explorer*`, `librarian*`, `observer`, `council`, `councillor`, `scout`, `validator`, `gist`, `wildcard`.

### 3.5 Reserved orchestrator-only skills (NEW concept)

Some skills must be exclusively callable by `orchestrator` and `orchestrator-beta`. Examples:

- `best-of-n-with-judge` (item 2 from user — only orchestrator orchestrates tournaments)
- `update-memory` (placeholder — skill being built in another session, will manage OpenCode memory layer)

Mechanism: a new closed-set whitelist in the patch:

```ts
const RESERVED_ORCHESTRATOR_ONLY_SKILLS: ReadonlyArray<string> = [
  'best-of-n-with-judge',
  'update-memory',  // PLACEHOLDER — skill will land in a future commit
];
```

For each skill in this list, the patch emits:
- `allow` for `orchestrator` + `orchestrator-beta`
- `deny` for ALL other agents

This prevents the `wildcard non-SP = * allow` class of bug from ever recurring on these specific skills, regardless of whether the agent's `non-SP` is `* allow` or `* deny`.

### 3.6 Misplaced skill cleanup

Three skills move:

| Skill | Was on | Move to | Why |
|---|---|---|---|
| `simplify` | `oracle*` | `fixer` only | "Make the change + Run tests" — oracle structurally can't under the new design |
| `verification-before-completion` (VBC) | `oracle*` (via SP allow) | `orchestrator*`, `fixer*`, `designer*`, `validator` (existing) | VBC = "evidence before claiming completion"; oracle reviews, doesn't claim |
| `receiving-code-review` (RCR) | `oracle*` (via SP allow) | `fixer*`, `designer*` | RCR = receiver of review feedback; oracle gives it, doesn't receive |

---

## 4. Per-Agent Matrix

### Notation

```
ws | c7 | ga | win | crd | plw

OMO 3 (whitelist via jsonc):
  X = allow   . = deny
Restricted 3 (closed-set blacklist via patch):
  D = patch emits deny rule
  - = patch emits nothing → OpenCode default (allow)

Other MCPs (gemini-tools, gmail, future) = untouched, not shown

SP allow: explicit skills. Others deny.
non-SP: * allow / * deny posture for all non-Superpowers skills.
custom: explicit grants on OMO custom/recommended skills.
reserved: from RESERVED_ORCHESTRATOR_ONLY_SKILLS — auto-allowed for orchestrator*.
```

### 4.1 Primary (orchestrator class)

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom | reserved |
|---|---|---|---|---|---|---|
| `orchestrator` | none | X . X / - - - | 13/14 (all but `using-superpowers`) | * allow | `codemap` | **`best-of-n-with-judge`, `update-memory`** |
| `orchestrator-beta` | none | X . X / - - - | 13/14 | * allow | `codemap` | **`best-of-n-with-judge`, `update-memory`** |

### 4.2 Implementer subagents

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom |
|---|---|---|---|---|---|
| `fixer` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | — |
| `designer` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | `agent-browser` |
| `laborer` | `task` | . . . / - - - | (none, `skills:[]`) | * deny | — |

### 4.3 Reviewer subagents (Approach D — symmetric, fully read-only)

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom |
|---|---|---|---|---|---|
| `oracle` | `edit, write, bash, task, todowrite` | . . . / D D D | DBG only | * deny | **— (simplify removed)** |
| `council` | `edit, write, bash, todowrite` | . . . / D D D | (none) | * deny | — |
| `councillor` | `*` deny + only allow `read/glob/grep/lsp/list/codesearch` | . . . / D D D | — | * deny | — |

Note: `council` retains `task` allow because it dispatches `councillor` subagents.

### 4.4 Recon subagents (read-only)

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom |
|---|---|---|---|---|---|
| `explorer` | `edit, write, bash, task, todowrite` | . . . / D D D | (none) | * deny | — |
| `librarian` | `edit, write, bash, task, todowrite` | X X X / D - - | (none) | * deny | — |
| `observer` | `edit, write, bash, task, todowrite` | . . . / D D D | (none) | * deny | — |

### 4.5 Best-of-N — fixer family

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom |
|---|---|---|---|---|---|
| `fixer-alpha` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | — |
| `fixer-beta` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | — |
| `fixer-gamma` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | — |
| `fixer-delta` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | — |

### 4.6 Best-of-N — oracle family

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom |
|---|---|---|---|---|---|
| `oracle-alpha` | `edit, write, bash, task, todowrite` | . . . / D D D | DBG only | * deny | — |
| `oracle-beta` | `edit, write, bash, task, todowrite` | . . . / D D D | DBG only | * deny | — |
| `oracle-gamma` | `edit, write, bash, task, todowrite` | . . . / D D D | DBG only | * deny | — |
| `oracle-delta` | `edit, write, bash, task, todowrite` | . . . / D D D | DBG only | * deny | — |

### 4.7 Best-of-N — designer family

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom |
|---|---|---|---|---|---|
| `designer-alpha` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | `agent-browser` |
| `designer-beta` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | `agent-browser` |
| `designer-gamma` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | `agent-browser` |
| `designer-delta` | `task` | . . . / - - - | TDD, DBG, VBC, **RCR** | * allow | `agent-browser` |

### 4.8 Best-of-N — explorer family

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom |
|---|---|---|---|---|---|
| `explorer-alpha` | `edit, write, bash, task, todowrite` | . . . / D D D | (none) | * deny | — |
| `explorer-beta` | `edit, write, bash, task, todowrite` | . . . / D D D | (none) | * deny | — |

### 4.9 Best-of-N — librarian family

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom |
|---|---|---|---|---|---|
| `librarian-alpha` | `edit, write, bash, task, todowrite` | X X X / D - - | (none) | * deny | — |
| `librarian-beta` | `edit, write, bash, task, todowrite` | X X X / D - - | (none) | * deny | — |

### 4.10 Fast-lane / utility

| agent | tool deny | ws c7 ga / win crd plw | SP allow | non-SP | custom |
|---|---|---|---|---|---|
| `scout` | `edit, write, bash, task, todowrite` | . . . / D D D | (none) | * deny | — |
| `validator` | `edit, write, bash, task, todowrite` | . . . / D D D | VBC only | * deny | — |
| `gist` | `edit, write, bash, task, todowrite` | . . . / D D D | (none) | * deny | — |
| `wildcard` | `edit, write, bash, task, todowrite` | . . . / D D D | (none) | * deny | — |

---

## 5. Implementation Plan

### 5.1 New code

#### `src/config/agent-mcp-blacklist.ts` (NEW)

Closed-set blacklist for restricted third-party MCPs.

```ts
import { resolveBaseAgentName } from '../cli/superpowers-policy';

const MCP_OPERATOR_BASES: Record<string, ReadonlyArray<string>> = {
  'windows-mcp':     ['fixer', 'orchestrator', 'laborer', 'designer'],
  'chrome-devtools': ['fixer', 'orchestrator', 'laborer', 'designer', 'librarian'],
  'playwright':      ['fixer', 'orchestrator', 'laborer', 'designer', 'librarian'],
};

/**
 * For a given agent, return the list of restricted MCP names that should be
 * denied. Returns [] if the agent is in the operator set for all restricted MCPs.
 *
 * Future-safe: adding a new restricted MCP requires only one new entry in
 * MCP_OPERATOR_BASES; no per-agent jsonc/markdown changes needed.
 */
export function getRestrictedMcpDenies(agentName: string): string[] {
  const base = resolveBaseAgentName(agentName);
  const denies: string[] = [];
  for (const [mcp, allowedBases] of Object.entries(MCP_OPERATOR_BASES)) {
    if (!allowedBases.includes(base)) {
      denies.push(mcp);
    }
  }
  return denies;
}
```

#### `src/config/orchestrator-only-skills.ts` (NEW)

Closed-set whitelist for orchestrator-exclusive skills.

```ts
/**
 * Skills that may only be invoked by orchestrator / orchestrator-beta.
 * For all other agents, the patch emits an explicit `deny` rule for these
 * skill names — regardless of the agent's non-SP * allow / * deny posture.
 *
 * Future skills that should be orchestrator-only just append here.
 */
export const RESERVED_ORCHESTRATOR_ONLY_SKILLS: ReadonlyArray<string> = [
  'best-of-n-with-judge',
  'update-memory',  // PLACEHOLDER: skill landing in a future commit (memory layer)
];

/**
 * Returns true if the given agent is allowed to invoke reserved orchestrator-
 * only skills. Currently: orchestrator + orchestrator-beta only.
 */
export function isReservedSkillAllowed(agentName: string): boolean {
  return agentName === 'orchestrator' || agentName === 'orchestrator-beta';
}
```

### 5.2 Modified code

#### `src/cli/superpowers-policy.ts`

Update per-base SP allowlists:

```ts
// Was: { TDD, DBG, VBC }
// Now: { TDD, DBG, VBC, RCR }
fixer: ['test-driven-development', 'systematic-debugging',
         'verification-before-completion', 'receiving-code-review'],

// Was: { TDD, DBG, VBC }
// Now: { TDD, DBG, VBC, RCR }
designer: ['test-driven-development', 'systematic-debugging',
            'verification-before-completion', 'receiving-code-review'],

// Was: { DBG, VBC, RCR }
// Now: { DBG }
oracle: ['systematic-debugging'],
```

#### `src/cli/custom-skills.ts`

Update `simplify` allowlist:

```ts
// Was: ['fixer', 'oracle']
// Now: ['fixer'] only
simplify: ['fixer'],
```

#### `src/index.ts` (MCP permission emission site)

Where the patch currently emits OMO 3 MCP permission rules (around line 586-612), also call `getRestrictedMcpDenies(agentName)` and emit `<mcp>_*: deny` rules for each result.

Where the patch builds skill permissions (via `getSkillPermissionsForAgent`), apply `RESERVED_ORCHESTRATOR_ONLY_SKILLS` after the existing logic:

- For each skill in the reserved list:
  - If `isReservedSkillAllowed(agentName)` → set to `allow`
  - Else → set to `deny`
- This overrides any conflicting `* allow` from non-SP posture.

#### `src/cli/skills.ts`

For agents with `non-SP: * deny` (Tier 3 read-only agents), ensure the resolution logic in `getSkillPermissionsForAgent` preserves `* deny` when the agent has no explicit `skills:` override but the SP allowlist is non-empty. This may already work; add a unit test to lock the contract.

### 5.3 Markdown agent files

Affected files under `~/.config/opencode/agents/`:

#### Add `write, todowrite` to existing tool denies

Files (all of them already deny `edit, bash, task` — just append):

```
oracle.md, oracle-alpha.md, oracle-beta.md, oracle-gamma.md, oracle-delta.md
explorer.md, explorer-alpha.md, explorer-beta.md
librarian.md, librarian-alpha.md, librarian-beta.md
observer.md
council.md
scout.md
validator.md
gist.md
wildcard.md
```

#### Create new markdown files (if missing)

```
fixer.md           — frontmatter: deny task only
designer.md        — frontmatter: deny task only
laborer.md         — frontmatter: deny task only
council.md         — frontmatter: deny edit/write/bash/todowrite (keep task allow)
```

(Variants for fixer/designer already have markdown files.)

#### Wildcard + utility tightening

`wildcard.md` and similar must also reflect the `non-SP * deny` posture in their description prose, even though the actual gate is in `superpowers-policy.ts`.

### 5.4 `oh-my-opencode-slim.jsonc`

#### Remove redundant manual MCP whitelisting (windows/chrome/playwright)

Once the closed-set blacklist is live, operator-class agents no longer need to list these MCPs explicitly — the patch will leave them untouched (which means OpenCode default = allow). Cleanup:

- `librarian-alpha`, `librarian-beta`: remove `chrome-devtools`, `playwright` if currently listed (keep OMO 3: `websearch`, `context7`, `grep_app`).
- Anywhere `windows-mcp`, `chrome-devtools`, or `playwright` is currently listed in jsonc `mcps`: remove. The patch handles them now.

#### `wildcard` cleanup

Currently has no explicit MCP grant in jsonc, but verify after patch lands that the patch's blacklist denies still apply.

### 5.5 Documentation

- `opencode-config/README.md` lines ~109-114: update the "non-superpowers skills are preserved" section. Note that:
  - `best-of-n-with-judge` and `update-memory` (and any future `RESERVED_ORCHESTRATOR_ONLY_SKILLS` entries) are exceptions: orchestrator-only.
  - Tier 3 agents have `non-SP * deny`, so they cannot load arbitrary skills regardless.

- `docs/architecture.md`: add a new section "Permission Model" pointing to this spec doc.

- `docs/verify.md`: update the verification checklist for the new permission posture (e.g. "verify @oracle cannot invoke `bash`", "verify @wildcard cannot load `simplify`").

---

## 6. Testing Plan

### 6.1 Unit tests

#### `src/agents/index.test.ts`

Add tests:

```ts
test('oracle has DBG only in SP allowlist', ...);
test('oracle has no custom skills (simplify removed)', ...);
test('oracle non-SP defaults to deny', ...);
test('fixer has RCR in SP allowlist', ...);
test('designer has RCR in SP allowlist', ...);
test('wildcard non-SP is deny', ...);
```

#### `src/config/agent-mcp-blacklist.test.ts` (NEW)

```ts
test('oracle gets denies for windows-mcp, chrome-devtools, playwright', ...);
test('fixer-alpha (operator family) gets no denies', ...);
test('librarian-alpha gets denies for windows-mcp only (operator for crd/plw)', ...);
test('orchestrator-beta (suffix variant of operator) gets no denies', ...);
test('unknown agent names default to all denies (safe-by-default)', ...);
```

#### `src/config/orchestrator-only-skills.test.ts` (NEW)

```ts
test('orchestrator can invoke best-of-n-with-judge', ...);
test('orchestrator can invoke update-memory (placeholder skill)', ...);
test('fixer cannot invoke best-of-n-with-judge', ...);
test('fixer cannot invoke update-memory', ...);
test('wildcard cannot invoke either', ...);
test('orchestrator-beta inherits orchestrator privileges', ...);
```

### 6.2 Integration / live verification

After merging:

1. Restart OpenCode.
2. Verify `[plugin] health check passed` shows expected `agents` count (31 + shadow agents).
3. Open the agent picker; verify `oracle*` family no longer shows `bash` access.
4. Live probe: have orchestrator try to call `task(@oracle, "bash gh issue 123")` — verify oracle reports back "I don't have bash; please paste the issue body".
5. Live probe: have orchestrator delegate to `wildcard` and ask wildcard to load `simplify` skill — verify denial.
6. Verify `@orchestrator-beta` can use `update-memory` (once skill exists in another session).

### 6.3 Regression checks

- `bun test src/cli/skills.test.ts` should still pass.
- `bun test src/agents/index.test.ts` should still pass with new assertions.
- `bunx tsc --noEmit` clean.
- `bun run build` clean.

---

## 7. Considered Alternatives

### Approach A — Status Quo Plus
Just patch items 6-9 (read-only consistency, MCP cleanup, etc.); leave oracle untouched.
**Rejected**: doesn't fix the prompt-vs-code inconsistency on built-in oracle, and best-of-N vs built-in capability gap remains.

### Approach B — Pure lockdown (B-lite)
All oracles deny `edit, write, bash, task, todowrite`. **Same final destination as Approach D**, but earlier framing didn't yet include the closed-set MCP blacklist or the skill cleanup. Subsumed by Approach D.

### Approach C — Tiered Oracle (asymmetric bash)
Built-in oracle keeps `bash`; oracle-* don't. Maps to Anthropic's two canonical archetypes (`code-reviewer` with bash, `rubric-judge` pure-text).
**Rejected by user**: mental load of "which oracle has bash" too high; tournament fairness risk if same family has asymmetric capabilities; prompt-slip risk because variants share `oracle-base.md`.

### Wildcard P1 — Context-attached permissions
Permissions belong to the call site, not the agent. E.g. `@oracle` invoked freeform gets one set; `@oracle` in a tournament gets another.
**Rejected**: OpenCode permission model doesn't support call-site-attached perms today. Would require upstream changes.

### Wildcard P2 — Drop best-of-N variants entirely
Just spawn base oracle 4 times in parallel.
**Considered, deferred**: the variant infrastructure is already in place and best-of-N is opt-in. Dropping it now would invalidate the oracle/fixer/designer alpha/beta/gamma/delta investment. Revisit if best-of-N usage stays low after a quarter.

### Wildcard P3 — Linux/Postgres-style role inheritance
Define base capability roles (`reader`, `executor`, `judge`); agents inherit roles; permissions compose.
**Considered, deferred**: bigger refactor than this redesign's scope. Worth doing as a follow-up once Approach D ships and we have empirical data on which inheritance failures actually hurt.

### Wildcard P4 — Concentrate dangerous powers in oracle
**Rejected**: contradicts the entire reviewer/implementer separation principle.

---

## 8. Future Work

1. **RCR audit on other agents**: confirm `fixer*` and `designer*` are the right places for `receiving-code-review`. Check if any other agent should also receive review feedback.
2. **`update-memory` skill landing**: once the skill is implemented in the parallel session, verify it's correctly orchestrator-only via the `RESERVED_ORCHESTRATOR_ONLY_SKILLS` mechanism. No code change needed to allow it — just removing the `// PLACEHOLDER` comment.
3. **Role inheritance refactor (wildcard P3)**: if maintaining N similar agents continues to drift, consider extracting a `roles` layer where agent permissions compose from named roles (`reader`, `implementer`, `coordinator`). This spec deliberately stops short of that to keep the change atomic.
4. **Remove orchestrator's `bash` for production**: out of scope here, but worth considering in a future spec — orchestrator currently can `bash` directly, which means even Tier 3 read-only enforcement can be bypassed by a malicious orchestrator prompt. Defense-in-depth: orchestrator could in principle be denied `bash` and forced to delegate to fixer/laborer for shell work.
5. **Drop `best-of-N` variants if usage stays low** (wildcard P2 revisit, ~Q3 2026).

---

## 9. Open Items

None at the time this spec is approved.

The `update-memory` skill is not yet implemented; this spec reserves its permission slot. Once the skill lands, the `// PLACEHOLDER` comment in `orchestrator-only-skills.ts` can be removed; no other change is needed to authorize orchestrator to use it.

---

## 10. Approval

| Role | Status |
|---|---|
| User (Overseer) | ✓ Approved Approach D + RCR sweep + closed-set MCP blacklist + reserved-skill placeholder |
| Spec drafter (this doc) | ✓ Self-reviewed for placeholders, internal consistency, scope, ambiguity |
| Implementation gate | Awaiting user review of THIS document before invoking `writing-plans` skill |

---

## Appendix A — Brainstorming Fan-out Source Material

Phase 4 ideation fan-out specialists and their key contributions:

- **`@oracle-alpha`** (architecture lens): identified the "tiered oracle" option (Approach C) and called out that asymmetry within the same agent family is a smell. Recommended Approach D's symmetric direction in the second iteration.
- **`@librarian-alpha`** (prior art lens): cited Anthropic's canonical `code-reviewer` (with bash) vs `rubric-judge` (pure-text) archetypes. Sourced from Claude Code subagents docs and the Anthropic multi-agent research blog post.
- **`@explorer-alpha`** (codebase lens): proved the orchestrator and oracle prompts both already declare oracle as read-only — the broad tool access in code is accidental/legacy. Found the `simplify` skill conflict and the `verification-before-completion` mismatch.
- **`@wildcard`** (contrarian): surfaced four contrarian takes (P1-P4); P1/P4 rejected, P2/P3 deferred to future work.

## Appendix B — File Touch Manifest

```
NEW FILES:
  patches/oh-my-opencode-slim/src/config/agent-mcp-blacklist.ts
  patches/oh-my-opencode-slim/src/config/orchestrator-only-skills.ts
  patches/oh-my-opencode-slim/src/config/agent-mcp-blacklist.test.ts
  patches/oh-my-opencode-slim/src/config/orchestrator-only-skills.test.ts

MODIFIED FILES (patch-kit):
  patches/oh-my-opencode-slim/src/cli/superpowers-policy.ts
  patches/oh-my-opencode-slim/src/cli/custom-skills.ts
  patches/oh-my-opencode-slim/src/index.ts
  patches/oh-my-opencode-slim/src/agents/index.test.ts (add new tests)
  opencode-config/README.md
  docs/architecture.md
  docs/verify.md
  CHANGELOG.md (new entry for v1.4.0)

USER LIVE FILES (post-deploy):
  ~/.config/opencode/agents/oracle.md             — append write/todowrite to deny
  ~/.config/opencode/agents/oracle-alpha.md       — append write/todowrite to deny
  ~/.config/opencode/agents/oracle-beta.md        — append write/todowrite to deny
  ~/.config/opencode/agents/oracle-gamma.md       — append write/todowrite to deny
  ~/.config/opencode/agents/oracle-delta.md       — append write/todowrite to deny
  ~/.config/opencode/agents/explorer.md           — append write/todowrite to deny
  ~/.config/opencode/agents/explorer-alpha.md     — append write/todowrite to deny
  ~/.config/opencode/agents/explorer-beta.md      — append write/todowrite to deny
  ~/.config/opencode/agents/librarian.md          — append write/todowrite to deny
  ~/.config/opencode/agents/librarian-alpha.md    — append write/todowrite to deny
  ~/.config/opencode/agents/librarian-beta.md     — append write/todowrite to deny
  ~/.config/opencode/agents/observer.md           — create + frontmatter
  ~/.config/opencode/agents/council.md            — create + frontmatter
  ~/.config/opencode/agents/scout.md              — append write/todowrite to deny
  ~/.config/opencode/agents/validator.md          — append write/todowrite to deny
  ~/.config/opencode/agents/gist.md               — append write/todowrite to deny
  ~/.config/opencode/agents/wildcard.md           — append write/todowrite to deny
  ~/.config/opencode/agents/fixer.md              — create + frontmatter (deny task)
  ~/.config/opencode/agents/designer.md           — create + frontmatter (deny task)
  ~/.config/opencode/agents/laborer.md            — create + frontmatter (deny task)
  ~/.config/opencode/oh-my-opencode-slim.jsonc    — drop redundant manual MCP grants
```

End of spec.
