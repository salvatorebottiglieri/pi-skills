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

| Type | Example | Reason |
|---|---|---|
| **Bug fix** | "NullPointerError when include is empty" | The diagnosis is the real work; fix is 2 lines |
| **Mechanical refactor** | "Rename `UserService` → `AccountService`" | LSP rename is enough |
| **Trivial feature** | "Add `phone` field to profile" | Follows existing pattern exactly |
| **One-shot** | One-off script, data migration | No reusable design needed |

| When in doubt, do it. 5 minutes of design > 2 hours of rework.

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

_CLASSIFY: <change type>_

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

> "The program design for this ticket is ready. The signatures are the
> contract for implementation. Would you like to modify anything or proceed
> with implement-loop?"

If the user corrects anything, incorporate the fix and re-append the section
(overwrite the previous one). Only when they approve does the ticket move to
`ready-for-agent` (or the implement-loop picks it up).

## How other skills use this

- **implement-loop:** Passes the full ticket body to the implement subagent.
  If `## Program Design` is present, the subagent treats the signatures there
  as contract — it implements exactly those interfaces without guessing.
- **code-review (in implement-loop):** The spec-axis reviewer compares the
  diff against the ticket body, which now includes the program design.
> "Signature says `createResource(input: ResourceInput): Promise<Resource>`,
>  but the implementation uses `save(input)` and returns `any`."

## Relationships with other skills

| Skill | Relationship |
|---|---|
| `codebase-design` | Provides vocabulary (deep module, seam). Program design is the *concrete* application: "the seam is this signature." |
| `domain-modeling` | Program design uses `CONTEXT.md` vocabulary. If an ambiguous term emerges, call `domain-modeling`. |
| `to-issues` | Produces tickets. Program design takes ONE ticket and details it with signatures. |
| `prototype` | If the design question is too broad for program design (e.g. "I don't know what shape this should take"), prototype first, then program design on the results. |
| `improve-codebase-architecture` | If an architectural concern emerges during program design ("this function shouldn't be here"), escalate it — don't resolve it in this skill. |

## Classification cheat sheet

| If the ticket says… | Then it's… | Program design? |
|---|---|---|
| "Add endpoint X" | Feature with orchestration | Yes — call-stack + signatures |
| "Add field X to model Y" | Trivial feature | Skip, or only signatures if the field changes logic |
| "Refactor: extract module X from Y" | Mechanical refactor | Skip |
| "Bug: crash when X is null" | Bug fix | Skip |
| "Integrate external API Z" | Feature with orchestration | Yes — call-stack + file-tree + signatures |
| "Update dependency X to v2" | One-shot | Skip |
