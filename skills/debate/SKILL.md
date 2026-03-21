---
description: Run a debate between two AI agents to refine anything — text, code, concepts, architecture
argument-hint: [--quick] [--rounds N] [--model opus|sonnet|haiku] [--no-steer] "topic or instruction" [@file]
---

You are the orchestrator for a two-agent adversarial collaboration. Follow these steps precisely and in order.

## Step 1: Parse Arguments

Check the value of $ARGUMENTS. If it is empty, blank, or contains no meaningful prompt, respond with exactly this message and then STOP (do not proceed to any further steps):

"You need to provide direction for the debate. Examples:
- `/debate --quick 'tighten this up' @draft.md` — single pass, lowest cost
- `/debate 'sharpen for execs' @report.md` — 2-round debate (default)
- `/debate --rounds 5 'stress test this argument' @proposal.md` — extended debate
- `/debate 'best approach to caching in this system'` — topic debate (no file)"

If $ARGUMENTS contains a prompt, continue parsing:

1. Check if $ARGUMENTS contains "--quick".
   - If found: set `quick_mode` to true and remove the `--quick` portion from the text.
   - If not found: set `quick_mode` to false.
2. Check if $ARGUMENTS contains "--rounds" followed by a number (e.g., `--rounds 5`) or "--rounds=N" (e.g., `--rounds=5`).
   - If found: extract that number as `total_rounds` and remove the `--rounds N` (or `--rounds=N`) portion from the text.
   - If not found: set `total_rounds` to 2.
3. If `total_rounds` is less than 1 or is not a valid number: set `total_rounds` to 2.
4. If `total_rounds` is greater than 10: display this message and wait for the user's response:
   "That is a LOT of rounds. Type exactly RIP MY TOKENS to confirm, or I will cap at 10."
   - If the user's response is exactly "RIP MY TOKENS" (case insensitive): keep `total_rounds` as requested.
   - Otherwise: set `total_rounds` to 10.
5. Check if $ARGUMENTS contains "--model" followed by a value (e.g., `--model sonnet`) or "--model=VALUE" (e.g., `--model=haiku`).
   - If found: extract the value as `agent_model` and remove the `--model VALUE` (or `--model=VALUE`) portion from the text.
   - Valid values are: `opus`, `sonnet`, `haiku` (case insensitive).
   - If the value is not one of these three, display: "Invalid model. Choose from: opus, sonnet, haiku. Defaulting to sonnet." and set `agent_model` to "sonnet".
   - If not found: set `agent_model` to "sonnet".
6. The remaining text after removing all flags is `user_prompt`.
7. Check if $ARGUMENTS contains "--no-steer".
   - If found: set `steering_enabled` to false and remove the flag from the text.
   - If not found: set `steering_enabled` to true.

## Step 2: Identify Inputs

1. Extract the user's text prompt from `user_prompt` (parsed in Step 1). This is the direction for the debate.
2. Check whether any file content has been provided via @file references in the conversation context. If file content is present, that is the source material to work with. If no file content is present, the agents will debate the topic or question described in the prompt itself.
3. Note the filename of any @file reference for later use. This is the `original_filename`. If no file was provided, set `original_filename` to "none".
4. Determine the `input_type` — what kind of material this is. Infer from the file extension, content, and prompt. Common types: "prose" (reports, essays, docs), "code" (source files, scripts, configs), "concept" (ideas, architecture, strategy — no file). This label is passed to agents so they adapt their approach.

Store these as:
- `user_prompt`: The user's direction (with flags already removed)
- `subject`: The file content (if @file provided), or the topic/question itself (if no file)
- `original_filename`: The filename from any @file reference (or "none" if topic debate)
- `input_type`: "prose", "code", or "concept"

## Quick Mode Branch

If `quick_mode` is true, jump to **Step Q** now. Skip all remaining steps except Step Q and Step 11.

## Step 3: Select Agent Names

Select 1 name at random from the MAXIMISER list for Agent A, and 1 name at random from the SKEPTIC list for Agent B. Do not reuse names.

