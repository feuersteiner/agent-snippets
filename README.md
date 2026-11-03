# agent-snippets

Turn instructions into reusable workflows that generate content.

An agent compiles `SNIPPET.md` into one `snippet.yaml`: triggers select inputs,
actions generate or transform content, and evals verify the results. The
framework mirrors [agent-tests](https://github.com/feuersteiner/agent-tests/blob/964076aed883c352036cc0677b5e94f26abc2304/README.md).

Agent-snippets owns actions: generating content, rendering templates, creating
files and folders, applying naming conventions, and sequencing transformations.
Agent-tests owns checks: inspecting targets and reporting findings without
modifying them. Neither CLI depends on the other. Input validation and evals
belong to each tool; they do not change these boundaries.

**Status:** specification stage; no CLI is implemented or published yet.

## Example

A snippet is two files in `.agents/snippets/french-labels/`. `SNIPPET.md`
describes the task in ordinary prose:

```md
Translate button labels in HTML files into French. Preserve the markup and
leave labels already written in French unchanged.
```

The agent writes `snippet.yaml`, containing everything that runs:

```yaml
id: french-labels
title: French button labels
description: Translate HTML button labels into French without changing markup.
source: sha256:1cb98ca371262f140900996433b5030240527fed8951bdf40c0516b1500febd7
triggers:
  path:
    match: ["src/**/*.html"]
  content:
    match: "<button\\b"
actions:
  - id: translate-labels
    message: Button labels could not be translated.
    fix: Review the labels and translation instructions in SNIPPET.md.
    deep:
      ask: "Translate button labels into French. Return the complete file, preserving all markup and other text."
      reasoning: medium
evals:
  - name: English label
    path: src/save.html
    content: <button>Save</button>
    expect: pass
    output:
      path: src/save.html
      content: <button>Enregistrer</button>
```

## Where it runs

- During an agent session, for a named generation or transformation task.
- On selected files or a repository area, including uncommitted changes or a
  change relative to a base branch.
- In CI, when the workflow explicitly requests generated content.
- Through a Vitest plugin, comparing generated outputs with expectations.
- As a library from the `agent-snippets` npm package.

Execution always names a snippet. `init` creates configuration and
`.agents/snippets/`; it does not install an automatic generation hook.

## Get started (WIP)

Planned commands; these are not available yet:

```sh
npm install -D agent-snippets
npx agent-snippets init                         # config and .agents/snippets/
npx agent-snippets add french-labels             # install a community snippet
npx agent-snippets add french-labels apps/web    # install for apps/web/ only
npx agent-snippets compile french-labels         # brief the agent to generate YAML
npx agent-snippets validate french-labels        # structure and source freshness
npx agent-snippets list                         # installed snippets as a tree
npx agent-snippets targets french-labels         # preview selected inputs
npx agent-snippets eval french-labels            # verify labeled examples
npx agent-snippets run french-labels --dry-run   # preview generated changes
npx agent-snippets run french-labels             # execute the workflow
npx agent-snippets run french-labels --changed   # select uncommitted inputs
npx agent-snippets run french-labels --base main # select inputs changed since main
# For a generation snippet: run <id> --input src/new-file.html
npx agent-snippets docs                         # authoring and execution skills
npx agent-snippets vote french-labels            # toggle a community vote
```

Choose model providers in `agent-snippets.config.yaml`; keep credentials in
`.env`. Agent-tests remains an independent tool for checking repositories.

## Package

```text
.agents/snippets/<id>/
  SNIPPET.md
  snippet.yaml
```

`SNIPPET.md` describes the task, examples, exceptions, inputs, and intended
outputs in ordinary prose. The agent generates `snippet.yaml`, which holds
`id`, `title`, `description`, `source`, `triggers`, `actions`, and `evals`.
Ids are lowercase kebab-case and match their directory. `source` is
`sha256:<SHA-256 of the exact SNIPPET.md bytes>`.

One YAML file contains the entire executable workflow, including inline
instructions, templates, and labeled examples. Separate `workflow.yaml`,
`action-*.yaml`, scripts, and external test files are not part of this format.
YAML remains editable by hand.

## Triggers and scope

Triggers use the same semantics as agent-tests: each trigger reads an input's
`path`, its `content`, or both. Every filter named in a trigger must pass, in
the order written. A list of triggers holds alternatives and stops at the
first passing trigger. Rejected inputs are skipped without running actions.

- `path.match`: globs on the input path; a list matches any listed glob.
- `path.ask`: a decision model judging the path.
- `content.match`: a regex on the input text.
- `content.ask`: a decision model judging the input text.

```yaml
triggers:
  - path:
      match: ["content/**/*.md"]
  - path:
      match: ["src/**/*.html"]
      ask: "This path belongs to a customer-facing page."
    content:
      match: "[Aa]cme"
      ask: "This file contains marketing copy shown to customers."
```

Any directory can have `.agents/snippets/`. Snippets are available within that
directory alongside those inherited from parents; `.agents/snippets/config.yaml`
can opt out of specific parent snippets. Paths are relative to the directory
owning that `.agents/snippets/`, with the same scope for targets and evals.

For generation, the caller supplies a proposed output path and optional source
text as the input; absent text is the empty string. The path need not exist.
This lets the same triggers select a new file before content is generated.
For transformations, the input is an existing file and its text. Triggers
select inputs once, before actions start.

## Compilation

`agent-snippets compile <id>` gives the calling agent `SNIPPET.md`, a
snippet-generator skill, and the repository layout. The agent writes triggers,
ordered actions, and evals into `snippet.yaml`, then loops on
`agent-snippets validate <id>` and `agent-snippets eval <id>` until they pass.
The agent generates the YAML; compilation does not introduce another workflow
language or a separate compiler service.

Validation checks the package shape, ids, regexes, action definitions, output
paths, eval expectations, and source hash. When `SNIPPET.md` changes, the YAML
is stale until recompiled. Invalid or stale snippets cannot run. Validation
also checks deterministic patterns against the evals and flags risky ones.
Library snippets arrive compiled; after `add`, the agent adapts them to the
repository and repeats validation and evals.

## Actions

Actions replace agent-tests' checks. They run in the order written, with each
action receiving the preceding action's proposed output. Each action has an
`id`, a failure `message`, recovery guidance in `fix`, and at least one of
`fast`, `smart`, or `deep`. Each input starts with its original path and content.

Levels run cheapest first and escalate only an unsettled action. A successful
level stops escalation for that action. An execution error fails it rather
than silently trying a different implementation. Pending outputs from an
unsettled level are hints, not changes passed to the next action.

- **`fast`** uses declarative templates, filesystem operations, naming
  conversions, or regex replacements. A complete deterministic result settles
  the action. A `maybe` pattern provides hints and escalates, as does an input
  the deterministic operation cannot handle when a later level exists.
- **`smart`** uses a decision model such as Jev, Clef, or OpenAI to select among
  declared output variants. Each variant has an `id`, an `ask` statement, and
  an `output`. Optional `bounds: [0.3, 0.8]` mean reject below the lower bound,
  unsure between bounds, and eligible above the upper bound. Exactly one
  eligible variant with no unsure alternatives settles the action; otherwise
  it escalates. The classifier selects content; it does not invent content.
- **`deep`** uses a structured LLM call to generate or transform content from
  `ask`, the snippet instructions, the input, and earlier hints. It returns a
  proposed path and content or an explicit unsure result. `reasoning` is
  optionally `low`, `medium`, or `high`.

If no later level can settle an action, its result is `unsure`. With
`human-tiebreak: true`, a person reviews the proposal; unresolved output is
not applied. An action only needs the levels appropriate to its task.

```yaml
actions:
  - id: formalize-button
    message: A formal button label could not be generated.
    fix: Specify the intended button action in SNIPPET.md.
    human-tiebreak: true
    fast:
      replace:
        match: '^<button>Go!</button>$'
        with: <button>Continue</button>
      maybe: ['\b(hey|awesome|folks)\b']
    smart:
      bounds: [0.3, 0.8]
      variants:
        - id: save
          ask: "The entire input is a button whose sole purpose is saving."
          output:
            content: <button>Save</button>
    deep:
      ask: "Return the complete HTML with a formal button label, preserving its purpose and markup."
      reasoning: medium
```

In this example, a `fast.replace` match settles the action unless a `maybe`
pattern also matches. No replacement match escalates. `output.content`
replaces the input text; omitted `output.path` preserves its current path.
For creation, a `fast.output` supplies literal content and an optional path;
it requires the destination to be absent. Further deterministic operations
must have declared inputs and outputs; arbitrary shell commands and custom
code are outside this declarative format.

For example, a generation snippet can create a file without a model call:

```yaml
actions:
  - id: create-component
    message: The component could not be created.
    fix: Choose a component path that does not already exist.
    fast:
      output:
        content: "<script lang=\"ts\"></script>\n\n<div></div>\n"
```

`run create-component --input src/lib/Card.svelte` uses the proposed path as
the output path. Evals use the same `path` and empty `content` to exercise it.

## Execution and reporting

`run <id> [targetDir]` executes a named snippet within its scope. `--changed`
and `--base <ref>` select changed inputs; `--dry-run` returns proposed changes
without writing them. `--input <path>` supplies a proposed path for generation,
using empty content when the file is absent. The library API also accepts
proposed paths and source text. `targets <id>` previews applicability without
actions. `list` renders installed snippets as a directory tree; `docs` supplies
skills that teach agents to author and run them.

Output paths stay inside the target directory. A transformation may replace
its selected input; creation or renaming into another existing path fails
without overwriting it. Folder creation follows the output path. Inputs run
in stable path order, actions run sequentially, and collisions fail explicitly.
There are no automatic retries of actions that write files.

Each result names the snippet, action, level, input path, output paths, and
outcome: `pass`, `fail`, `unsure`, or `skip`. Failures include the intended
effect, actual outcome, and recovery guidance. CLI output, the Vitest plugin,
and the library expose the same results. A run reports all completed changes
and the input where it stopped; it does not imply rollback of earlier inputs.
An unresolved or failed action stops subsequent actions for that input and
does not publish that input's pending changes.

## Evals

Evals benchmark generated workflows and model changes against approved results.
They live under `evals` in `snippet.yaml` and use the real triggers and actions
in an isolated temporary directory. They never modify the user's files.
`path` and `content` supply invocation inputs. An optional `files` map seeds
existing fixture paths with their text, allowing output-collision evals.
Fixture paths stay inside the temporary directory.

```yaml
evals:
  - name: English button label
    path: src/save.html
    content: <button>Save</button>
    expect: pass
    output:
      path: src/save.html
      content: <button>Enregistrer</button>
  - name: French label remains unchanged
    path: src/save.html
    content: <button>Enregistrer</button>
    expect: pass
    output:
      path: src/save.html
      content: <button>Enregistrer</button>
  - name: Source code is outside scope
    path: src/save.ts
    content: "export const save = () => {};"
    expect: skip
```

`expect` is `pass`, `fail`, `unsure`, or `skip`. Unlike a test's `fail` verdict,
a snippet's `fail` means generation or transformation could not complete.
Every `pass` example declares the expected output path and content; other
outcomes assert no published output. `eval <id>` reports both outcome and
output mismatches. Include successful generation, unchanged content, skipped
inputs, and expected failures such as invalid input. Providers use the
repository's model configuration, so model upgrades exercise real decisions
and generation rather than substituting mocks.

## Community library

A contribution changes one snippet and exactly two files:
`library/<id>/SNIPPET.md` and `library/<id>/snippet.yaml`. Ids are permanent.
The first prose line credits the PR author as
`By [your-handle](https://github.com/your-handle)`; updates preserve the credit.
The YAML must match the id and source hash and contain `pass`, `fail`, and
`skip` evals. Contributors include validation and eval output in the PR,
keeping credentials and run artifacts out of Git.

Library snippets are declarative: no scripts or extra executable files.
Describe only the intended task and outputs; instructions to read secrets,
bypass controls, or manipulate the executing agent are prompt injection.
The [agent-tests library review rules](https://github.com/feuersteiner/agent-tests/blob/964076aed883c352036cc0677b5e94f26abc2304/library/AGENTS.md) carry over: scan both files for hidden
or bidirectional Unicode, HTML comments, scripts, encoded payloads, and links
other than the credit. Hidden content closes a PR with `security-flagged`;
suspected injection requires `security-review` and a maintainer's
`security-cleared`, revoked on any new commit. Confirmed malicious accounts
are banned. These are planned enforcement requirements, not existing checks.

`add <id> [targetDir]` copies the two files from
`feuersteiner/agent-snippets` on `main` into
`<targetDir>/.agents/snippets/<id>/`, records the source commit, prints the
adaptation brief, and runs evals. Every snippet merged into `main` is
available. `update` compares against the recorded commit; removed library
entries are reported while their local copies are kept. `score.csv` is never
installed.

`vote <id> [--up | --down]` uses the signed-in GitHub CLI account to toggle or
set a vote through a PR, following the agent-tests vote flow. Each snippet's
`score.csv` has one `github_handle` column, one unquoted handle per row, no
`@` or blank rows, and case-insensitive uniqueness. A vote PR changes only
one row in one score file, adding or removing only the PR author's own handle.
Pending vote PRs determine the effective vote; a matching request is a no-op,
and reversing a pending vote closes its PR. Otherwise the command creates a
branch in the library repo or the voter's fork and opens a PR without changing
the local working tree. Scores express interest and never gate availability.

Planned `Library add`, `Library update`, and `Library score` workflows run
trusted default-branch checks against PR Git data. Require those checks, an
up-to-date branch, and an approving review on `main`. Valid vote PRs receive
automated approval and auto-merge once required checks pass; conflicting votes
require another `vote` run. Publishing and docs deployment follow the same
package boundaries as agent-tests, adapted to the `agent-snippets` name.

## Repository structure

The planned repository layout matches agent-tests: `core/` for the npm package,
`docs/` for a static Bref-ui documentation site, `library/` for community
snippets, and `.github/workflows/` for docs, publishing, and library checks.
These packages and workflows are not implemented in this specification PR.

These packages and workflows are planned; none is implemented yet.

## Verification

No installation or build is required. Run `git diff --check` and review the
YAML examples against the definitions above. Runtime validation and eval
commands become verification commands once the CLI exists.

## License

[MIT](LICENSE).
