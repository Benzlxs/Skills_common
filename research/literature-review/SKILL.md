---
name: literature-survey
description: >
  Use this skill whenever the user asks for a literature review, related work section, research survey,
  or paper lineage on any ML/CV/AI topic. Trigger on phrases like "do a literature review of X",
  "what are the papers after X", "related work on Y", "survey of Z papers", "what came after paper X",
  "lineage of X research", "how has X evolved". Also trigger when the user uploads or describes a paper
  and asks what followed from it, or asks for a chronological figure of a research area.
  This skill produces: (1) a thorough web-searched paper catalog organized chronologically with per-paper
  summaries, (2) extracted research trends, and (3) an interactive multi-track lineage diagram with
  hover tooltips. Always use this skill for related work requests — even partial ones like "add more papers
  on topic X to the existing survey".
---

# Literature Survey + Lineage Diagram

A skill for producing comprehensive, searchable literature surveys on ML/AI/CV research topics,
paired with an interactive chronological lineage diagram that visualizes paper relationships across
research tracks.

---

## Overview

This skill executes a three-phase pipeline:

1. **Search** — systematic multi-query web search covering arXiv, Semantic Scholar, and venue proceedings
2. **Catalog** — chronological per-paper summaries organized into research tracks with trend extraction
3. **Visualize** — interactive multi-track HTML lineage diagram with hover tooltips and cross-track influence lines

The output is a written survey artifact plus an inline interactive diagram. The diagram is the primary
deliverable for visual communication; the written survey is the primary deliverable for reference.

---

## Phase 1: Search Strategy

### Step 1 — Anchor paper identification

Before searching, identify the anchor paper(s):
- Extract title, authors, venue, date, arXiv ID
- Note the paper's core contribution in one sentence (this seeds search queries)
- Identify the 2–3 most-cited predecessors (these become "context precursors" in the diagram)

### Step 2 — Multi-angle search queries

Run **at least 6 distinct web searches** covering different angles. Do not rely on a single broad query.
Use `web_search` and `web_fetch` for full paper pages.

Required query types:
```
1. Direct follow-up: "<paper title> follow-up extensions 2024 2025"
2. Method name: "<core method name> improvements arXiv"
3. Application domain: "<method> applied to <domain> 2025"
4. Venue scan: "<topic> CVPR ICCV NeurIPS ICML 2025 2026"
5. Competitor methods: "<paper> vs <competing approach> comparison"
6. Theoretical: "<method> theory analysis convergence 2025"
```

For 3D vision / video topics, add:
```
7. "<method> 3D reconstruction video generation"
8. "<method> streaming online long-context video"
9. "test-time <domain> arXiv 2025 2026"
```

### Step 3 — Per-paper data extraction

For each paper found, record:
- Full title, authors (first + et al.), venue or arXiv ID, month/year
- One-sentence core contribution
- Specific improvement over the anchor paper (what limitation does it address?)
- Whether it is a **key paper** (shifts field direction) or incremental
- Whether it is a **context precursor** (foundation model the method was applied to, not a TTT/method paper itself)

### Step 4 — Scope boundaries

**Include:**
- Direct extensions and variants of the method
- Papers that apply the method to a new domain
- Theoretical analyses of the method
- Papers that compete with or unify the method
- Concurrent work from the same time period

**Exclude:**
- Papers that merely cite the anchor without building on it
- General surveys that mention the method in passing
- Papers outside the requested scope (e.g., if asked for 3D vision only, exclude pure NLP)

---

## Phase 2: Catalog and Written Survey

### Structure

Organize the written survey with this structure:

```markdown
# [Topic] Literature Survey

## Foundational context and concurrent work
[2–3 paragraphs: the intellectual lineage leading to the anchor paper,
 concurrent work from the same month/year]

## Chronological catalog of follow-up papers

### [Year] [Quarter]: [Descriptive subtitle]
**[Paper Title]** — [Authors], arXiv [ID] / [Venue], [Month Year].
[2–4 sentences: what it does, what it improves, key result if available]

[repeat for each paper in this period]

## [N] Major trends shaping [topic] research

**Trend 1: [Name]** — [2–3 sentences]
[repeat for each trend]

## Notable research forks
[Describe 2–4 forks: e.g., "language vs vision", "efficiency vs expressiveness",
 "Group A vs Group B", "memorization vs reasoning"]

## Conclusion
[2–3 sentences synthesizing trajectory]
```

