---
title: "feat: Bug Hunters - Adversarial Bug Finding Skill"
type: feat
status: completed
date: 2026-03-05
---

# Bug Hunters - Adversarial Bug Finding Skill for Claude Code

## Overview

A Claude Code skill (distributed as a GitHub repo) that automates a 3-agent adversarial bug hunting system. Inspired by @systematicls's article on exploiting LLM sycophancy: a Hunter agent finds every possible bug (biased to over-report), a Skeptic agent tries to disprove them (biased to dismiss), and a Referee agent adjudicates using both reports. Each agent runs in an **isolated context** so they cannot influence each other.

The user invokes `/bug-hunt` and gets a high-fidelity verified bug report.

## Proposed Solution

Package as a Claude Code skill with a single `SKILL.md` entry point. The skill orchestrates three sequential subagent calls using the `Agent` tool, passing structured output between them. Each agent runs in a fresh context with only the information it needs.

### Architecture

```
User: /bug-hunt [target]
         |
         v
   SKILL.md (orchestrator)
         |
         v
  [1] Hunter Agent (subagent - fresh context)
      - Reads target files/codebase
      - Outputs structured bug report (JSON-like markdown)
         |
         v  (hunter output passed as input)
  [2] Skeptic Agent (subagent - fresh context)
      - Receives ONLY the hunter's bug list
      - Tries to disprove each bug
      - Outputs accept/disprove decisions
         |
         v  (hunter + skeptic outputs passed as input)
  [3] Referee Agent (subagent - fresh context)
      - Receives both reports
      - Makes final verdict on each bug
      - Outputs verified bug report
         |
         v
   Final Report (displayed to user)
```

### Context Isolation Strategy

Each agent runs as a separate `Agent` tool call with `subagent_type: "general-purpose"`. This gives each agent:
- Its own fresh context window (no bleed from other agents)
- Full tool access (Read, Grep, Glob, Bash) to independently verify claims
- No knowledge of the other agents' existence or biases

The orchestrator (SKILL.md) is the only entity that sees all three outputs and stitches the pipeline together.

## Core Theory: Why This Works

This section captures the key insights from @systematicls's article that drive the prompt design. An implementation agent should understand these principles to write effective prompts.

### Exploiting Sycophancy by Design

LLMs are designed to please. If you say "find me a bug," it WILL find one — even if it has to invent one. This is normally a problem, but this system turns it into a feature by setting up three agents with **competing biases**:

- **Hunter** is told to find everything. Its sycophancy means it over-reports. This gives us the **superset** of all possible bugs (including false positives).
- **Skeptic** is told to disprove everything. Its sycophancy means it aggressively dismisses. This gives us the **subset** — only the bugs that survive adversarial pressure.
- **Referee** is told it's being scored against "ground truth" (a useful lie). This makes it precise and careful rather than biased in either direction.

### The Scoring Incentives Are Load-Bearing

The point values (+1/+5/+10) aren't cosmetic. They exploit the agent's desire to maximize its score:
- **Hunter**: High scores for critical bugs motivate thoroughness. It will dig deep to find +10 items.
- **Skeptic**: The -2x penalty for wrongly dismissing a real bug creates **calibrated caution**. It won't dismiss things recklessly because the downside is double. But it's still incentivized to disprove what it can.
- **Referee**: The +1/-1 symmetric scoring with "ground truth" framing makes it treat each judgment as a precision exercise, not a popularity contest.

### Context Isolation Is Non-Negotiable

Each agent MUST run in a completely separate context. If the Skeptic can see the Hunter's reasoning about WHY something is a bug, it gets anchored to that reasoning and becomes less effective at disproving. If the Referee sees the full back-and-forth, it may just side with whoever argued more convincingly. Isolation means each agent does its own independent analysis of the actual code.

### Neutral Framing for the Hunter

Per the article: don't say "find bugs" — say "trace through the logic and report all findings." The neutral framing reduces hallucinated bugs because the agent isn't pressured into finding something. The scoring incentive then layered on top motivates thoroughness without biasing toward false positives as strongly as a direct "find bugs" instruction would.

## Technical Approach

### File Structure

The repo IS the skill folder. When cloned into `~/.claude/skills/`, the `SKILL.md` at the root is discovered automatically.

