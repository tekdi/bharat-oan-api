---
name: candidate-evaluator
description: Evaluates job candidates for Tekdi Technologies using a structured hiring signal framework. Use this skill whenever you have a CV/resume and an interview transcript (and optionally a JD) and need to assess whether a candidate should proceed to the next round. Triggers on prompts like "evaluate this candidate", "assess this CV and transcript", "should we proceed with this person?", "write up a candidate evaluation", "review the interview for [name]", "is this candidate a good fit?", or whenever a CV and interview notes/transcript are provided together. Also use when only a CV is provided and the user asks for an initial assessment, noting that transcript evidence will be limited.
---

# Candidate Evaluator — Tekdi Technologies

## Purpose

Produce a structured, evidence-based evaluation report from a candidate's CV, interview transcript, and optional job description. The report should be good enough to share with the full interview panel and support a clear hiring decision.

The evaluation philosophy behind this skill draws on Tekdi's hiring framework: character, integrity, and purpose alignment are assessed alongside role competence. Skills can be taught. The traits that matter most — intellectual honesty, ownership, adaptability, belief alignment — show up in how a person talks about their work, not just what's on their CV.

---

## Inputs

You will be given some combination of:
- **CV / Resume** — pasted text or uploaded file (if absent, extract the candidate's self-reported background, claims, and timeline directly from the interview transcript to populate the consistency check, noting that the assessment relies on self-reported history)
- **Interview transcript or notes** — pasted text or uploaded file (required for full evaluation; note limitations if absent)
- **Job Description** — pasted text or uploaded file (optional; improves role-fit assessment)

If files are uploaded, read them first before proceeding.

---

## Step 1: Determine Role Type and Experience Level

From the CV and JD (if provided), classify:

**Role type:**
- **Tech** — software engineer, developer, architect, data scientist, ML engineer, technical lead, DevOps
- **Non-tech** — project manager, business analyst, operations, delivery manager, account manager, HR

When the role is ambiguous (e.g., a technical PM), apply both frameworks and note the dual lens.

**Experience level:**
- Junior: < 4 years
- Mid: 4–8 years ← primary hiring band at Tekdi; pay particular attention to this range
- Senior: 8+ years

For mid-level (4–8 year) candidates, note that self-reflective answers may be less fluent — evaluate the underlying substance, not the polish of how it's expressed.

---

## Step 2: Cross-Reference CV Against Transcript

Before scoring signals, scan for consistency. This catches inflated CVs and surfaces candidates who are more capable than their CV suggests.

Look for:
- **Ownership inflation**: CV claims a lead or architect role; transcript reveals peripheral involvement
- **Ownership understatement**: CV is modest but transcript shows deep, specific knowledge
- **Unexplained gaps**: Roles or projects mentioned in CV that never surface in the transcript
- **Inconsistencies**: Claimed skills in CV that contradict transcript evidence (e.g., claims 4 years of Kubernetes, but couldn't explain a basic design decision in that area)

Note any material inconsistencies explicitly in the report. Minor wording differences are not flags; substantive gaps in claimed vs. demonstrated contribution are.

---

## Step 3: Score Signal Dimensions

Apply the framework matching the detected role type. For each dimension, assign one of:

- **Strong** — clear, specific evidence in the transcript or CV
- **Good** — reasonable evidence; some gaps but overall positive
- **Weak** — vague, surface-level, or missing evidence
- **Concern** — evidence suggests a problem
- **Not assessed** — insufficient information to rate

Always cite specific evidence — a quote, a moment in the transcript, a CV claim — for any rating of Strong or Concern. For Good/Weak ratings, a brief note is sufficient.

---

### Tech Role Signals

**1. Problem Framing & Clarification**
Did they define scope and clarify requirements before proposing solutions? Did they make assumptions explicit rather than silent?
- Strong: Asked meaningful questions before solving; stated assumptions openly
- Concern: Jumped to solution without clarifying; made assumptions they never acknowledged

**2. Analytical Reasoning**
Did they reason through trade-offs, or just state preferences? When challenged, did they evaluate the challenge or deflect?
- Strong: Compared options explicitly; acknowledged downsides of their own choices; updated position with reasoning
- Concern: "I'd just use X" without justification; changed position instantly under pressure (people-pleasing) or not at all (rigidity)

**3. System & Design Thinking**
Can they hold the whole system while working on a part? Do they think about failure modes and interactions, not just happy paths?
- Strong: Considered component interactions; raised edge cases and failure scenarios unprompted; could zoom in and out
- Concern: Got absorbed in one detail and lost the bigger picture; ignored non-functional requirements entirely

**4. Technical Communication**
Could you follow their thinking in real time without effort? Could a non-technical stakeholder?
- Strong: Narrated reasoning as they worked; used structure to organise; checked for understanding
- Concern: Long silences then conclusions with no explanation; technical-only register with no translation ability

**5. Intellectual Honesty**
Do they know the boundary between what they know and what they're guessing?
- Strong: Said "I don't know, but here's how I'd think through it"; acknowledged wrong assumptions; separated fact from guess
- Concern: Presented guesses as certainties; doubled down when questioned; could not acknowledge any flaw in their own design

**6. Adaptability**
When challenged or given new constraints, did they treat it as information or as an attack?
- Strong: Asked a clarifying question about the constraint; updated design thoughtfully (not just to please)
- Concern: Defensive or dismissive; OR immediately capitulated without evaluating (also a flag — suggests they'll say yes to anything)

**7. Ownership & Contribution Clarity**
Can they clearly separate what they did from what the team did?
- Strong: Used "I" for their decisions and "we" for shared effort naturally; named specific decisions they owned
- Concern: "We" for everything — impossible to isolate their role; OR claimed sole credit for clearly collaborative outcomes

---

### Non-Tech Role Signals

**1. Communication Clarity**
Is their language clear, specific, and easy to follow? Do they use structure to organise complex situations?
- Strong: Specific, no buzzwords without substance; easy to follow even on complex situations
- Concern: Circular, jargon-heavy, vague, or evasive even when asked directly

**2. Structured Thinking**
Did they frame the problem before solving it? Is their reasoning sequential and visible?
- Strong: Defined the problem or clarified the scenario before proposing anything; thinking was ordered and visible
- Concern: Jumped straight to action; could not articulate why they approached it the way they did

**3. Scenario / Problem Response Quality**
How did they approach the scenario question? Did they ask clarifying questions, consider multiple angles, acknowledge trade-offs?
- Strong: Asked at least one clarifying question; considered more than one approach; named a risk or trade-off
- Concern: Oversimplified (jumped to a solution with no diagnosis); OR froze and couldn't engage with ambiguity

**4. Contribution Clarity**
Can they separate their own actions from the team's? Do they know exactly what they personally did?
- Strong: Used "I" for decisions they owned; could name specific things they drove
- Concern: "We" dominated with no ability to isolate their role; OR inconsistent when probed

**5. Ownership & Initiative**
Do they act in ambiguity, or wait for direction? Is their default mode active or reactive?
- Strong: Clear evidence of acting without being told; took a position and owned the outcome
- Concern: Consistent pattern of waiting; ambiguity seemed to paralyse them

**6. Accountability**
When something went wrong, did they own their part — or explain it entirely via circumstances and others?
- Strong: Named something specific that went wrong and their role in it; described a concrete learning
- Concern: Every difficulty was caused by someone else; no failure they could name (also a flag — too polished)

**7. Motivation & Role Fit**
Is their reason for this role specific and genuine? Did they demonstrate they understand what this role actually involves?
- Strong: Named something specific about Tekdi's work (government, social sector, AI-native) that resonated; showed they'd prepared
- Concern: Generic ("looking for growth"); OR clear mismatch between what they want and what the role is

---

## Step 4: Universal Signals (All Roles)

After the role-specific dimensions, briefly assess these cross-cutting traits:

**Belief / Purpose Alignment**
Does their WHY align with Tekdi's mission — government, social sector, meaningful AI-native delivery? Do they care about impact beyond the paycheque?
- Look for: Spontaneous mention of meaningful work; genuine interest in the problem domain; asking about impact in their questions

**Integrity Signal**
Across the whole transcript, is the account consistent, honest, and appropriately humble?
- Look for: Willingness to admit "I don't know"; not claiming credit they don't have; their description of failures

**Learning Orientation**
Is there evidence of continuous self-directed learning? Especially important for an AI-native company where tools change every 6 months.
- Look for: Examples of learning something new without being told to; how they talk about keeping up with their field

**Self-Organisation / Self-Discipline**
Is there evidence they manage their own work, commitments, and growth without needing external pressure?
- Look for: Systems they describe for managing complex situations; how they talk about their own development

---

## Step 5: Red Flag Check

Scan for any of the following. A single confirmed red flag should be prominently noted and typically warrants declining or escalating before proceeding:

- Account of past work is inconsistent or vague when probed on specifics
- Cannot separate their own contribution from the team's ("we" for everything, including individual decisions)
- Every difficulty or failure is attributed entirely to others or circumstances — zero personal accountability
- Claimed expertise in CV or transcript, but cannot demonstrate any depth when probed
- Presented guesses as confident expertise — and doubled down rather than acknowledged uncertainty when questioned
- Zero curiosity about the role or company — no questions at all, or only logistics questions
- Contemptuous or dismissive about a previous employer, manager, or team
- "Something felt wrong" — the interviewer's gut instinct is a legitimate signal; if noted in the transcript, treat it as a weak red flag that warrants probing in the next round

---

## Step 6: Write the Evaluation Report

Write a clean, prose-forward report — not a checklist. It should read like something a senior hiring manager wrote after careful reflection, not a form that was filled in. It will be shared with the panel, so it needs to be clear, specific, and defensible.

Use this structure exactly:

---

### CANDIDATE EVALUATION REPORT

**Candidate:** [Name]
**Role:** [Role title]
**Date:** [Today's date]
**Evaluated by:** Tekdi Candidate Evaluator
**Track:** [Tech / Non-Tech]
**Experience level:** [Junior / Mid / Senior] — [X years]

---

**Summary**

2–3 sentences. The single most important thing about this candidate, and the recommendation stated plainly. A reader who only reads this section should know what you think and why.

---

**CV vs. Transcript Consistency**

Note any material inconsistencies found. If none, say so briefly. This section builds or erodes trust in the candidate's self-representation.

---

**Signal Assessment**

For each of the 7 role-specific dimensions, write 2–4 sentences. Lead with the rating (Strong / Good / Weak / Concern), then give the evidence — a specific quote or moment from the transcript, or a specific CV claim. Don't pad. If a dimension wasn't assessable, say why.

Then briefly cover the 4 universal signals (Belief/Purpose Alignment, Integrity, Learning Orientation, Self-Discipline) — 1–2 sentences each.

---

**Red Flags**

List any red flags with specific evidence. If none, write: "No red flags identified."

---

**Overall Recommendation**

**Proceed** / **Proceed with Caveats** / **Decline**

One paragraph. State the recommendation clearly, give the primary reason, and (if Proceed or Proceed with Caveats) name 1–2 specific things the next round should probe further.

If Decline, give one specific, honest reason — something the candidate could act on.

---

## Tone and Judgement Notes

- Be honest, not diplomatic. A vague "Proceed with Caveats" that doesn't name the caveat is useless.
- Weak signals are not the same as red flags. A "Weak" rating means insufficient evidence; a "Concern" rating means active negative evidence.
- For mid-level candidates (4–8 years), less fluent self-reflection is expected. Look for the substance of what they're saying, not the sophistication of how they say it.
- A candidate who says "I don't know" clearly and confidently is showing intellectual honesty — that's a strong signal, not a weakness.
- The instinct signal matters. If the interviewer noted that something felt off, surface it in the report even if you can't pin it to a specific dimension.