### Trend identification criteria

A "major trend" must satisfy all three:
1. At least 3 papers independently pursue it
2. It represents a deliberate response to a specific limitation of the anchor paper
3. It has a clear direction (not just "people are using X more")

Good trend names are action-oriented: "The inner loop is getting richer", "Hardware efficiency is no
longer a bottleneck", "3D vision is the breakout application domain".

---

## Phase 3: Interactive Lineage Diagram

### Track assignment

Assign each paper to exactly one track. Tracks should represent **research communities or problem
framings**, not just topics. Good track names: "Core Architecture", "Google Memory", "Theory &
Unification", "Video Generation", "3D Reconstruction", "Domain Adaptation".

Rules:
- 4–6 tracks maximum (more = unreadable at 960px width)
- Each track has a distinct color from the palette: purple, teal, amber, blue, coral, gray
- Context precursors get dashed borders within their track
- Key papers get double-stroke borders

### Diagram HTML template

Use the following HTML widget structure. The key design decisions are:

1. **Container width**: 960px with horizontal scroll wrapper — allows 6 tracks at 149px each
2. **Node layout**: absolute positioned divs, not SVG text — allows proper CSS theming
3. **SVG overlay**: used only for connectors and shaded era bands, not for nodes
4. **Hover tooltips**: inline JS, positioned to avoid viewport overflow
5. **Down edges**: solid lines within a track (sequential development)
6. **Cross edges**: dashed bezier curves between tracks (influence/inspiration)
7. **Era bands**: shaded `<rect>` elements grouping papers by time period
8. **Context zone**: separate shaded rect highlighting precursor papers