MAXIMISER NAMES:
Aristotle, Cicero, Leonardo da Vinci, Giordano Bruno, Pythagoras, Archimedes, Pericles, Lorenzo de' Medici, Marsilio Ficino, Leon Battista Alberti, Seneca, Plutarch, Heraclitus, Pico della Mirandola, Vitruvius

SKEPTIC NAMES:
Socrates, Diogenes, Sextus Empiricus, Niccolò Machiavelli, Galen, Epicurus, Thucydides, Hypatia, Lucretius, Zeno of Elea, Marcus Aurelius, Tacitus, William of Ockham, Pietro Pomponazzi, Protagoras

## Step 4: Initialize State

Set up the following state variables:
- `subject`: Copy of subject from Step 2 (persists — both agents need the original)
- `current_draft`: Set to subject (will become the working draft)
- `input_type`: From Step 2
- `criteria`: Empty (set in Step 5)
- `criteria_a`: Empty (Maximiser's owned criteria)
- `criteria_b`: Empty (Skeptic's owned criteria)
- `choice_points`: Empty (set in Step 5)
- `version_a`: Empty (set in Step 6)
- `version_b`: Empty (set in Step 6)
- `version_a_summary`: Empty
- `version_b_summary`: Empty
- `disagreements`: Empty (set in Step 7)
- `resolution_log`: Empty list
- `current_round`: 1
- `original_filename`: From Step 2
- `agent_model`: From Step 1 (default "sonnet")
- `steering_enabled`: From Step 1 (default true)
- `first_resolver`: Randomly select as either `maximiser` or `skeptic` with equal probability. (Same randomization approach used for name selection.)

## Agent Model Selection

**All sub-agents spawned in Steps 6 and 8 MUST use `agent_model` as the `model` parameter on the Agent tool call.** For example, if `agent_model` is "haiku", every Agent tool invocation must include `model: "haiku"`. This applies to version generation and resolution agents alike.

## Display Guidelines

All visual formatting below uses **plain text output** (not code blocks unless showing the final version). Use the exact formatting shown. Use plain `---` separators between sections. No emoji in section headers.

Avoid box-drawing characters (│┌┐└┘├┤) around lines that contain variable-length content — alignment is fragile.

## Step 4b: Opening Display

After initializing state, display this single status line:

```
**DEBATE** | Model: {agent_model} | Rounds: {total_rounds} | Type: {input_type}
```

**Concept mode tip:** If `input_type` is "concept", display immediately after:
```
TIP: Concept debates work best with reference material.
Try providing a file: /debate "your topic" @notes.md
```

## Step 5: Define & Assign Success Criteria

Display:
```
---
ESTABLISHING SUCCESS CRITERIA
---
```

You (the orchestrator) define 4-6 success criteria yourself — do NOT spawn a sub-agent for this. Analyze the user's direction and input material, then assign each criterion to the appropriate role.

The two roles should create productive tension appropriate to the input type:
- For PROSE: Maximiser owns impact, boldness, and engagement. Skeptic owns accuracy, evidence, and credibility.
- For CODE: Maximiser owns elegance, expressiveness, and design. Skeptic owns correctness, edge cases, and robustness.
- For CONCEPTS: Maximiser owns ambition, differentiation, and vision. Skeptic owns feasibility, risks, and assumptions.

Adapt to the specific material — these are starting points.

**Push beyond obvious edits.** At least one criterion per role should target something a straightforward single-pass editor would likely miss. For the Maximiser, this might be a structural reorganization, a reframing of the core argument, or an unexploited opportunity in the material. For the Skeptic, this might be an unstated assumption that weakens the argument, a logical gap, or a risk the material glosses over. The goal is to ensure the debate produces insights that justify the adversarial process — if both agents would make the same changes as a solo editor, the debate adds cost without value.

**For each criterion, identify one concrete choice point** — a specific place in the source material where the two roles would make different decisions. This forces the debate to be about real content, not abstract principles.

Output in EXACTLY this format and display to the user:

```
MAXIMISER:
- [Criterion]: [One sentence definition]
- [Criterion]: [One sentence definition]
SKEPTIC:
- [Criterion]: [One sentence definition]
- [Criterion]: [One sentence definition]
```

Note: Choice points and rationale are generated internally but NOT displayed to the user. They are stored for use in agent prompts (Steps 6 and 8).

Store `criteria` (full list), `criteria_a` (Maximiser's criteria with choice points), `criteria_b` (Skeptic's criteria with choice points), `choice_points` (all choice points collected).

## Step 5b: Introduce Agents

After displaying criteria, introduce the agents as single lines showing what criteria each owns:

```
MAXIMISER — {Agent A name} — owns: {comma-separated list of criteria_a names}
SKEPTIC — {Agent B name} — owns: {comma-separated list of criteria_b names}
```

## Step 6: Generate Independent Versions

Display:
```
---
INDEPENDENT DRAFTS
---
```

Display both stage announcements:
```
{Agent A name} (Maximiser) takes the stage...
{Agent B name} (Skeptic) takes the stage...
```

**Spawn BOTH agents simultaneously using two parallel Agent tool calls in a single response.** These agents are explicitly independent and must run in parallel.

### Agent A prompt (Maximiser)

```
{Agent A name}, Maximiser — produce an opinionated version pushing the material further on your owned criteria.

USER'S DIRECTION: {user_prompt}
YOUR CRITERIA: {criteria_a}
CHOICE POINTS (where you MUST make a different decision than a conservative editor would):
{choice_points for criteria_a}

SUBJECT: {subject}

THE OTHER AGENT (Skeptic) will focus on: {criteria_b names, comma-separated}. They will likely tighten, add caveats, and shore up weaknesses. You should NOT do the same things they will — instead, push in your own direction: restructure, reframe, find the unexploited opportunity in this material that a cautious editor would never touch.

Do NOT make obvious improvements that any competent editor would make (fixing typos, basic cleanup, surface-level tightening). Those will happen regardless. Instead, make changes that ONLY a Maximiser would make — changes that require boldness, vision, or willingness to break the existing structure.

BLIND SPOT: Identify one unexploited opportunity in the material — something valuable that the author left on the table. Build your version around exploiting it.

You MUST make at least 2 changes that the other agent will disagree with. (Match length to material complexity.)

Respond in EXACTLY this format:

BLIND SPOT: [The unexploited opportunity you identified]
RATIONALE: [2-3 sentences on your philosophy and key choices]
YOUR VERSION: [Complete improved version — full document, full code, or full response.]
```

### Agent B prompt (Skeptic)

```
{Agent B name}, Skeptic — produce an opinionated version tightening the material on your owned criteria.

USER'S DIRECTION: {user_prompt}
YOUR CRITERIA: {criteria_b}
CHOICE POINTS (where you MUST make a different decision than a bold editor would):
{choice_points for criteria_b}

SUBJECT: {subject}

THE OTHER AGENT (Maximiser) will focus on: {criteria_a names, comma-separated}. They will likely restructure, reframe, and push for bolder claims. You should NOT do the same things they will — instead, push in your own direction: find the hidden weaknesses, challenge unstated assumptions, and demand evidence where the material hand-waves.

Do NOT make obvious improvements that any competent editor would make (fixing typos, basic cleanup, surface-level tightening). Those will happen regardless. Instead, make changes that ONLY a Skeptic would make — changes that require intellectual rigor, honesty about weaknesses, or willingness to cut claims that sound good but lack support.

BLIND SPOT: Identify one hidden weakness or unstated assumption in the material — something the author is hoping nobody will notice or question. Build your version around addressing it.

You MUST make at least 2 changes that the other agent will disagree with. (Match length to material complexity.)

Respond in EXACTLY this format:

BLIND SPOT: [The hidden weakness or unstated assumption you identified]
RATIONALE: [2-3 sentences on your philosophy and key choices]
YOUR VERSION: [Complete improved version — full document, full code, or full response.]
```

**Concept mode divergence:** If `input_type` is "concept", add one additional line to each agent prompt before the SUBJECT line:
- Maximiser: "This is a concept debate. You MUST commit to a clear position and argue it fully — do not hedge or present both sides. Pick the bolder, more ambitious option and make the strongest possible case for it."
- Skeptic: "This is a concept debate. You MUST commit to the OPPOSITE position from the one the Maximiser would likely take. Argue it fully and honestly — do not just be cautious about the same recommendation. Pick the more conservative or contrarian option and make the strongest possible case for it."

Wait for both sub-agents to complete. Extract YOUR VERSION from each → `version_a` and `version_b`. Extract BLIND SPOT and RATIONALE from each → `version_a_summary` and `version_b_summary` (combine blind spot + rationale as the summary). After extracting, the full agent response text is no longer needed.

## Step 7: Surface Disagreements

Display:
```
---
DISAGREEMENTS
---
```

You (the orchestrator) compare `version_a` and `version_b` directly — do NOT spawn a sub-agent for this. Produce a numbered list of 3-7 disagreements in this format:

```
For each disagreement:
- POINT: What the disagreement is about
- {Agent A name} (Maximiser): What they chose [quote/paraphrase]
- {Agent B name} (Skeptic): What they chose [quote/paraphrase]

(blank line between each disagreement)
```

Store as `disagreements`. Display the full disagreements list to the user.

After displaying: do NOT set `current_draft` to either version. Both `version_a` and `version_b` are carried forward as equal inputs to the first resolution round. Do not discard full version text yet — Round 1 needs both.

## Step 8: Resolution-Approval Loop

Loop from 1 to `total_rounds`. After completing all rounds or receiving approval, exit the loop and proceed to Step 9.

**Termination check:** Before starting each round, verify that `current_round` is less than or equal to `total_rounds`. If `current_round` > `total_rounds`, STOP the loop and proceed to Step 9.

Alternate which agent resolves each round. In odd rounds (1, 3, 5…) `first_resolver` resolves. In even rounds (2, 4, 6…) the other agent resolves. The opposing agent cross-reviews after the resolver completes.

**Resolution log capping:** Before injecting `resolution_log` into an agent prompt, apply this rule: for round 1, pass the full log. For round 2+, summarize all rounds except the most recent into a single line each (e.g., "Round 1: Maximiser resolved — chose bold framing. Rejected: weak evidence.") and include full detail only for the most recent round.

Display before the first round of the loop:
```
---
RESOLUTION ROUNDS
---
```

### Resolve + Cross-Review (two agents per round)

Build a visual round header. Use filled blocks (█) to show progress and empty blocks (░) to show remaining rounds. For example, round 2 of 5 would show `██░░░`. Display:

```
-- ROUND {current_round} of {total_rounds}  {progress_bar} --
   {resolving agent name} resolves, {reviewing agent name} reviews
```

#### Resolver agent

Spawn a sub-agent using the Agent tool. The prompt varies by round:

**Round 1 prompt** (merging two equal versions):

```
{resolving agent name}, {Maximiser|Skeptic} — merge two independently-produced versions into a unified draft. Your conviction is ABSOLUTE. When evidence genuinely favors the other side, concede GRUDGINGLY as tactical retreat.

Neither version is the base — treat both as equal inputs.

YOUR CRITERIA: {criteria_a or criteria_b}

DISAGREEMENTS: {disagreements}
{Agent A name}'s approach: {version_a_summary}
{Agent B name}'s approach: {version_b_summary}

VERSION A ({Agent A name}, Maximiser):
{version_a}

VERSION B ({Agent B name}, Skeptic):
{version_b}

RESOLUTION LOG: {resolution_log}

(Match length to material complexity.)

Respond in EXACTLY this format:

RESOLUTIONS:
[For each disagreement, label as CHOSE A, CHOSE B, or SYNTHESIZED, then explain why in 1-2 lines. For at least 1 disagreement, you MUST propose a SYNTHESIZED option — a third approach that neither agent considered, combining their insights into something better than either version alone. Do not just alternate between picking A and B.]

UNIFIED DRAFT: [Complete merged version — full document, full code, or full response.]
```

**Round 2+ prompt** (iterating on current draft):

```
{resolving agent name}, {Maximiser|Skeptic} — resolve disagreements in the current draft. Your conviction is ABSOLUTE. When evidence genuinely favors the other side, concede GRUDGINGLY as tactical retreat.

YOUR CRITERIA: {criteria_a or criteria_b}

DISAGREEMENTS: {disagreements}
{Agent A name}'s approach: {version_a_summary}
{Agent B name}'s approach: {version_b_summary}

CURRENT DRAFT: {current_draft}
RESOLUTION LOG: {resolution_log}

If the resolution log contains USER STEERING notes, treat those as highest-priority direction — they override agent preferences.

(Match length to material complexity.)

Respond in EXACTLY this format:

RESOLUTIONS:
[For each disagreement, label as CHOSE A, CHOSE B, or SYNTHESIZED, then explain why in 1-2 lines. For at least 1 disagreement, you MUST propose a SYNTHESIZED option — a third approach that neither agent considered. Do not just alternate between picking A and B.]

UNIFIED DRAFT: [Complete merged version — full document, full code, or full response.]
```

Wait for the resolver sub-agent to complete. Extract the UNIFIED DRAFT → `current_draft`. Append RESOLUTIONS to `resolution_log`.

After Round 1 completes: set `current_draft` to the unified output and discard the full text of `version_a` and `version_b`. Only `version_a_summary`, `version_b_summary`, and `disagreements` are carried forward.

#### Cross-review agent

After the resolver completes, spawn a lightweight cross-review sub-agent — the **opposing** agent (the one who did NOT resolve this round). This agent reviews but does NOT produce a new version.

```
{reviewing agent name}, {Maximiser|Skeptic} — review the unified draft produced by {resolving agent name}.

Assess whether this draft adequately serves YOUR criteria. You did NOT write this — the other agent did. Be suspicious but honest.

YOUR CRITERIA: {criteria_b or criteria_a}
UNIFIED DRAFT: {current_draft}

Check specifically:
- Did the resolver WATER DOWN your criteria's contributions? If your best ideas were softened or dropped, that is grounds for rejection.
- Did the resolver just pick the other agent's version for your disagreements without genuine engagement? That is also grounds for rejection.
- Does the draft contain at least one SYNTHESIZED resolution (a third option beyond what either agent proposed)? If every resolution is just picking a side, push back.

VERDICT: [APPROVE or REJECT]
ASSESSMENT: [2-3 sentences. You MUST identify at least one specific remaining weakness even if approving — no draft is perfect. If rejecting, state SPECIFIC objections.]
REMAINING WEAKNESS: [One concrete thing that could still be better, even if you are approving overall.]
```

Wait for the cross-review agent to complete. Check the VERDICT:

- **APPROVE** → Display `APPROVED by {reviewing agent name}` and append the REMAINING WEAKNESS to `resolution_log` as context (even though the loop exits). Then exit the loop immediately and proceed to Step 9.
- **REJECT** → Display `REJECTED by {reviewing agent name} — back to the ring` and append the ASSESSMENT (objections) to `resolution_log`.

  **Steering checkpoint:** If `steering_enabled` is true AND `current_round` is less than `total_rounds`:

  Display:
  `Steering? Enter notes to redirect, or press Enter to continue.`

  Wait for user input.
  - If the user provides non-empty text: append to `resolution_log` as
    "USER STEERING (after round {current_round}): {user's text}".
    Display: `Noted — steering applied for round {current_round + 1}.`
  - If the user presses Enter with no text (empty response): continue without modification.

  Increment `current_round` by 1, continue the loop

If all rounds are exhausted without approval, proceed to Step 9 anyway. Note in the journey summary that full consensus was not reached.

## Step 9: Journey Summary

After the loop completes, synthesize a 3-5 line journey summary from the `resolution_log`. You (the orchestrator) write this directly — do NOT spawn a sub-agent for this.

The journey summary should:
- Reference both agents by name and role (Maximiser/Skeptic)
- Mention the criteria that guided decisions
- Highlight the key tension between Maximiser and Skeptic approaches
- Note whether consensus was reached or not (and on which round if early exit)
- Describe how the output evolved

Display the journey summary in this format:

```
---
THE JOURNEY
---

{3-5 line narrative synthesized from resolution_log, referencing agents by name and role, highlighting key tensions and whether consensus was reached}
```

## Step 9b: Tradeoffs

After the journey summary, you (the orchestrator) identify the key tradeoffs made during the debate.
Do NOT spawn a sub-agent for this. Review the resolution_log and compare the original material
with current_draft.

For each tradeoff (2-5 items), state what was gained and what was sacrificed or risked.
Be honest — not everything improved. If the debate pushed hard on one criterion, another likely softened.

Display:

```
---
TRADEOFFS
---

- GAINED: {what improved} / COST: {what was sacrificed or risked} (criteria: {relevant criteria})
- GAINED: {what improved} / COST: {what was sacrificed or risked} (criteria: {relevant criteria})
- ...
```

If you cannot identify a genuine cost for a gain, do not list it as a tradeoff —
list it only in WHAT CHANGED (Step 9c).

If no genuine tradeoffs exist (all changes were unambiguous improvements), display:
"No significant tradeoffs — changes were unambiguous improvements."

## Step 9c: Semantic Diff Summary

After the tradeoffs section, you (the orchestrator) produce 3-7 bullets describing what specifically changed between the original material (`subject`) and the final version (`current_draft`). Do NOT spawn a sub-agent for this.

Each bullet names the change and links it to the relevant criterion from Step 5. This is not a line-by-line diff — it is a human-readable summary of material improvements.

Display:

```
---
WHAT CHANGED
---

- {Change description} (criterion: {relevant criterion name})
- {Change description} (criterion: {relevant criterion name})
- ...
```

## Step 10: Present Final Output

After the semantic diff summary, display the final version:

```
--- FINAL VERSION ---
```

{current_draft}

## Step 11: Save Options (file debates only)

IF original_filename is "none" (topic debate, no source file):
  Do NOT display any save options. STOP.

IF original_filename is not "none" (a source file was provided):

Generate `auto_filename` by inserting "-refined" before the file extension of original_filename. For example: `report.md` becomes `report-refined.md`, `essay.txt` becomes `essay-refined.txt`. If the filename has no extension, append `-refined` to the end (e.g., `README` becomes `README-refined`).

Display this prompt and wait for the user's response:

```
What would you like to do with the result?

1. Overwrite original file ({original_filename})
2. Save to new file (suggested: {auto_filename})
3. Print to screen (already displayed above)
```

**If the user chooses 1 (overwrite original):**
  Use the Write tool to write current_draft to the original file path (original_filename).
  Display: "Saved to {original_filename}"
  STOP. Do not offer to re-run or ask any follow-up questions.

**If the user chooses 2 (Save to new file):**
  Display the suggested filename ({auto_filename}) and ask the user if they want to use it or provide their own filename.
  Use the Write tool to write current_draft to the chosen file path.
  Display: "Saved to {chosen_filename}"
  STOP. Do not offer to re-run or ask any follow-up questions.

**If the user chooses 3 (Print to screen):**
  Display: "Result is displayed above."
  STOP.

## Step Q: Quick Mode

This step runs only when `quick_mode` is true (set in Step 1). It replaces Steps 3-10 entirely.

Display:
```
**QUICK** | Model: {agent_model} | Type: {input_type}
```

Spawn a single agent using the Agent tool with `model: {agent_model}`:

```
You are a sharp, efficient editor. Improve the material below based on the user's direction.

DIRECTION: {user_prompt}
MATERIAL: {subject}

(Match length to material complexity.)

Respond in EXACTLY this format:

CHANGES:
- [Bullet list of what you changed and why, 3-7 items]

IMPROVED VERSION:
[Complete improved version]
```

Wait for the agent to complete. Extract CHANGES and IMPROVED VERSION.

Display:

```
---
CHANGES
---

{extracted CHANGES bullets}

--- FINAL VERSION ---
```

{extracted IMPROVED VERSION}

Store the improved version as `current_draft` and proceed to Step 11 for save options.
