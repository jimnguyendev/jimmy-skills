---
name: prompt-engineering-domain-research
description: "Templates for research and analysis: paper summaries, literature synthesis, PESTLE, root-cause (5 Whys), gap analysis, source evaluation (CRAAP), and data-analysis methodology guidance. Forces grounded outputs with cited sources and explicit unknowns. AI cannot replace verification — these templates make verification structural."
user-invocable: true
license: MIT
compatibility: "Stack-agnostic. Pair with retrieval (RAG over papers, internal research) for grounded answers; never trust LLM citations without verification."
metadata:
  author: jimnguyendev
  version: "0.1.0"
allowed-tools: Read Edit Write Glob Grep
---

# Domain — Research & Analysis

The single rule for research prompts: **always verify AI claims independently and cite original sources.** These templates make verification structural — the model produces structured artifacts that point at sources, not floating prose.

## Template — paper / source summary

```text
<role>You summarize academic and technical sources for re-use by other researchers.</role>

<task>Summarize the source below into the structured fields.</task>

<inputs>
<source type="paper | article | report | preprint">
{{title, authors, year, venue, DOI/URL}}
{{abstract or full text}}
</source>
<reading_level>{{specialist | adjacent-field | informed-public}}</reading_level>
</inputs>

<constraints>
- Ground every field in the source. If absent: return null. Do NOT invent.
- Quote directly when stating findings.
- Match vocabulary to the reading level.
</constraints>

<output>
JSON:
{
  "citation": "...",
  "thesis": "1–2 sentences",
  "methodology": "...",
  "data": "what was used, how much, how collected",
  "key_findings": ["...", "..."],
  "limitations": ["explicit limitations stated by authors", "..."],
  "relevance_to_{{your_topic}}": "...",
  "open_questions": ["...", "..."],
  "verification_needed": ["claims that need independent check before reuse"]
}
</output>
```

## Template — literature synthesis

For combining N sources into one view:

```text
<role>You synthesize multiple sources into a coherent comparative view.</role>

<inputs>
<sources>
[ {{source 1 with structured summary above}}, {{source 2}}, ... ]
</sources>
<question>{{the comparative question — narrow}}</question>
</inputs>

<output>
## Common themes
Where 3+ sources agree.

## Contradictions
Where sources disagree, with sides and the empirical pivot point.

## Gaps
Questions the literature does NOT yet answer.

## Evolution
How the view has changed across publication years.

## Integrated understanding
Best current synthesis, with confidence level (high / medium / low) and
the conditions under which it would shift.

## Sources by claim
For each major claim above, list the specific sources supporting it.
</output>
```

The "sources by claim" appendix is what makes the synthesis verifiable. Without it, the synthesis is just confident prose.

## Template — PESTLE

```text
<role>You apply the PESTLE framework to surface structural drivers.</role>

<task>Analyze {{domain / market / decision}} across PESTLE axes.</task>

<output>
| Axis | Forces at play | Direction (5–10y) | Implication |
| --- | --- | --- | --- |
| Political | ... | ... | ... |
| Economic | ... | ... | ... |
| Social | ... | ... | ... |
| Technological | ... | ... | ... |
| Legal | ... | ... | ... |
| Environmental | ... | ... | ... |

## Cross-axis interactions
Two examples of axes that compound.

## Top 3 forces to watch
Ranked by impact × velocity.

## Verification needed
Claims that should be checked against current data.
</output>
```

## Template — root cause (5 Whys)

```text
<role>You find root causes, not symptoms.</role>

<inputs>
<observed_problem>{{specific, measurable, recent}}</observed_problem>
<context>{{system, team, recent changes}}</context>
</inputs>

<constraints>
- Each Why is grounded in evidence (a metric, an event, a quote).
- Stop when you reach a cause that, if fixed, would prevent the problem.
- If two parallel causes emerge, branch the chain.
- DO NOT collapse 5 Whys into one — show the chain.
</constraints>

<output>
1. Why did {{problem}} happen?  → {{cause 1, evidence}}
2. Why did {{cause 1}} happen?   → {{cause 2, evidence}}
3. Why ... ?                      → {{cause 3, evidence}}
4. ...
5. Root cause: ...

## Counter-evidence
What would falsify the chain above?

## Recommended fix
At which level to intervene and why.
</output>
```

## Template — gap analysis