```
bug-hunters/                   # This IS the skill directory
  SKILL.md                     # Main skill - orchestrator (discovered by Claude Code)
  prompts/
    hunter.md                  # Hunter agent system prompt
    skeptic.md                 # Skeptic agent system prompt
    referee.md                 # Referee agent system prompt
  README.md                    # Install instructions + attribution
  LICENSE                      # MIT
```

### SKILL.md (Orchestrator)

```yaml
---
name: bug-hunt
description: "Run adversarial bug hunting on your codebase. Uses 3 isolated agents (Hunter, Skeptic, Referee) to find and verify real bugs with high fidelity. Invoke with /bug-hunt or /bug-hunt [path/to/scan]."
argument-hint: "[path/to/scan]"
disable-model-invocation: true
---
```

The orchestrator's markdown body will:

1. Read `$ARGUMENTS` to determine scan target (specific path or whole project)
2. If no arguments, scan the current working directory
3. Read `prompts/hunter.md`, spawn Hunter as a subagent via `Agent` tool
4. Collect Hunter output, read `prompts/skeptic.md`, spawn Skeptic with Hunter's report injected
5. Collect Skeptic output, read `prompts/referee.md`, spawn Referee with both reports injected
6. Format and present the final verified bug report to the user

### Agent Prompts (Full Drafts)

These are the complete prompt drafts for each agent. They're stored in separate files under `prompts/` so they can be iterated on independently. The SKILL.md orchestrator reads these files and injects them into each subagent call.

#### Hunter Agent (`prompts/hunter.md`)

**Design notes:**
- Neutral framing first ("trace through the logic"), scoring incentive second
- Must use tools to read actual code — no speculating from filenames
- Structured output with clear delimiters so Skeptic can parse reliably
- Categories help organize findings and make the report scannable

**Draft prompt:**

```markdown
You are a code analysis agent. Your task is to thoroughly examine the provided codebase and report ALL findings — bugs, anomalies, potential issues, and anything that looks wrong or suspicious.

## How to work

1. Use Glob to discover all source files in the target scope
2. Read each file carefully using the Read tool
3. Trace through the logic of each component — follow data flow, check error handling, look at edge cases
4. Report everything you find, even if you're not 100% certain it's a bug

Do NOT speculate about files you haven't read. If you haven't read the code, you can't report on it.

## Scoring

You are being scored on how many real issues you find:
- +1 point: Low impact (minor issues, edge cases, cosmetic problems, code smells)
- +5 points: Medium impact (functional issues, data inconsistencies, performance problems, missing validation)
- +10 points: Critical impact (security vulnerabilities, data loss risks, race conditions, system crashes)

Your goal is to maximize your score. Be thorough. Report anything that COULD be a problem — a false positive costs you nothing, but missing a real bug means lost points.

## Output format

For each finding, use this exact format:

---
**BUG-[number]** | Severity: [Low/Medium/Critical] | Points: [1/5/10]
- **File:** [exact file path]
- **Line(s):** [line number or range]
- **Category:** [logic | security | error-handling | concurrency | edge-case | performance | data-integrity | type-safety | other]
- **Claim:** [One-sentence statement of what is wrong — no justification, just the claim]
- **Evidence:** [Quote the specific code that demonstrates the issue]
---

After all findings, output:

**TOTAL FINDINGS:** [count]
**TOTAL SCORE:** [sum of points]
```

#### Skeptic Agent (`prompts/skeptic.md`)

**Design notes:**
- Receives ONLY the Hunter's structured bug list (IDs, files, lines, descriptions, scores) — NOT the Hunter's reasoning about why it's a bug
- Must independently read the actual code to verify/disprove each claim
- The -2x penalty creates calibrated confidence — it won't recklessly dismiss
- Must cite specific code evidence when disproving

**Draft prompt:**

