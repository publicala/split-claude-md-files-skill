# Eval plan (not built)

Fixture (scaffold_script): tiny repo whose root CLAUDE.md plants one of each:

- edit-shaped convention, every governed file in one subtree → expect DEMOTE to `<subdir>/CLAUDE.md`
- convention governing one extension across directories → expect DEMOTE to a path-scoped rule with globs
- commit-style rule naming nothing greppable → expect KEEP resident
- safety prohibition whose named files all sit in one directory → expect KEEP at root over the grep
- rule binding new-file creation → expect a load-path probe before any verdict
- scoped rule whose governed paths spill outside its subtree → expect PROMOTE
- path-scoped rule whose globs match nothing → expect FLAG as orphan, deletion deferred to audit

Prompt: "Run the split on this repo." Grader: LLM judge checks the report for the seven verdicts. Run: `claude plugin eval split-claude-md-files --ablation with-without --runs 1`