```text
<role>You compare current state to desired state and produce an actionable gap map.</role>

<inputs>
<current_state>
{{capability / metric / quality — bullet list with evidence per item}}
</current_state>
<desired_state>
{{same axes as current, target levels}}
</desired_state>
<constraints>{{budget, time, dependencies}}</constraints>
</inputs>

<output>
| Axis | Current | Desired | Gap | Effort | Priority |
| --- | --- | --- | --- | --- | --- |

## High-leverage gaps (top 3)
For each: why it's the most leveraged, and the smallest first move.

## Low-priority gaps
Why we are choosing NOT to close these now.

## Sequencing
What must happen before what.
</output>
```

## Template — source evaluation (CRAAP)

When the model retrieves sources via search and you need to weight them:

```text
<role>You evaluate sources before citing.</role>

<task>Apply the CRAAP test to the source below.</task>

<output>
| Criterion | Score 1–5 | Note |
| Currency  | ... | publication date, recency of data |
| Relevance | ... | match to question + audience |
| Authority | ... | author credentials, publisher |
| Accuracy  | ... | citations, methodology transparency |
| Purpose   | ... | inform / persuade / sell — and any conflicts of interest |

## Verdict
Use as primary | use as supporting | use with caveat (state caveat) | do not use.

## Independent corroboration
What other source(s) support or contradict the central claim?
</output>
```

## Template — data analysis methodology

```text
<role>
You guide the methodology for an analysis. You do NOT have access to the
data — your job is to scope the right questions and the right method.
</role>

<inputs>
<question>{{analytical question}}</question>
<data_description>{{shape, size, source, time range, known biases}}</data_description>
<decisions_it_will_inform>{{what will be done with the answer}}</decisions_it_will_inform>
<constraints>{{tooling, time, statistical sophistication of audience}}</constraints>
</inputs>

<output>
## Sharpened question
The analytical question, rephrased to be measurable and falsifiable.

## Approach
- Method (descriptive / inferential / causal / predictive) and why.
- Required transformations and joins.
- Statistical tests if any, with assumptions to check.
- Visuals that would communicate the result.

## Threats to validity
Sampling, confounding, selection, measurement error. Specific to THIS data.

## Sanity checks
- Counts and totals to verify the dataset.
- Edge cases to spot-check.

## What this analysis CANNOT tell you
Boundary of valid inference. Important.

## Plain-language interpretation template
How to phrase the result for non-statistician stakeholders.
</output>
```

The methodology prompt explicitly notes: **AI cannot access or process your dataset.** Don't paste sensitive data; use the model to scope and interpret, not to compute.

## Citation hygiene

For any prompt asking the model to cite:

```text
<constraints>
- Quote the cited passage directly when stating a finding.
- For each citation: title, author(s), year, venue, DOI or URL.
- If a citation cannot be verified from the provided <sources>: do NOT cite it.
- Mark any claim without a source: [unsupported].
</constraints>
```

Without these constraints, models hallucinate plausible-looking citations. Treat any citation that didn't come from retrieved context as suspect until verified.

## Anti-patterns

- **Trusting model citations.** Hallucinated DOIs and journals are common. Verify or use RAG over a known corpus.
- **5 Whys without evidence at each step.** Becomes opinion, not analysis.
- **Synthesis without per-claim attribution.** The synthesis is unverifiable.
- **PESTLE as a checklist.** Listing forces without direction or interaction is wallpaper.
- **Gap analysis without "low-priority" call-outs.** Treats every gap as equal; nothing gets prioritized.
- **Asking for "the answer" on subjective or contested questions.** Prefer a contested-claims framing: "Here are the strongest views, here's where they disagree."
- **Pasting sensitive data into a methodology prompt.** Use synthetic descriptions; data work happens in your tooling.

## Cost & routing

| Task | Tier |
|---|---|
| Single-paper summary | cheap to mid |
| Multi-source synthesis (5+ papers) | mid → frontier |
| Methodology design / threats to validity | frontier with reasoning |
| Routine PESTLE / SWOT / gap analysis | mid |
| High-stakes lit review (with retrieved corpus) | frontier with RAG |

## Cross-references

- `jimmy-skills@prompt-engineering-context` — RAG over a known corpus is essential for citation hygiene.
- `jimmy-skills@prompt-engineering-output-json` — structured summaries.
- `jimmy-skills@prompt-engineering-chain` — search → summarize → synthesize.
- `jimmy-skills@prompt-engineering-edge-cases` — confidence + verification fields.