```html
<style>
/* Scroll wrapper — allows 6-track diagram to extend past widget width */
.ttw{overflow-x:auto;width:100%}
.ttc{position:relative;width:960px;font-family:var(--font-sans)}

/* Column headers */
.tthdr{display:flex;margin-left:64px;gap:2px;height:48px;margin-bottom:6px}
.ttth{flex:0 0 146px;font-size:10px;font-weight:500;text-align:center;
      border-radius:6px;border:1px solid;display:flex;align-items:center;
      justify-content:center;padding:4px;line-height:1.3}

/* Main body — height is calculated from tallest track's last node y + NH + 60 */
.ttbody{position:relative;height:TOTAL_HEIGHT_PX}
.ttsvg{position:absolute;top:0;left:0;width:960px;height:TOTAL_HEIGHT_PX;pointer-events:none}

/* Paper nodes */
.ttn{position:absolute;width:134px;border-radius:6px;border:1px solid;
     padding:4px 6px;box-sizing:border-box;cursor:default;
     transition:transform .12s;z-index:2}
.ttn.key{border-width:2px}      /* key paper */
.ttn.ctx{border-style:dashed;opacity:0.72}  /* context precursor */
.ttn:hover{transform:scale(1.07);z-index:50}
.tnt{font-size:10.5px;font-weight:500;line-height:1.35}
.tns{font-size:9.5px;line-height:1.2;margin-top:1px;opacity:0.68}

/* Date labels (left margin) */
.ttdl{position:absolute;left:2px;font-size:9px;font-weight:500;
      color:var(--color-text-tertiary)}

/* Hover tooltip */
.tttip{display:none;position:absolute;z-index:200;
       background:var(--color-background-primary);
       border:0.5px solid var(--color-border-secondary);
       border-radius:8px;padding:8px 10px;font-size:10px;
       max-width:186px;line-height:1.5;color:var(--color-text-secondary);
       pointer-events:none}

/* Color classes — light/dark mode via CSS variables */
.tc0{background:#EEEDFE;border-color:#534AB7;color:#3C3489}  /* purple */
.tc1{background:#E1F5EE;border-color:#0F6E56;color:#085041}  /* teal */
.tc2{background:#FAEEDA;border-color:#854F0B;color:#633806}  /* amber */
.tc3{background:#E6F1FB;border-color:#185FA5;color:#0C447C}  /* blue */
.tc4{background:#FAECE7;border-color:#993C1D;color:#712B13}  /* coral */
.tc5{background:#F1EFE8;border-color:#5F5E5A;color:#444441}  /* gray */
@media(prefers-color-scheme:dark){
.tc0{background:#26215C;border-color:#AFA9EC;color:#CECBF6}
.tc1{background:#04342C;border-color:#5DCAA5;color:#9FE1CB}
.tc2{background:#412402;border-color:#EF9F27;color:#FAC775}
.tc3{background:#042C53;border-color:#85B7EB;color:#B5D4F4}
.tc4{background:#4A1B0C;border-color:#F0997B;color:#F5C4B3}
.tc5{background:#2C2C2A;border-color:#B4B2A9;color:#D3D1C7}
}
</style>

<div class="ttw">
<div class="ttc">
  <!-- Legend row -->
  <div style="display:flex;flex-wrap:wrap;gap:7px;margin-bottom:8px;...">
    <!-- Color swatches per track -->
    <!-- double-border = key paper, dashed = context precursor -->
  </div>

  <!-- Column headers -->
  <div class="tthdr">
    <div class="ttth tc0">Track 0 Name</div>
    <!-- repeat for each track -->
  </div>

  <!-- Main diagram body -->
  <div class="ttbody" id="ttbody">
    <svg class="ttsvg" id="ttsvg"></svg>  <!-- connectors only -->
    <div class="tttip" id="tttip"></div>  <!-- hover tooltip -->
  </div>
</div>
</div>

<script>
(function(){
  /* ── Layout constants ── */
  const L=64;     // left margin (date labels)
  const TW=149;   // track width (column pitch)
  const NW=134;   // node width
  const NH=42;    // node height

  /* Track color values for SVG strokes (match tc* CSS) */
  const TC=['#7F77DD','#1D9E75','#BA7517','#378ADD','#D85A30','#888780'];

  /* Helper: x center of track t */
  const tx = t => L + t*TW + TW/2;
  /* Helper: x left edge of node in track t */
  const nx = t => L + t*TW + 7;

  /* ── Paper data ──
     Each entry: {
       id:    unique string key
       t:     track index (0–5)
       y:     top pixel position
       title: short display name (≤20 chars)
       sub:   "Author et al. Mon 'YY"
       key:   true = double border
       ctx:   true = dashed border (context precursor)
       desc:  tooltip full description
     }
  */
  const papers = [
    // Example — replace with actual papers:
    {id:'anchor', t:0, y:70,  title:'Anchor Paper',    sub:"Author et al. '24", key:true,  ctx:false, desc:'The paper being surveyed. Describe its core contribution here.'},
    {id:'ext1',   t:0, y:130, title:'Extension 1',     sub:"Author et al. '24", key:false, ctx:false, desc:'First direct extension. What it improves.'},
    {id:'pre1',   t:2, y:48,  title:'Precursor Model', sub:"Author et al. '23", key:false, ctx:true,  desc:'CONTEXT: Foundation model the anchor was applied to. Not a follow-up paper itself.'},
  ];

  /* ── Down edges (sequential within a track) ──
     [parentId, childId] — must share the same track t
  */
  const downEdges = [
    ['anchor','ext1'],
    // ...
  ];

  /* ── Cross edges (influence between tracks) ──
     {f: sourceId, t: targetId}
     Rendered as dashed bezier curves
  */
  const crossEdges = [
    {f:'anchor', t:'pre1'},
    // ...
  ];

  /* ── Date labels (left margin) ──
     {label: string, y: pixel y where the date line sits}
  */
  const dates = [
    {label:"Jul '24", y:88},
    // ...
  ];

  /* ── Build lookup map ── */
  const pmap = {};
  papers.forEach(p => pmap[p.id] = p);

  const body  = document.getElementById('ttbody');
  const svgEl = document.getElementById('ttsvg');
  const tip   = document.getElementById('tttip');

  /* Arrow marker */
  const defs = document.createElementNS('http://www.w3.org/2000/svg','defs');
  defs.innerHTML = `<marker id="arr" viewBox="0 0 10 10" refX="8" refY="5"
    markerWidth="5" markerHeight="5" orient="auto-start-reverse">
    <path d="M2 2L8 5L2 8" fill="none" stroke="context-stroke"
          stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
  </marker>`;
  svgEl.appendChild(defs);

  /* Era band shading — add one rect per period grouping */
  [[238, 328], [410, 560]].forEach(([y1, y2]) => {
    const r = document.createElementNS('http://www.w3.org/2000/svg','rect');
    r.setAttribute('x', 0); r.setAttribute('y', y1);
    r.setAttribute('width', 960); r.setAttribute('height', y2 - y1);
    r.setAttribute('fill', 'var(--color-background-secondary)');
    r.setAttribute('opacity', '0.45');
    svgEl.appendChild(r);
  });

  /* Optional context zone — shaded rect behind precursor papers */
  // const ctxBand = document.createElementNS(...); // add if needed

  /* Date labels */
  dates.forEach(({label, y}) => {
    const d = document.createElement('div');
    d.className = 'ttdl';
    d.style.top = (y - 8) + 'px';
    d.textContent = label;
    body.appendChild(d);
  });

  /* Down edges */
  downEdges.forEach(([fid, tid]) => {
    const fp = pmap[fid], tp = pmap[tid];
    if (!fp || !tp || fp.t !== tp.t) return;
    const x = tx(fp.t);
    const y1 = fp.y + NH, y2 = tp.y;
    if (y2 <= y1) return;
    const line = document.createElementNS('http://www.w3.org/2000/svg','line');
    line.setAttribute('x1', x); line.setAttribute('y1', y1);
    line.setAttribute('x2', x); line.setAttribute('y2', y2);
    line.setAttribute('stroke', TC[fp.t]);
    line.setAttribute('stroke-width', '1');
    line.setAttribute('stroke-opacity', fp.ctx ? '0.35' : '0.5');
    if (fp.ctx) line.setAttribute('stroke-dasharray','3 3');
    line.setAttribute('marker-end','url(#arr)');
    svgEl.appendChild(line);
  });

  /* Cross edges (dashed bezier) */
  crossEdges.forEach(({f: fid, t: tid}) => {
    const fp = pmap[fid], tp = pmap[tid];
    if (!fp || !tp) return;
    const x1 = tx(fp.t), y1 = fp.y + NH/2;
    const x2 = tx(tp.t), y2 = tp.y + NH/2;
    const midY = (y1 + y2) / 2;
    const path = document.createElementNS('http://www.w3.org/2000/svg','path');
    path.setAttribute('d', `M${x1},${y1} C${x1},${midY} ${x2},${midY} ${x2},${y2}`);
    path.setAttribute('fill','none');
    path.setAttribute('stroke', TC[fp.t]);
    path.setAttribute('stroke-width','0.9');
    path.setAttribute('stroke-opacity','0.35');
    path.setAttribute('stroke-dasharray','4 3');
    path.setAttribute('marker-end','url(#arr)');
    svgEl.appendChild(path);
  });

  /* Paper nodes */
  papers.forEach(p => {
    const el = document.createElement('div');
    el.className = `ttn tc${p.t}${p.key ? ' key' : ''}${p.ctx ? ' ctx' : ''}`;
    el.style.left = nx(p.t) + 'px';
    el.style.top  = p.y + 'px';
    el.innerHTML  = `<div class="tnt">${p.title}</div>
                     <div class="tns">${p.sub}</div>`;

    el.addEventListener('mouseenter', () => {
      tip.innerHTML = `
        <div style="font-weight:500;font-size:11px;margin-bottom:3px;
                    color:var(--color-text-primary)">${p.title}</div>
        <div style="color:var(--color-text-tertiary);font-size:9.5px;
                    margin-bottom:4px">${p.sub}</div>
        <div>${p.desc}</div>`;
      tip.style.display = 'block';
      // Flip tooltip to left for rightmost tracks
      let tipLeft = nx(p.t) + NW + 5;
      if (p.t >= 4) tipLeft = nx(p.t) - 192;
      if (tipLeft < 2) tipLeft = 2;
      tip.style.left = tipLeft + 'px';
      tip.style.top  = Math.max(0, p.y - 5) + 'px';
    });
    el.addEventListener('mouseleave', () => { tip.style.display = 'none'; });
    body.appendChild(el);
  });
})();
</script>
```

