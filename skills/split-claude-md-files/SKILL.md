---
name: split-claude-md-files
description: >
  Moves CLAUDE.md rules to the load scope of the sessions they govern: demotes always-resident rules that bind one subtree or one file pattern into nested CLAUDE.md files or path-scoped rules, promotes scoped rules that outgrew their file, and flags orphaned scoped files. Use when a resident CLAUDE.md grows past about 200 lines, when the root file accumulates rules that only apply to one area, or after an audit-claude-md-files run.
user-invocable: true
disable-model-invocation: true
---

Judge every rule of every in-scope CLAUDE.md against one question: **which sessions does this rule govern?** A resident rule that governs one subtree taxes every other session for nothing, and a scoped rule that governs the whole project is missing from most sessions that need it. The goal is placement: every rule loads in the sessions it governs, and in no others.

Throughout, "demote" and "promote" mean recording a verdict in the report. No file changes before approval, no exceptions.

This skill decides where rules live, never whether they live. A rule that looks cuttable gets deferred to `audit-claude-md-files`, not moved. Run split after an audit, so placement work is spent only on lines that earned their keep.

## Scope

Default scope is the current project tree. Writes only ever land in the project's CLAUDE.md files and `.claude/rules/` files. User-level and ancestor-directory files are out of scope in both directions: they load in every project, so no in-project placement can host their content.

## Load model

Placement chooses among three load classes:

- **Always resident**: the root CLAUDE.md, ancestor and user-level files, and anything they pull in with `@path` imports. Full price in every session.
- **Nested CLAUDE.md** (`<subdir>/CLAUDE.md`): loads when the session reads a file under that directory, then stays for the session. Scopes by subtree.
- **Path-scoped rules** (`.claude/rules/<topic>.md` with `paths` globs in the frontmatter): load when the session reads a file matching a glob. The only class that scopes by file pattern across directories.

The trigger for both scoped classes is a read, and only a read. A session that creates a new in-scope file without reading a neighbor first, or that touches in-scope paths only through shell commands, never loads the scoped file. Edits are covered because an edit requires a prior read. Every demotion verdict hangs on this fact. Compaction adds a second asymmetry: the root file is re-injected after a compaction automatically, while scoped files return only on the next in-scope read.

## Never demote

Check each rule against this list before measuring its scope. A match stays resident whatever the measurements say.

- Safety prohibitions, narrowly defined: rules guarding actions whose harm lands outside the working tree the moment the action runs (data loss, mutating a shared or production system, publishing, deleting history). Review cannot catch these, and the action is rarely gated behind reading one subtree, so the scoped copy can be absent at the moment it matters. This holds even when every file the rule names lives in one directory: the grep says demote, this list says stay. A convention phrased with "never" is not a safety prohibition: "Never import Carbon\Carbon" bans a pattern a reviewer catches in the diff, and it goes through the normal pass.
- Precedence and routing clauses, and read-triggers for referenced docs. They exist to route sessions that have not read anything yet.

## The pass, in order

### 1. Inventory

List each in-scope file with its load class and est. token cost (label every figure "est."). Include `.claude/rules/*.md` files with their `paths` globs resolved against the tree.

### 2. Scope evidence

For each rule in an always-resident file, grep the first-party paths the rule binds: the files where following or violating it is possible. Use the subjects the rule itself names (a class, a directory, an extension, an API). Report the paths found, and let them decide the shape:

- Every governed path under one directory: subtree-shaped, destination `<subdir>/CLAUDE.md`.
- Governed paths span directories but match one or two globs: glob-shaped, destination a path-scoped rule file.
- The rule names nothing greppable (commit style, PR flow, shell habits): path-free, stays resident.

The grep is the verdict on shape. The author's sense of where a rule "belongs" is not evidence.

### 3. Load-path check

A demotion only works if the scoped file is loaded by the time the rule matters. Name the mistake the rule prevents, then classify the action that commits it:

- Editing an existing in-scope file: demote. The harness refuses an edit without a prior read in the session, and the read loads the scoped file.
- Creating a new in-scope file, or acting on in-scope paths through shell or git: the read trigger can miss. For these rules no verdict exists until a load-path probe runs. Give one fresh low-effort agent the task that would commit the mistake, in the real repo, without mentioning the rule, and record from its tool calls whether it reads an in-scope file before the point where the mistake happens. It reads first: demote. It acts without reading: the rule stays resident. The probe's outcome is the verdict. The author guessing whether sessions read first is the same simulation the probe replaces.

### 4. The other direction

For each rule already in a scoped file, run step 2's grep from that file's viewpoint. Governed paths outside the file's subtree or globs mean the rule outgrew its home: promote it to the smallest target that covers every governed path (a wider glob, an ancestor directory's CLAUDE.md, or root). Promotions pass through "Never demote" in reverse trivially: widening a rule's audience is always load-safe.

Flag orphans while there: a nested CLAUDE.md whose directory no longer holds first-party files, or a path-scoped rule whose globs match nothing. Deletion verdicts belong to `audit-claude-md-files`, so record orphans as flags, not cuts.

### 5. Approval and apply

Ask at the start of the pass, one AskUserQuestion: does the user want the report as an interactive artifact or as plain text? The artifact is a live doc (`capabilities: {artifact: {}}`): one row per rule with its repo-relative source and destination paths, an approve checkbox per verdict, a free-text input per open question, per-file diffs of the proposed moves (drafted in scratchpad, doubling as the apply source), and a copy-decisions control as the fallback path back into the session.

Present the full report before editing anything. Per rule: the verdict (demote, promote, keep, flag), the destination, and the evidence (governed paths, load-path result). Only edit after approval, on a branch with a PR for checked-in files. Close with est. resident tokens per session before and after, listing separately what every session pays and what only in-scope sessions pay.

After applying, verify that each scoped file the edits created or re-globbed actually loads: one probe per created or changed glob, each a fresh low-effort agent that reads one file matched by that glob and quotes every context line containing a marker phrase unique to the scoped file. One file cannot vouch for the globs that did not select it: a valid glob loads the rule and hides a malformed neighbor. Tell each agent explicitly to search its entire context window and not just the file it read: asked about "the file", an agent answers about the file alone and returns a false NONE. The phrase comes back: the glob fires. NONE comes back: rerun that probe once before judging, since self-report probes return false NONEs at a measurable rate even with correct phrasing. NONE twice: the placement is broken, and the move reverts until the globs are fixed.

## Building the decision artifact

The artifact is a decision surface, not a document. The user often decides from a phone, so every item renders as a compact card: path, a one-sentence claim, one evidence line. Compact governs your own prose and never the source: text the user is approving (the line being cut, the rule being written) appears verbatim and whole, however long it runs.

