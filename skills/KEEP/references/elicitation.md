# Eliciting tacit knowledge — taxonomy and examples

This file is consulted by `/compile` (and, more rarely, by `/observe` when the diff is ambiguous) before asking the user questions to capture knowledge that the diff alone cannot establish.

The protocol is in `SKILL.md` under **Eliciting tacit knowledge**. This file is the operational detail: *which* questions are worth asking for each file type, what good vs bad questions look like, and how to stay below the noise floor.

## The cardinal rule

A good question produces information that **changes what gets written**. A bad question is one whose answer doesn't actually affect the output, or whose answer the user has already given implicitly elsewhere.

If you can write a sensible, conservative version of the field without asking — ask only if the missing detail is load-bearing.

## The evidence-first rule

Before asking *any* question, scan the available evidence: the diff, the commit messages, the PR description, recent conversation history. If the load-bearing information is already present in any of these, **quote it into the file and do not ask**.

Examples of evidence that bypasses elicitation:

- **Commit message:** *"feat: switched to Ray Serve. Tried KServe first but the CRD complexity was unmanageable for our team size. BentoML lacked distributed support."* → All three alternatives and their rejection rationales are on record. Write the ADR with this content (with a `<!-- source: commit a3f21b -->` comment) and skip the elicitation.

- **PR description:** *"This change handles three edge cases the current implementation misses: expired tokens, rotated keys mid-request, and clock skew >30s between API and Auth0."* → Edge cases are stated. Lift them into the spec verbatim; do not ask.

- **User said earlier in conversation:** *"We picked Postgres because we need ACID and the data volume isn't large enough to justify a distributed store."* → Decision rationale is captured. Write the ADR; do not ask.

Quote the evidence in the file with a brief source marker (commit SHA, PR number, or "stated in conversation 2026-05-12"). This makes the lineage traceable and makes it clear the content is not invented.

Only ask when none of the available evidence covers the load-bearing field. Asking for what the user already wrote down is friction without value.

## Question taxonomy by file type

### When writing an ADR

The three load-bearing fields are **Alternatives considered**, **Consequences**, and **Drivers**. Without them, an ADR degrades into a status note. Each carries a different kind of rationale:

- **Alternatives considered** — *why didn't we pick X?*
- **Consequences** — *what cost are we accepting?*
- **Drivers** — *why this option specifically, on its own merits?*

A good batch interview for a new ADR asks one question for each, in that order.

**Worth asking:**

1. *"What other options did you consider for this, even briefly?"* — opens the alternatives space without prejudging which were serious. Multi-select with candidates from the diff's ecosystem plus "Other".
2. *"What costs or downsides did you accept by picking this option?"* — surfaces the tradeoffs. Multi-select with common categories (operational complexity, vendor lock-in, breaking change, payload size, coupling, etc.) plus "Other".
3. *"What made this option specifically right — what factors drove the choice?"* — surfaces the positive drivers. Multi-select (single-select is risky here — see "Designing the options well" below).

**Avoid:**

- *"Why did you make this decision?"* — too open; you'll get a story rather than structured rationale across the three dimensions.
- *"Is this a good decision?"* — not your job to evaluate; just record.
- *"Should I list X as an alternative?"* — leading; let the user volunteer alternatives.

**Example (good):**

> I'm about to create `ADR-0014-ray-serve.md` for the Ray Serve introduction. The diff shows the switch but not the rationale. Quick batch:
> 1. Were KServe, BentoML, or anything else on the table? (multi-select: KServe / BentoML / custom FastAPI orchestration / Triton-direct / Other)
> 2. What tradeoffs did you accept? (multi-select: tighter Ray coupling / operational complexity of a Ray cluster / Python-only orchestration / Other)
> 3. What drove the choice toward Ray Serve specifically? (multi-select: scalability / Python ergonomics / Kubernetes integration / team familiarity / Other)

### When writing a new spec

The load-bearing fields are **Edge cases** and **Acceptance criteria**. The happy path is visible in the code; what is *deliberately* handled and what proves the implementation is correct are not.

**Worth asking:**

1. *"What cases does this handle that aren't obvious from the happy path?"* — pulls out edge cases that exist in code as defensive branches.
2. *"What would have to be true at runtime for you to say this works?"* — drives acceptance criteria toward observable, testable conditions.
3. *"What's the expected behavior when [specific failure mode visible in the code, e.g. a network call fails]?"* — concrete and answerable; better than asking abstractly for failure modes.

