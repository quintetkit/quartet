# Quartet

A configuration set for Claude Code that assigns four distinct personas—**Architect**, **Coder**, **Reviewer**, and **Conflict Resolver**—to enable parallel development centered around GitHub Issues and branches.

This repository does not contain code. Copy `CLAUDE.md`, `.claude/agents/`, and `.github/ISSUE_TEMPLATE/` to your project to use them. It is technology-stack agnostic.

## Why this is needed

When giving large requests to Claude Code, the following issues tend to occur:

- **Unrequested changes.** The diff expands, making reviews impossible to keep up with.
- **File conflicts in parallel runs.** Conflicts become difficult to resolve later.
- **Progress is trapped in conversation history.** You cannot know where work stopped yesterday without tracing back through logs.
- **Self-review and self-merge.** There are no checks and balances when one persona writes, reviews, and merges code.

The cause is not capability. It is permission design. When one persona designs, implements, reviews, and merges, nothing stops it.

Quartet divides responsibilities and **reduces permissions for each role**.

- The Coder cannot modify files outside the scope defined in the Issue.
- Only the Reviewer can merge to main.
- Conflict resolution is handled exclusively by a dedicated persona.
- Progress is tracked via GitHub Issue labels, not conversation logs.

## The Four Personas

| Persona | Role | Writes Code? |
|---|---|---|
| **Architect** | Splits requests into Issues. Defines scope (target paths) and dependencies to form groups that can work in parallel. | No |
| **Coder** | Handles a single Issue. Modifies only files within their scope and creates a PR. | Yes |
| **Reviewer** | Verifies acceptance criteria and scope violations, then merges. **The only persona allowed to merge.** | No |
| **Conflict Resolver** | Appears only when conflicts occur, resolves them, and returns the result to the Reviewer. | Resolution only |

```
Request → [Architect] Split into Issues (create groups with non-overlapping scopes)
          │
          ├─ [Coder] issue/12-xxx ─┐
          ├─ [Coder] issue/13-yyy ─┤ Parallel
          └─ [Coder] issue/14-zzz ─┘
                                    │ One by one per PR
                                    ▼
                             [Reviewer] Verify acceptance criteria and scope → Merge
                                    │
                                    └─ Conflict → [Conflict Resolver] → Reviewer
```

## What is enforced and what is only asked

**"Does not write code" is not left to instructions.** It is removed at the tool level.

| Persona | `tools` | Write / Edit |
|---|---|---|
| Architect | `Bash, Read, Grep, Glob` | **not available** |
| Reviewer | `Bash, Read, Grep, Glob` | **not available** |
| Conflict Resolver | `Bash, Read, Edit, Grep, Glob` | Edit only |
| Coder | `*` | available |

Architect and Reviewer do not have the tools that edit files. They are not being
asked not to write code; they have no way to.

### What is not enforced

Stated plainly: **all four personas have `Bash`.**

So these three rest on instructions alone:

- the Coder staying inside its declared scope
- the Coder not merging (`gh pr merge` is reachable from Bash)
- the Reviewer not implementing (a file can be written from Bash)

Removing `Bash` would take away test runs, `git` and `gh`, and the workflow
stops working. **It cannot be removed, so the boundary is documented instead.**

If you want it enforced, add rules such as `Bash(gh pr merge:*)` to
[`permissions.deny`](https://code.claude.com/docs/en/settings-reference) in
`.claude/settings.json`. Note that those apply to **the whole session**, so the
Reviewer's merges stop too. There is currently no way to vary Bash subcommand
permissions per persona.

**Not mixing "the machine stops this" with "we asked nicely" is the policy of
this repository.**

## Usage

```bash
git clone https://github.com/quintetkit/quartet.git
cp -R quartet/CLAUDE.md quartet/.claude quartet/.github quartet/setup.sh /path/to/your-project/

cd /path/to/your-project
./setup.sh          # Creates labels used for state management (requires gh CLI)
```

`.github/workflows/issue-status.yml` is included as well. It moves the status
label on the linked Issue in step with the PR.

| What happened on the PR | Issue label |
|---|---|
| PR opened | `status:review` |
| Review requested changes | `status:changes-requested` |
| PR merged | `status:done` |

It finds the Issue from `Closes #<n>` in the PR body. Personas forget to move
labels, so state tracking is left to the machine.

Then, ask Claude Code as follows:

```
Use the architect to split the login feature into Issues.
```

Once the Issues are created, launch the Coders in the parallel work group **simultaneously with a single message**. Calling them sequentially will not result in parallel execution. This is explicitly stated in `CLAUDE.md`.

## Design Decisions

Three points to understand before reading:

- **Reviewer merges are not parallel.** While implementation can be parallel, merges are processed one by one sequentially. Merging in parallel causes frequent conflicts and ends up slower.
- **The Coder stops if it goes out of scope.** It does not widen the scope on its own. It reports back so the Architect can split the work into a new Issue. This prevents "I fixed it while I was at it" scenarios.
- **The Conflict Resolver does not act until called.** It does not proactively prevent conflicts. Scope design errors are the Architect's responsibility; covering them up with conflict resolution hides the root cause.

## Regarding UI Development

This four-persona structure does not address UI design. Therefore, if you ask it to create screens, it will generate typical AI defaults (full-width centered hero sections, 3-column cards with icons, purple gradients).

You can avoid this by inserting a fifth persona that finalizes UI specifications before implementation. We distribute **Quintet**, which adds this UI Designer persona, the Reviewer's decision criteria, a per-Issue parallel execution script, and a 10-chapter practical guide in English and Japanese ([Quintet, $39](https://quintetkit.gumroad.com/l/quintet)).

The evaluation criterion for the UI Designer is solely "whether traces of human judgment remain." It explicitly prohibits AI defaults such as gradient backgrounds, pure black `#000000` and pure white `#ffffff`, full-width centered hero sections, and 3-column equal-width cards with icons. It requires designing states for all eight conditions: default / hover / focus-visible / active / disabled / loading / error / success.

## Related tools

This workflow rests on one rule: **Issues that share a file must not run in
parallel.** Two MIT tools check things it takes for granted.

- [scopecheck](https://github.com/quintetkit/scopecheck) reports the open Issues
  whose declared scopes overlap, naming the files they have in common
- [ccheck](https://github.com/quintetkit/ccheck) lints a `.claude/` directory,
  citing the documentation behind every finding

Both have no dependencies and run without installing anything.

```bash
npx @quintetkit/ccheck        # from your repository root
npx @quintetkit/scopecheck --repo owner/name
```

## License

MIT

The format for sub-agent definitions is based on
[VoltAgent/awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents).