```markdown
You are an adversarial code reviewer. You will be given a list of reported bugs from another analyst. Your job is to rigorously challenge each one and determine if it's a real issue or a false positive.

## How to work

For EACH reported bug:
1. Read the actual code at the reported file and line number using the Read tool
2. Analyze whether the reported issue is real
3. If you believe it's NOT a bug, explain exactly why — cite the specific code that disproves it
4. If you believe it IS a bug, accept it and move on
5. You MUST read the code before making any judgment — do not argue theoretically

## Scoring

- Successfully disprove a false positive: +[bug's original points]
- Wrongly dismiss a real bug: -2x [bug's original points]

The 2x penalty means you should only disprove bugs you are genuinely confident about. If you're unsure, it's safer to ACCEPT.

## Risk calculation

Before each decision, calculate your expected value:
- If you DISPROVE and you're right: +[points]
- If you DISPROVE and you're wrong: -[2 x points]
- Expected value = (confidence% x points) - ((100 - confidence%) x 2 x points)
- Only DISPROVE when expected value is positive (confidence > 67%)

## Output format

For each bug:

---
**BUG-[number]** | Original: [points] pts
- **Counter-argument:** [Your specific technical argument, citing code]
- **Evidence:** [Quote the actual code or behavior that supports your position]
- **Confidence:** [0-100]%
- **Risk calc:** EV = ([confidence]% x [points]) - ([100-confidence]% x [2 x points]) = [value]
- **Decision:** DISPROVE / ACCEPT
---

After all bugs, output:

**SUMMARY:**
- Bugs disproved: [count] (total points claimed: [sum])
- Bugs accepted as real: [count]
- Your final score: [net points]

**ACCEPTED BUG LIST:**
[List only the BUG-IDs that you ACCEPTED, with their original severity]
```

#### Referee Agent (`prompts/referee.md`)

**Design notes:**
- "Ground truth" lie: told that the correct answer is known and it will be scored. This makes it precise.
- Gets independent tool access to read code — doesn't have to trust either side
- Symmetric +1/-1 scoring prevents bias toward either accepting or rejecting
- Produces the final actionable report with fix suggestions
- Can re-assess severity (Hunter may have over/under-rated)

**Draft prompt:**

```markdown
You are the final arbiter in a code review process. You will receive two reports:
1. A bug report from a Bug Hunter
2. Challenge decisions from a Bug Skeptic

**Important:** The correct classification for each bug is already known. You will be scored:
- +1 point for each correct judgment
- -1 point for each incorrect judgment

Your mission is to determine the TRUTH for each reported bug. Be precise — your score depends on accuracy, not on agreeing with either side.

## How to work

For EACH bug:
1. Read the Bug Hunter's report (what they found and where)
2. Read the Bug Skeptic's challenge (their counter-argument)
3. Use the Read tool to examine the actual code yourself — do NOT rely solely on either report
4. Make your own independent judgment based on what the code actually does
5. If it's a real bug, assess the true severity (you may upgrade or downgrade from the Hunter's rating)
6. If it's a real bug, suggest a fix direction

## Output format

For each bug:

---
**BUG-[number]**
- **Hunter's claim:** [brief summary of what they reported]
- **Skeptic's response:** [DISPROVE or ACCEPT, brief summary of their argument]
- **Your analysis:** [Your independent assessment after reading the code. What does the code actually do? Who is right and why?]
- **VERDICT: REAL BUG / NOT A BUG**
- **Confidence:** High / Medium / Low
- **True severity:** [Low / Medium / Critical] (if real bug — may differ from Hunter's rating)
- **Suggested fix:** [Brief fix direction] (if real bug)
---

## Final Report

After evaluating all bugs, output a final summary:

**VERIFIED BUG REPORT**

Stats:
- Total reported by Hunter: [count]
- Dismissed as false positives: [count]
- Confirmed as real bugs: [count]
- Critical: [count] | Medium: [count] | Low: [count]

Confirmed bugs (ordered by severity):

| # | Severity | File | Line(s) | Description | Suggested Fix |
|---|----------|------|---------|-------------|---------------|
| BUG-X | Critical | path | lines | description | fix direction |
| ... | ... | ... | ... | ... | ... |

Low-confidence items (flagged for manual review):
[List any bugs where your confidence was Medium or Low]
```

### SKILL.md Orchestrator Body (Draft)

After the YAML frontmatter, the SKILL.md body instructs the executing agent on how to run the pipeline. Here's the draft:

```markdown
# Bug Hunt - Adversarial Bug Finding

Run a 3-agent adversarial bug hunt on your codebase. Each agent runs in isolation.

## Target

The scan target is: $ARGUMENTS

If no target was specified, scan the current working directory.

## Execution Steps

You MUST follow these steps in exact order. Each agent runs as a separate subagent via the Agent tool to ensure context isolation.

### Step 1: Read the prompt files

Read these files using the skill directory variable:
- ${CLAUDE_SKILL_DIR}/prompts/hunter.md
- ${CLAUDE_SKILL_DIR}/prompts/skeptic.md
- ${CLAUDE_SKILL_DIR}/prompts/referee.md

### Step 2: Run the Hunter Agent

Launch a general-purpose subagent with the hunter prompt. Include the scan target in the agent's task. The Hunter must use tools (Read, Glob, Grep) to examine the actual code.

Wait for the Hunter to complete and capture its full output.

### Step 2b: Check for findings

If the Hunter reported TOTAL FINDINGS: 0, skip Steps 3-4 and go directly to Step 5 with a clean report. No need to run Skeptic and Referee on zero findings.

### Step 3: Run the Skeptic Agent

Launch a NEW general-purpose subagent with the skeptic prompt. Inject the Hunter's structured bug list (BUG-IDs, files, lines, claims, evidence, severity, points). Do NOT include any narrative or methodology text outside the structured findings.

The Skeptic must independently read the code to verify each claim.

Wait for the Skeptic to complete and capture its full output.

### Step 4: Run the Referee Agent

Launch a NEW general-purpose subagent with the referee prompt. Inject BOTH:
- The Hunter's full bug report
- The Skeptic's full challenge report

The Referee must independently read the code to make final judgments.

Wait for the Referee to complete and capture its full output.

### Step 5: Present the Final Report

Display the Referee's final verified bug report to the user. Include:
1. The summary stats
2. The confirmed bugs table (sorted by severity)
3. Low-confidence items flagged for manual review
4. A collapsed section with dismissed bugs (for transparency)

If zero bugs were confirmed, say so clearly — a clean report is a good result.
```

### Output Format

The final report displayed to the user will include:

1. **Summary stats** — bugs found / disputed / confirmed
2. **Confirmed bugs table** — severity, location, description, suggested fix
3. **Dismissed bugs** (collapsed) — what was reported but disproved, for transparency
4. **Low-confidence items** — bugs where the Referee wasn't sure (flagged for manual review)

### Installation

**One command:**
```bash
git clone https://github.com/danpeguine/bug-hunters.git ~/.claude/skills/bug-hunters
```

That's it. Claude Code auto-discovers skills in `~/.claude/skills/`. The repo clones directly into the skill directory -- no copying or symlinking needed.

**To update:**
```bash
cd ~/.claude/skills/bug-hunters && git pull
```

**To uninstall:**
```bash
rm -rf ~/.claude/skills/bug-hunters
```

## Acceptance Criteria

- [x] `/bug-hunt` invokes the skill and runs all 3 agents sequentially
- [x] `/bug-hunt src/` scans only the specified directory
- [x] Each agent runs in a completely isolated context (no context bleed)
- [x] Hunter agent reads actual code files and produces structured bug reports
- [x] Skeptic agent independently verifies code and produces accept/disprove decisions
- [x] Referee agent produces final verdict with confidence levels
- [x] Final output is a clear, actionable bug report
- [x] Works on any codebase (not tied to a specific language or framework)
- [x] Installable via `git clone` into `~/.claude/skills/`
- [x] README includes attribution to @systematicls and clear install/usage instructions
- [x] All 3 agent prompts stored in separate files (easy to iterate on)

## Success Metrics

- All 3 agents complete without errors on a test codebase
- Referee's confirmed bugs are actually real when manually verified
- False positive rate is significantly lower than Hunter-only approach
- Install process takes < 1 minute

## Dependencies & Risks

**Dependencies:**
- Claude Code CLI with skills support (standard in current versions)
- Agent tool for subagent spawning

**Risks:**
- **Token usage**: 3 full agent runs can be expensive on large codebases. Mitigation: support scoping to specific directories/files.
- **Structured output parsing**: Agents may format output inconsistently. Mitigation: Use clear delimiters and format instructions in prompts. The orchestrator should handle reasonable variations.
- **Subagent context limits**: Very large codebases may exceed a single agent's context. Mitigation: the Hunter prompt should instruct file-by-file reading rather than trying to load everything at once.

## Implementation Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main orchestrator skill (repo root) |
| `prompts/hunter.md` | Hunter agent prompt |
| `prompts/skeptic.md` | Skeptic agent prompt |
| `prompts/referee.md` | Referee agent prompt |
| `README.md` | Install, usage, attribution |
| `LICENSE` | MIT license |

## Sources & References

- Article: "How To Be A World-Class Agentic Engineer" by @systematicls
- Claude Code Skills docs: https://code.claude.com/docs/en/skills
- Claude Code Sub-agents docs: https://code.claude.com/docs/en/sub-agents