### Diagram layout rules

**Vertical positioning (y values):**
- Anchor paper: y = 70
- Each subsequent paper in a track: y += 58–80 (more spacing when years are far apart)
- Papers from the same month across different tracks should have the same y
- Context precursors start at y = 48 and stack up before the anchor paper y level

**Era band placement:**
- One band per major time cluster (e.g., Nov–Dec 2024, Apr–Jun 2025)
- Each band spans y1 to y2 covering all papers in that period
- Use `var(--color-background-secondary)` at opacity 0.45

**Total diagram height:**
- Find the paper with the highest y value: `max_y = max(p.y for all p) + NH + 60`
- Set `height:max_y` in `.ttbody` and `height:max_y` in `.ttsvg`

**Track count vs width:**
- 4 tracks → TW = 180, total width = 784px (fits without scroll)
- 5 tracks → TW = 160, total width = 864px (fits without scroll)
- 6 tracks → TW = 149, total width = 958px (needs scroll wrapper at 960px)
- 7+ tracks → split into two diagrams or merge related tracks

**Tooltip side:**
- Tracks 0–3: tooltip appears to the right of the node (`tipLeft = nx(p.t) + NW + 5`)
- Tracks 4–5: tooltip appears to the left of the node (`tipLeft = nx(p.t) - 192`)

