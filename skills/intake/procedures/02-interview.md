# Interview

This procedure captures project requirements from the author.

## Smart Intake — Consuming Existing Docs

If the user provides existing documents (design doc, Google Doc, outline, meeting notes):

1. Read and parse whatever documents the user provides
2. Extract answers to the standard intake questions
3. Normalize into PH format (design.md, module outlines, spec.yaml fields)
4. Present what was found: "I found the following in your docs — does this look right?"
5. Only ask questions for fields that are missing or ambiguous

## Detect Entry Path

First check if `project.description` exists in `publishing-house/spec.yaml` and is non-empty.

**If a description exists**, show it and offer four options:

> I see this project description:
>
> *"{project.description}"*
>
> How would you like to proceed?
>
> 1. Use this description as our starting point
> 2. I have a spec or design doc (file, URL, or paste)
> 3. I have a different idea I want to develop
> 4. I have a Jira issue with requirements

- Option 1 → Path B (Idea), but pre-seeded: treat the description as the user's initial
  answer to "Tell me about your idea." Extract what you can, then follow up on gaps.
- Option 2 → Path A
- Option 3 → Path B (fresh — ignore the description)
- Option 4 → Path C

**If no description exists**, fall back to the original three options:

> How would you like to start?
>
> 1. I have a spec or design doc (file, URL, or paste)
> 2. I have an idea I want to develop
> 3. I have a Jira issue with requirements

## Path A: Full Spec Provided

1. Read the document (file path, pasted content, or URL)
2. Parse against spec template format
3. Identify gaps — missing sections, vague content
4. Ask about each gap ONE at a time
5. Proceed to `procedures/03-design-doc.md`

## Path B: Idea

The user has an idea. Start conversational, get structured later.

**Discover, don't interrogate.** Ask one question at a time. The user's words are the
spec — you are the scribe, not the author. If something is unclear, ask — don't fill it in.

### Opening

Ask ONE open-ended question:

> "Tell me about your idea."

Accept whatever the user provides. Do NOT immediately ask structured questions.

### Extract and Follow Up

After reading the user's description:

1. **Extract what you already know** from the description
2. **Ask targeted follow-ups for what's missing** — one at a time

**Use what the user gives you.** When the user describes specific module content,
use that description — don't substitute your own idea. You are capturing their vision,
not designing a better one.

**Write to spec.yaml immediately.** After each answer, update `publishing-house/spec.yaml`
with the captured fields right away. Do NOT wait until the end of the interview.

**Follow the canonical question list exactly.** Read the intake questions reference file at
`@rhdp-publishing-house/skills/intake/references/intake-questions.md`. Ask each question
using the **exact wording** in that file, **in that exact order**, **one at a time**. Do not
rephrase, merge, reorder, or add questions. Skip any question whose spec.yaml field already
has a value.

## Path C: Jira Issue with Requirements

1. Ask for the Jira issue key or URL
2. Ask the author to paste the relevant requirements from the Jira issue
3. Present what was provided: "Here's what I got from the Jira issue — does this capture it?"
4. Treat extracted requirements like an idea from Path B — follow up on gaps

After all questions are answered, proceed to `procedures/03-design-doc.md`.
