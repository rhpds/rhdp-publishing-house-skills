# RCARS Vetting

Phase 3 of the intake flow. Validate the design against the existing RHDP catalog.

**If offline → skip this phase entirely with a warning:**
> "RCARS vetting is unavailable offline. This will run when you submit for review."

## Query RCARS

Call Central's RCARS advisor with the project slug:

```bash
python publishing-house/tools/ph-rcars.py submit "QUERY"
```

Build the QUERY from design.md — summarize the project in one sentence using the title,
products, audience, and key learning objectives:

`"A [audience] [content_type] covering [products] that teaches [key objectives from design.md]"`

Extract `job_id` from the output. If empty or failed → skip RCARS with a note:
> "RCARS advisor query failed. Proceeding without vetting — this will run at submission."

## Poll for Results

```bash
python publishing-house/tools/ph-rcars.py poll JOB_ID
```

If `status` is `running` or `queued`, wait 5 seconds and poll again. Keep polling every
5 seconds for up to 90 seconds total. The advisor typically takes 10–20 seconds but can
be slower under load.

## Present Results

**If advisor returns candidates** (status=complete, candidates non-empty):

> "I checked the RHDP catalog against your design. Here's what the advisor found:
>
> 1. **[display_name]** ([relevance_score]% relevance)
>    *Similar because:* [why_it_fits — first sentence]
>    *Gap:* [caveats — first sentence]
>
> 2. **[display_name]** ([relevance_score]% relevance)
>    *Similar because:* [why_it_fits — first sentence]
>
> **How does your lab specifically differ from these?** Do you want to adjust the design
> based on what the catalog already covers, or does this confirm your direction?"

**If no candidates** (empty list or all relevance_score < 50):

> "I checked the RHDP catalog — no close matches found. This looks like new territory.
> Moving on to module outlines."

No question needed — the differentiation is self-evident if nothing similar exists.

**If advisor failed or timed out:**

> "RCARS advisor wasn't available this run. We'll skip vetting for now — this will
> run again at submission. Moving on to module outlines."

Do NOT ask the author to explain differentiation when RCARS fails. That's a platform
problem, not the author's problem. The differentiation field will be populated when
RCARS is available (either in a later session or at submission).

## Handle Adjustments

If the author wants to adjust the design based on RCARS findings:
1. Update `publishing-house/spec/design.md` with the changes
2. Re-run the inline structure check from Phase 2
3. The author can also edit design.md directly and commit

If no adjustments → proceed.

## Write Point

Write RCARS results and differentiation to spec.yaml:

```yaml
approval_checklist:
  content:
    rcars_overlap_pct: [highest relevance_score or null]
    rcars_top_matches:
      - title: "[display_name]"
        ci_name: "[ci_name]"
        url: "https://catalog.demo.redhat.com/catalog?item=[ci_name]"
        relevance_score: [score]
        why_it_fits: "[why_it_fits]"
    differentiation: "[author's response about what makes this unique]"
```

```bash
git add publishing-house/spec.yaml publishing-house/spec/design.md
git diff --cached --quiet || git commit -m "feat: phase 3 — RCARS vetting complete" 2>/dev/null || true
```

Proceed to Phase 4 (module outlines).