**Avoid:**

- *"What are the edge cases?"* — too open; you'll get a shrug.
- *"What's the goal of this feature?"* — the diff usually shows this; asking suggests you didn't read it.
- *"Should I add more requirements?"* — Claude's judgment, not the user's.

**Example (good):**

> Creating `specs/auth/jwt.md` from the new code. I can see the happy path and the issuer/audience validation. Two things the diff is ambiguous about:
> 1. The code has retry logic on JWKS fetch — what failure is that defending against? (network blip / Auth0 rate limit / key rotation race / not sure)
> 2. If an invalid token comes in, what should the API return? (401 with generic message / 401 with reason / 403 / depends on which validation failed)

### When writing a runbook

The load-bearing fields are **Symptoms** and **Causes**. Speculative runbooks are worse than no runbook — they send on-call engineers down false paths.

**Worth asking:**

1. *"Did this failure actually happen, or are we documenting a hypothetical?"* — if hypothetical, do not write the runbook. Skip.
2. *"What was the first thing you noticed when it happened — error message, metric, user report?"* — anchors the symptoms in observed reality.
3. *"How did you arrive at the cause? Was there a clear signal or did you have to dig?"* — useful for the "how to diagnose" portion of the runbook.

**Avoid:**

- *"What could go wrong with this code?"* — generates speculative runbooks. Refuse to write a runbook on this basis.
- *"What's the mitigation?"* — usually visible in the fix itself; ask only if the fix is partial.

**Example (good):**

> The commit message references a JWKS rate-limit incident. Before I write `runbooks/jwks-rate-limit.md`:
> 1. What was the first symptom — 401s spiking, latency, a specific log line? (pick one or describe)
> 2. How did you diagnose it was rate-limiting from Auth0 specifically?
> 3. Is the retry-with-backoff a full fix, or is there a follow-up needed?

### When updating architecture

The load-bearing fields are **Boundaries** and **Constraints**. What the component *isn't* and what invariants must hold.

**Worth asking:**

1. *"What's deliberately outside this component's responsibility?"* — surfaces the negative space, which is often where bugs hide later.
2. *"Is there an invariant this enforces that callers can rely on?"* — captures contracts that aren't in the type signatures.

**Avoid:**

- *"What does this component do?"* — visible in the code and the spec.

## The anti-bombardment heuristic

Four rules to stay below the noise floor:

1. **Hard cap: three questions per turn.** If you have more, ask the top three and flag the rest for the next interaction. Pick the three by load-bearing value, not by category coverage — three good ADR questions beat one each for ADR, spec, and runbook.

2. **Choose the right mode (batch vs reactive).** See the next section.

3. **Skip questions the user has already answered.** Scan the conversation history. If the user said "we picked Ray Serve over KServe because of Python ergonomics" three turns ago, don't ask about rejected alternatives again — use that.

4. **One escalation rule: if the user declines or pushes back, stop asking.** Write the conservative version (high-stakes fields omitted, TODO marker added) and move on. Do not negotiate.

## Batch mode vs reactive mode

The two modes serve different file types. Picking the wrong one wastes turns or produces inconsistent files.

### When to use batch mode

