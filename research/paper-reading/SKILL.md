---
name: paper-explainer
description: >
  Deep-read and explain ML/CV/AI research papers from arxiv or uploaded PDFs.
  Use this skill whenever the user shares a paper link or PDF and asks about
  its content — including equations, methods, contributions, comparisons to
  prior work, or conceptual questions. Also trigger for follow-up questions
  like "how does equation N work?", "explain this intuitively", or "show me a diagram of this". Always use
  this skill when a paper URL or identifier (e.g. arxiv:XXXX.XXXXX) appears
  in the user's message alongside any reading/explaining request.
---

# Paper Explainer

A skill for reading, deeply understanding, and explaining ML/CV/AI research
papers. Handles equations, method comparisons, visual diagrams, and
multi-turn conceptual Q&A.

---

## Step 1 — Fetch the paper

Try fetching in this priority order until you have readable content:

1. **HTML version** (best): `https://arxiv.org/html/{id}v{N}`
   — Full text, equations as MathML, figures described.
2. **PDF with text extraction**: `https://arxiv.org/pdf/{id}v{N}`
   — Use `web_fetch` with `web_fetch_pdf_extract_text: true`.
3. **Abstract page**: `https://arxiv.org/abs/{id}`
   — Falls back to abstract + any search results for supplementary detail.
4. **Web search**: search `"{title}" arxiv site:arxiv.org` and fetch the
   HTML version from the result URL directly.

If the HTML URL returns a rate-limit error, wait a moment and retry once,
then fall back to the abstract + web search for details.

**Never** tell the user you cannot read the paper without exhausting all
four options.

---

## Step 2 — Identify the request type

After fetching, classify what the user is asking:

| Request type | How to handle |
|---|---|
| "Explain the main contribution" | See §3 |
| "Explain equation N" | See §4 |
| "How does X compare to Y?" | See §5 |
| "I don't understand X, explain simply" | See §6 |
| "Show me a diagram" | See §7 |

A single message may combine multiple types — address them in order.

---

## Step 3 — Main contribution summary

Structure the summary around these four questions:

1. **What problem does it solve?** — State the limitation of prior work in
   one or two sentences. Be specific (e.g. "quadratic memory in full
   attention", "catastrophic forgetting in RNNs").
2. **What is the core technical idea?** — The single insight that makes the
   paper work. Avoid listing all contributions; isolate the one thing.
3. **How does it work mechanically?** — Method in 3–5 sentences. Enough
   that a reader could re-implement at a high level.
4. **What does it achieve?** — Key results vs. the strongest baseline.

Keep the summary under 400 words. Add diagrams (§7) proactively if the
method has a non-trivial architecture or data flow.

---

## Step 4 — Equation explanation

For each equation:

1. **Write the equation cleanly** using LaTeX-style math in prose
   (`$...$`). Identify its number and section in the paper.
2. **Name each symbol** in a small table: symbol | shape/type | meaning.
3. **State the objective** — what loss or operation the equation defines.
4. **Derive the gradient or expansion** if the equation is the result of
   optimising a loss (show the loss → gradient → closed form steps).
5. **Give the intuition** — what the equation *does* in one sentence
   stripped of symbols (e.g. "it updates the state only when the current
   observation is poorly recalled").
6. **Flag any subtleties** — assumptions, edge cases, or places where the
   paper's notation is ambiguous.

### Common pitfalls to avoid

- Do not say "K and V are equal because both come from X" — they use
  separate projection matrices W_K ≠ W_V with different roles (matching
  vs. payload). Always clarify this if K/V confusion arises.
- Do not conflate "TTT as conceptual framing" with "TTT as actual gradient
  descent on model weights". Check whether slow weights are frozen at
  inference before claiming the paper does real TTT.
- Residual / reconstruction errors in fixed-size state models are
  non-trivial due to capacity saturation — explain this if asked.

---

## Step 5 — Method comparison

When asked to compare paper A to paper B (or to prior work):

1. Build a comparison table: rows = key dimensions, columns = methods.
   Useful dimensions: memory complexity, compute complexity, training-free?,
   output format, length generalisation, explicit/implicit representation.
2. Identify where the methods share lineage vs. where they genuinely differ.
3. Be honest about "TTT in name only" — some papers reframe an existing
   update rule through the TTT lens without performing actual gradient
   descent on parameters at inference. Call this out explicitly.

---

## Step 6 — Simple / intuitive explanation

When the user says they are confused or asks for a simpler explanation:

- Drop all notation. Explain using an analogy (library lookup, fast/slow
  memory, writing in a notebook, etc.).
- Use the "what" before the "how" — establish what the mechanism achieves
  before explaining how it achieves it.
- Confirm understanding with a one-line "so in short: ..." summary at the
  end.
- Offer to go deeper with a diagram (§7) if the mechanism is spatial or
  sequential.

---

## Step 7 — Diagrams

Always offer or proactively draw a diagram when:
- The paper introduces a new architecture with a write/read split
- The method has a non-obvious data flow
- The user says they are confused
- A comparison table alone would not convey the difference

### Diagram style

Use `show_widget` (Visualizer tool). Call `read_me` with
`["diagram", "interactive"]` before the first diagram in a session.

**For architecture diagrams** (write/read pipelines, encoder-decoder flows):
- Use `c-blue` for input tokens, `c-coral` or `c-amber` for the memory /
  state component, `c-teal` for query/virtual tokens, `c-green` for outputs.
- Label arrows WRITE / READ when there is a two-phase pattern.
- Keep to ≤ 6 boxes. Complexity goes in prose, not in the SVG.

**For method comparison** (two or three methods side by side):
- Use an HTML tab stepper — one tab per method, same SVG layout.
- Each tab ends with `✓` / `✗` lines summarising the trade-offs.

**For intuition diagrams** (explaining how attention, fast weights, or
associative memory work):
- Draw the mechanism spatially: tokens as boxes, weights as line thickness,
  states as filled matrices.
- Keep labels to ≤ 5 words per box; put details in prose between diagrams.

**General rules**:
- Never stack multiple `show_widget` calls without prose between them.
- First diagram = the problem / motivation. Second = the solution. Third
  (optional) = comparison or ablation insight.
- Always write a short paragraph before each diagram explaining what it shows.

---

## Multi-turn Q&A

After the initial explanation, remain in "paper expert" mode for follow-up
questions. Common patterns and how to handle them:

| Follow-up | Approach |
|---|---|
| "Does this really use TTT?" | Audit whether slow weights update at inference. Answer honestly. |
| "Why K ≠ V in self-attention?" | Explain role separation: K = discriminative addressing, V = rich content payload. Forced sharing is a conflicting objective. |
| "What happens if the state is saturated?" | Explain finite-capacity interference: outer-product superposition, Hopfield-style recall degradation. |
| "How does this compare to [other paper]?" | Fetch the other paper if not already in context, then build a comparison table (§5). |
| "Can you make a diagram?" | Jump to §7. |

---

## Quality checklist

Before delivering any explanation, verify:

- [ ] Fetched the paper (not relying on training memory for equation details)
- [ ] Equation symbols are defined with shapes and semantics
- [ ] No conflation of K=V, TTT-as-metaphor, or trivial-residual mistakes
- [ ] Comparison to prior work is grounded in what the paper actually claims
- [ ] Diagrams use correct color conventions and have prose context
- [ ] Multi-turn answers stay anchored to the paper's actual text
