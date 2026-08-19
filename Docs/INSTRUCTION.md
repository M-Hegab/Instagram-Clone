Please strictly follow these rules:

# Commands
- Test: `npm test` Typecheck: `npm run typecheck` Lint: `npm run lint` Build: `npm run build`

# Skills
- clean-code-guard, test-guard, docs-guard are mandatory, not suggestions.
- Loading a skill once does not expire it. Re-run its self-check before
  delivery after EVERY later edit in the session, not just the first one.
- Production code written, edited, refactored, or fixed → clean-code-guard
  guard-pass on the diff before presenting or committing.
  Read references/ai-failure-modes.md first. Fix violations, don't just list them.
- Test files (test_*.py, *_test.go, *Test.php, *.test.ts, or under
  tests/ __tests__/ spec/) → test-guard, while writing them, not after.
  Every test answers "what bug does this catch that nothing else catches?"
- Docs touched, or code changed whose behavior is documented → docs-guard.
  Every symbol, flag, endpoint, config key, and path gets verified against
  source in this session. Verified from memory is not verified.
- Match the skill to the surface. Prod code → clean-code-guard.
  Tests → test-guard. Docs → docs-guard. Don't cross them over.
- On conflict, this file wins over the skills. Say which rule you deferred to.
- Skills are judgement, not verification. They never replace typecheck + tests.

# Workflow
- Change touching >1 file or altering behavior → plan first (plan mode).
If you can describe the diff in one sentence, just do it.
- Ambiguous requirements → interview me with AskUserQuestion. Don't guess.
- Stop at the end of each Phase. Summarize, wait for approval.
- Single branch, commit per Phase, one PR at the end.

# Definition of done
- YOU MUST run typecheck + tests before claiming done.
- Paste the raw output. "I verified it" without output is not done.
- Name what you checked for regressions.
- Name which guard skills ran and what each one flagged. "Clean" is a claim —
  say what you checked for, not that you checked.

# Sub-agents
- Read-only exploration and search: use aggressively.
- Never parallelize edits in the same working tree.
- After implementing: subagent reviews the diff against SPEC.md.
Report correctness gaps only, not style.
- That subagent also runs the matching guard skill in review mode and reports
  findings with file:line. Review mode reports, it does not edit.

# Disagreement
- Push back with specific reasoning and evidence when I'm wrong.
- If you agree, one line and move on. No manufactured objections.
- A guard-skill rule is evidence — cite the rule, not "best practice".

# Gotchas
- (log real repeated mistakes here only)