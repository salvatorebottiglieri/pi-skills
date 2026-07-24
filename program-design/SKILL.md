---
name: program-design
description: >
  Produce program design artifacts (call-stack tree, file-tree diff, type
  signatures) for a ticket and append them to the issue on the tracker.
  Use between /to-issues and implement-loop to give the implement and review
  subagents a concrete contract before they write code.
disable-model-invocation: true
---

# Program Design

Bridge the gap between **architecture** (how services talk, what the schema
looks like) and **implementation** (the actual code). Most code written by
agents has the right architecture but wrong *shape* — wrong function names,
wrong parameters, wrong file boundaries. Program design fixes the shape
before an agent writes a line.

`$ARGUMENTS` is the ticket/issue to design. Pass a number, URL, or path.

The issue tracker should have been provided to you — run
`/setup-matt-pocock-skills` if not.

## Process

### 1. Gather context

- **Read the ticket.** Fetch it from the tracker. Read the full body, comments,
  acceptance criteria, and blocking edges.
- **Read the terrain.** Explore the files and modules the ticket touches. Read
  `CONTEXT.md` for domain vocabulary. Read relevant ADRs in `docs/adr/`. Read
  existing function signatures, types, and test patterns in the area.
- **Classify the change.** Determine what kind of work this is (see §3). This
  decides which artifacts are worth producing.

### 2. Produce artifacts

For each artifact type that fits the change, produce a draft. Do **not** write
code or pseudocode for the implementation body — only the *shape*.

#### Call-stack tree

Produce this when the change introduces new orchestration or modifies control
flow (new route handler, new service method, new branch in an existing function).

Show the call stack with diff syntax:

```diff
 entrypoint
   authenticate
+    handleCreateResource
+      validateInput(input)
+      resourceRepo.insert(input)
+      notifySubscribers(resourceId)
   sendResponse
```

+ = nuovo  ~ = modificato  - = rimosso

Include only the functions that are part of **this** change. Existing functions
that don't change are reference context, not part of the diff.

If the change is purely data or config (add a field, toggle a flag), skip this
section entirely.

#### File-tree diff

Produce this when the change creates or modifies files.

```diff
 src
 └── resource
+    ├── resource-client.ts       # NEW — wraps API contract calls
+    ├── resource-client.test.ts  # NEW — request/response mapping
~    └── resource-route.ts        # MODIFIED — wires create action
```

+ = creato  ~ = modificato  - = rimosso

Each new or modified file gets a one-line rationale. Test files follow the
project's naming convention.

If the change doesn't add or remove files (only edits inside existing files),
omit the tree and note the files modified in prose.

#### Type signatures

Produce this when the change introduces new functions, modifies existing
signatures, or defines new data types.

Use the project's language (TypeScript, Python, Go, etc.):

```ts
// Nuove
interface ResourceInput {
  name: string
  sourceUrl: string
  tags: string[]
}

function createResource(input: ResourceInput): Promise<Resource>

// Modificate
// resource-route.ts — PUT /resources/:id now accepts `tags`
function handleCreateResource(req: Request): Promise<Response>
```

**Rules:**
- Every function in the call-stack tree MUST have a signature here.
- Names use the domain vocabulary from `CONTEXT.md`.
- If a function is called but not modified, reference its existing location
  ("vedi signature in src/resource/resource-service.ts") — don't repeat it.
- Types are concrete, not generic. `Promise<Resource>`, not `Promise<T>`.
- If the change adds or modifies zero functions (config only, copy tweak),
  skip this section.

### 3. When to skip (classify the change)

Before producing artifacts, classify the ticket. If it matches any of these,
skip program design entirely, append a one-line note to the ticket, and stop:

| Tipo | Esempio | Ragione |
|---|---|---|
| **Bug fix** | "NullPointerError quando include è vuoto" | La diagnosi è il lavoro vero; il fix è 2 righe |
| **Refactor meccanico** | "Rename `UserService` → `AccountService`" | LSP rename basta |
| **Feature banale** | "Aggiungi campo `phone` al profilo" | Segue esattamente il pattern esistente |
| **One-shot** | Script monouso, migrazione dati | Non serve design riusabile |

Se non sei sicuro, fallo. 5 minuti di design > 2 ore di riwork.

### 4. Append to the ticket

Append a `## Program Design` section to the issue body on the tracker.

**How** depends on the tracker configured in `/setup-matt-pocock-skills`:

- **GitHub:** Read the current body with `gh issue view <number> --json body`,
  extract the body text, append the section, write back with
  `gh issue edit <number> --body "<full-body>"`.
- **Local files:** Read the file, find the end of the content (before
  `## Comments` if it exists), insert the section there, write back.
- **GitLab / Linear:** Follow the tracker's convention for updating issue
  descriptions (see the tracker doc from setup).

The appended section uses this structure:

```markdown
## Program Design

_CLASSIFY: <tipo di cambiamento>_

### Call-stack tree

```diff
...
```

### File-tree diff

```diff
...
```

### Type signatures

```ts
...
```
```

Omit sections that are empty — don't produce "N/A" placeholders.

### 5. Stop for review

Do **not** proceed to implementation. Report what was produced and ask:

> "Il program design per questo ticket è pronto. I signatures sono contratto
> per l'implementazione. Vuoi modificare qualcosa o procedere con
> implement-loop?"

If the user corrects anything, incorporate the fix and re-append the section
(overwrite the previous one). Only when they approve does the ticket move to
`ready-for-agent` (or the implement-loop picks it up).

## How other skills use this

- **implement-loop:** Passes the full ticket body to the implement subagent.
  If `## Program Design` is present, the subagent treats the signatures there
  as contract — it implements exactly those interfaces without guessing.
- **code-review (in implement-loop):** The spec-axis reviewer compares the
  diff against the ticket body, which now includes the program design.
  "Signature dice `createResource(input: ResourceInput): Promise<Resource>`,
  ma l'implementazione usa `save(input)` e restituisce `any`."

## Relationships with other skills

| Skill | Relazione |
|---|---|
| `codebase-design` | Fornisce il lessico (deep module, seam). Program design è l'applicazione *concreta*: "la seam è questa signature." |
| `domain-modeling` | Program design usa il vocabolario di `CONTEXT.md`. Se emerge un termine ambiguo, chiama `domain-modeling`. |
| `to-issues` | Produce i ticket. Program design prende UN ticket e lo dettaglia con le signature. |
| `prototype` | Se il dubbio di design è troppo grosso per program design (es. "non so che forma deve avere"), prima fai un prototype, poi program design sui risultati. |
| `improve-codebase-architecture` | Se durante program design emerge un dubbio architetturale ("questa funzione non dovrebbe stare qui"), escalalo — non risolverlo in questa skill. |

## Classification cheat sheet

| Se il ticket dice… | Allora è… | Program design? |
|---|---|---|
| "Aggiungi endpoint X" | Feature con orchestrazione | Sì — call-stack + signatures |
| "Aggiungi campo X al modello Y" | Feature banale | Skip, o solo signatures se il campo cambia logica |
| "Refactor: estrai modulo X da Y" | Refactor meccanico | Skip |
| "Bug: crash quando X è null" | Bug fix | Skip |
| "Integra API esterna Z" | Feature con orchestrazione | Sì — call-stack + file-tree + signatures |
| "Aggiorna dipendenza X a v2" | One-shot | Skip |