- **Show the line, don't cite it.** Any claim about text the reader cannot see gets a collapsed "See the lines" block quoting the offending line verbatim in diff-removed styling, with the replacement (where one exists) in diff-added styling. The reader must never need the repo open to decide. Abridge very long lines with `[...]` and point at the full diff. Quote blocks hold file lines only: never explanation prose inside the block (that belongs in the card's claim or evidence line), and each quoted line carries its real line number in a muted, unselectable gutter.
- **A note field on every row.** Every decision row carries a free-text note field, always visible and in the tab order, never behind a disclosure the user must click first: opening a control per row breaks the review flow. Keep it one row tall and let it grow with content (`field-sizing: content`, plus `resize: vertical` as the fallback). Give every interactive control a visible `:focus-visible` outline so the whole page is walkable by keyboard. The user may have an opinion or question anywhere, including near-certain items.
- **Keyboard flow.** <kbd>j</kbd>/<kbd>k</kbd> walk the decision rows, <kbd>x</kbd> toggles the row, <kbd>n</kbd> focuses its note, <kbd>e</kbd> toggles its quoted lines, <kbd>Esc</kbd> leaves the note (document the keys on the page, hidden on touch widths). On wide viewports open the quote blocks by default, so deciding needs no clicks at all.
- **Open-in-editor links.** Every file reference gets a small open link built on the user's editor URL scheme, detected from the machine (`$EDITOR`, installed apps): `zed://file{abs}:{line}`, `vscode://file{abs}:{line}`, `cursor://file{abs}:{line}`, `phpstorm://open?file={abs}&line={n}`. Always absolute paths, so the links hold from any worktree, and percent-encode them (keeping `/` in the path, encoding the whole PhpStorm query value) so a `#`, `?`, `%`, or `&` inside a path cannot truncate the target. Evidence references carrying file:line link the same way. Use `target="_blank"`, and stop click propagation on these links in a capture-phase handler, since they sit inside the checkbox label and a click must never toggle the row. The link opens on the machine running the browser, so emit these only when the session runs on the user's own machine: a remote session (SSH, devcontainer, cloud sandbox) reads a different filesystem and a different set of installed editors, so omit the links there rather than pointing an absent editor at a path that does not exist.
- **Word-level diffs.** Pair adjacent removed/added runs (SequenceMatcher on tokens, similarity at or above 0.4) and highlight only the changed tokens. Render each diff line as a `display:block` span and join the spans with no separator: a newline between block spans inside a `<pre>` renders as a phantom blank line. Lint-enforced docs keep whole paragraphs on one physical line, so wrap with `white-space:pre-wrap`, `overflow-wrap:anywhere`, and a hanging indent.
- **Questions as options.** Render each open question as radio options with a "recommended" chip plus a free-text field, mirroring AskUserQuestion. A bare textarea is only for questions with no concrete options.
- **Sticky decision bar.** Section links, a changed-from-default counter, and the copy-decisions control stay reachable while scrolling.
- **A fallback that carries everything.** The copy-decisions control serializes every control on the page: each checkbox, the selected option of each question, and each free-text field. The pasted export is an accepted approval path, so any state it drops is a decision the session then applies wrongly.
- **Triage chips.** Label each diff row by the review effort it needs (cuts-only rows are skimmable, rewrites are worth opening, adds carry new lines) and provide an expand-all-diffs control.
- **Mobile pass.** Verify the page at around 390px width before publishing. Collapse method and other secondary sections by default.
- **Republish safety.** Viewer decisions reach the session only because the page is a live doc (`capabilities: {artifact: {}}`), which saves the DOM a viewer's gesture changed back into the served document. Before any republish, fetch the live artifact and compare its state against defaults. Carry any non-default state into the rebuilt HTML, or do not republish: republishing over live decisions destroys them. Publish without that capability and the state never leaves the viewer's browser, where the session cannot read it: then the clipboard export is the only path back, and republishing is off the table once the user starts deciding.
- **Generate, don't hand-edit.** Build the page from a data-plus-template script in the scratchpad so every iteration regenerates it whole.
- **End with the trigger.** Close the page by telling the user the exact phrase that resumes the session, such as "read the artifact and apply".

## Moves are verbatim

A moved rule keeps its exact text. The only permitted edit is deleting a scope qualifier the destination now expresses: "In tests/, never use RefreshDatabase" becomes "Never use RefreshDatabase" inside `tests/CLAUDE.md`. Phrasing improvements belong to `audit-claude-md-files`.

A demotion leaves nothing behind. The scoped file loads automatically, so a pointer at root would rebuild the cost the move removed. Create the destination file if it is missing, and append under a heading that matches its existing structure.

## Skill candidates

Content shaped as a procedure (a numbered workflow, a deploy guide, a checklist walked start to finish) belongs in a skill body, not in any CLAUDE.md. Flag it with a proposed skill name and stop there. Writing skills is a different job with its own standards.

## The quartet

- `feed-claude-md-files` adds rules from observed patterns
- `bake-claude-md-files` converts crystallized rules into tooling and removes the prose
- `audit-claude-md-files` prunes and verifies what remains
- `split-claude-md-files` moves what remains to the scope that reads it

Run `feed` after a working session, `bake` once enough rules have accumulated to be worth automating, `audit` when CLAUDE.md files have grown without review, and `split` after an audit leaves a resident file carrying rules that govern one area.