Use batch mode for **new ADRs** and **new specs**. The load-bearing fields in these files are correlated — you can't write good `Consequences` without knowing the `Alternatives considered` first, and you can't pick `Acceptance criteria` without knowing the `Edge cases` scope. Asking one at a time produces inconsistencies (you've already drafted the consequences, now the user reveals an alternative that would have changed them) and wastes turns.

Batch protocol:

1. **Draft the entire file tentatively** based on what the diff and existing context support, with the high-stakes fields filled with your best conservative guess marked clearly.
2. **Collect all questions in a single turn.** Use `ask_user_input_v0` with multiple questions when on Claude.ai, or a single numbered list in terminal environments.
3. **Wait for all answers** before writing the final file.
4. **Write the file once**, with the answers integrated. Do not write a partial file and iterate.

Example — batch interview for a new ADR:

> I'm about to create `ADR-0014-ray-serve.md`. Here's the draft based on the diff alone:
>
> ```
> # ADR-0014: Use Ray Serve for inference
> ## Status: Accepted
> ## Context: The previous direct FastAPI → Triton path didn't scale ...
> ## Decision: Use Ray Serve as the orchestration layer ...
> ## Alternatives considered: (need input)
> ## Consequences: (need input)
> ```
>
> Three quick questions to fill in the rest — then I'll write the file:
>
> 1. **What else was on the table?** (multi-select: KServe / BentoML / custom FastAPI orchestration / Triton-direct scaling / nothing serious)
> 2. **What was the deciding factor?** (single-select: Python ergonomics / distributed support / team familiarity / Triton integration / other)
> 3. **What should we revisit if circumstances change?** (free text, optional)

This is a single turn. The user answers all three; the file is written in the next turn.

### When to use reactive mode

Use reactive mode for **runbooks** and for **incremental updates** to existing specs/ADRs/architecture files. Runbook fields are mostly independent of each other (symptoms, causes, mitigation can each be answered in isolation). Incremental updates touch one specific thing — asking about unrelated parts of the file is noise.

Reactive protocol: ask only the question needed for the specific update at hand, write the update, move on. Multiple updates to the same file in one `/compile` should still batch their questions if the questions are correlated, but the file itself is updated incrementally.

## When the user declines or answers partially

### Full decline

Acceptable forms of full decline include "skip it", "just write your best guess", "I'll fill it in later", and silence on the question.

Do **not** fabricate. Write the file with the uncertain fields **omitted**, and insert a marker:

```md
<!-- TODO(KEEP): rejected alternatives not captured at compile time -->
```

`/govern` will surface files with these markers in its next periodic run, giving the user a natural prompt to enrich them later when the context is fresher.

### Partial answers in batch mode

When you asked three questions in batch and the user answered, say, only two — leaving the third blank, marking it "skip", or saying "I don't know" — handle each unanswered field individually:

- If the unanswered field is **load-bearing** (rejected alternatives in an ADR, edge cases in a new spec, root cause in a runbook): omit the section entirely and insert a TODO marker for that specific field:
  ```md
  ## Alternatives considered
  <!-- TODO(KEEP): not captured at compile time -->
  ```

- If the unanswered field is **enrichment** (e.g. "what should we revisit later", "any related work"): simply omit it from the file. No marker. Its absence is not a problem and creating a TODO for it just adds noise.

**Do not re-ask in a follow-up turn.** The user already saw the questions and chose to answer some and skip others. Pushing for the missing answers is the interrogation pattern this protocol is designed to prevent. The TODO marker preserves the possibility of enrichment later, when `/govern` surfaces incomplete files in a natural moment.

Write the file once with what you have. Move on.

## Structured input on Claude.ai

When `ask_user_input_v0` is available (Claude.ai chat interface), present discrete choices as tappable options. This drastically reduces friction. Patterns that work well:

- **Single-select** for "which one": *"What forced this choice?"* with options drawn from common forcing factors (constraint type, alternative comparison, etc.)
- **Multi-select** for "which of these apply": *"Which alternatives were considered?"* with the candidates dedupable from the diff's neighborhood and ecosystem.
- **Free text** only for genuinely open answers, or as the last option in a select ("Other — I'll describe it").

In environments without the tool (Claude Code, Cursor), do the same conceptually with numbered lists the user can answer with a digit.

### Designing the options well

The quality of an elicitation question lives in its options. Three rules:

1. **Single-select options must be mutually exclusive.** If the user could reasonably want to pick two, the question should be multi-select instead. Common mistake: framing "What was the deciding factor?" as single-select when the real answer is often "a combination of these three". When in doubt, prefer multi-select and let the user pick one if that's their reality.

2. **Cover the answer space.** The options should be reasonably exhaustive for the situation at hand. Always include an "Other — I'll describe it" or, for multi-select, an "All of the above" fallback. If the user lands in "Other", follow up with a single free-text question on the next turn.

3. **Don't combine multiple dimensions in one option.** Bad: *"Python ergonomics + native distributed support"* as a single option — that's actually two factors. The user might want one without the other. Each option should describe one thing.

### Handling user answers that don't fit your option list

If the user's answer reveals that your options were structured wrong — they pick multiple in a single-select, give a free-text answer that combines categories, or write something orthogonal — **do not re-ask with better options**. The user has now given you the information. Use it. Re-asking would be punishing them for your option-design mistake.

In the file, render their actual answer faithfully. If they said "scalability, python adoption, and kubernetes capabilities" to a single-select, write all three in the resulting document. The structured-input layer is a friction-reducer, not a constraint on what the user can mean.