---

## Workflow Checklist

Use this checklist when executing the skill:

```
[ ] 1. Identify anchor paper(s) and their core contribution
[ ] 2. Run ≥6 distinct web searches (see query types above)
[ ] 3. For 3D/video topics: add 3 extra domain-specific queries
[ ] 4. Collect papers: title, authors, venue, date, contribution, key/ctx flag
[ ] 5. Assign each paper to a track (4–6 tracks, named by research community)
[ ] 6. Identify 3–5 major trends (≥3 papers each, clear direction)
[ ] 7. Identify 2–4 notable research forks
[ ] 8. Write chronological catalog (see written survey structure)
[ ] 9. Calculate y positions for all nodes (no overlaps within a track)
[ ] 10. Calculate total diagram height
[ ] 11. Build down edges list (sequential within track)
[ ] 12. Build cross edges list (influence between tracks)
[ ] 13. Place date labels at correct y positions
[ ] 14. Add era bands for major time groupings
[ ] 15. Add context zone highlight if precursor papers exist
[ ] 16. Verify tooltip flip logic for rightmost tracks
[ ] 17. Deliver written survey artifact + interactive diagram
```

---

## Domain-Specific Guidance

### For 3D Vision / Reconstruction papers

Always include a "foundation models" context zone at the top of the 3D track:
- These are the offline/quadratic models the TTT-style paper improves on (e.g., DUSt3R, VGGT, CUT3R)
- Mark them `ctx:true` (dashed borders)
- Add a labeled shaded rect: `'foundation models (precursors)'`
- Connect them with dashed down edges

Recommended tracks for 3D/video papers:
- Track 0: Core architecture (the method itself and direct extensions)
- Track 1: Memory/scaling (Google-style memory systems)
- Track 2: Theory (unification frameworks)
- Track 3: Video generation (diffusion-based video)
- Track 4: 3D reconstruction (feed-forward 3D, streaming recon)
- Track 5: Adaptation / downstream apps

### For NLP / LLM papers

Recommended tracks:
- Track 0: Core architecture
- Track 1: Efficiency / hardware
- Track 2: Theory
- Track 3: Reasoning / benchmarks
- Track 4: Domain adaptation
- Track 5: Multimodal applications

### For robotics / embodied AI papers

Recommended tracks:
- Track 0: Core method
- Track 1: Sim-to-real / deployment
- Track 2: Theory / analysis
- Track 3: Manipulation
- Track 4: Navigation / locomotion
- Track 5: Perception / sensing

---

## Quality Standards

**Written survey:**
- Every paper gets a unique one-sentence contribution statement — no two should say the same thing
- Dates must be consistent with arXiv submission dates, not acceptance dates
- Trend names must be action-oriented verbs, not just noun phrases
- Forks must describe two genuine competing directions, not just subfields

**Diagram:**
- No two nodes in the same track should have overlapping y ranges (min 58px gap)
- All cross edges must be bezier curves — no straight diagonal lines
- Solid down edges = sequential development; dashed down edges = loose influence within context zone
- Dashed bezier = cross-track influence; solid bezier = cross-track strong dependency
- Dark mode must be tested: all text must be readable on dark backgrounds via CSS variables
- Tooltips must not overflow the widget boundary (flip logic for tracks ≥4)

**Completeness check before delivering:**
- Does the anchor paper appear as a node? (It always should)
- Are all papers mentioned in the written survey present in the diagram?
- Are cross-track influence arrows consistent with the written trends?
- Is the timeline (y positions) monotonically increasing with date?
