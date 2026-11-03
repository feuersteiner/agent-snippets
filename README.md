# agent-snippets

Turn instructions into reusable CLI workflows.

[Join the Discord community](https://discord.gg/E7hsgN7pdW).

Describe how something should be made once. An agent turns that description
into a snippet that creates folders, names files, renders templates, makes
decisions, and generates content. Run it again with different inputs, keeping
the conventions and workflow that made the first result useful.

A snippet can scaffold a UI component, set up an API route, organize a folder,
or turn a rough brief into content. It combines the repeatability of a text
snippet with an agent's ability to interpret a task.

**Status:** specification stage; no CLI is implemented or published yet.

## Example: add a UI component

Describe the convention in `.agents/snippets/ui-component/SNIPPET.md`:

```md
Add a component to the UI package from a supplied name. Use a kebab-case
folder and .svelte filename, with types.ts beside the component. Export a
PascalCase props interface: dashboard-navbar becomes DashboardNavbarProps.
Leave /* WIP: insert props here */ inside the interface. Import that type
into the component, use it for $props(), and add WIP comments and placeholder
text. Create both files together; do not overwrite existing files.
```

The agent compiles this into `snippet.yaml`. The scaffold is predictable,
so reusable templates can produce it without a model call on every run:

```yaml
id: ui-component
title: Add a UI component
description: Create a Svelte component and its exported props interface.
source: sha256:46343a3392c9ec57e89b2d45a62ffeb562674cab17cedf012cebc946a2d867e1
inputs:
  name:
    type: string
    required: true
  directory:
    type: path
    default: packages/ui/src/lib
actions:
  - id: scaffold-component
    fast:
      output:
        files:
          "{{ inputs.directory }}/{{ inputs.name | kebabCase }}/types.ts": |
            export interface {{ inputs.name | pascalCase }}Props {
              /* WIP: insert props here */
            }
          "{{ inputs.directory }}/{{ inputs.name | kebabCase }}/{{ inputs.name | kebabCase }}.svelte": |
            <script lang="ts">
              import type { {{ inputs.name | pascalCase }}Props } from './types';

              // WIP: use the component props.
              let props: {{ inputs.name | pascalCase }}Props = $props();
            </script>

            <nav>WIP: implement {{ inputs.name | pascalCase }}</nav>
```

The planned command:

```sh
npx agent-snippets run ui-component --name "Dashboard Navbar"
```

produces:

```text
packages/ui/src/lib/dashboard-navbar/
  dashboard-navbar.svelte
  types.ts
```

`types.ts`:

```ts
export interface DashboardNavbarProps {
  /* WIP: insert props here */
}
```

`dashboard-navbar.svelte`:

```svelte
<script lang="ts">
  import type { DashboardNavbarProps } from './types';

  // WIP: use the component props.
  let props: DashboardNavbarProps = $props();
</script>

<nav>WIP: implement DashboardNavbar</nav>
```

Run it with another name to create another component using the same convention.
A snippet that also designs the component from a brief could follow this
scaffold with a generation step.

## Another example: set up an API route

An `api-route` snippet takes a route name and scaffolds a SvelteKit endpoint,
a handler, and a response type. For `--name dashboard`, it creates:

```text
apps/web/src/routes/api/dashboard/
  +server.ts
  handler.ts
  types.ts
```

Its compiled action can use templates too. This excerpt assumes a required
string input named `name`:

```yaml
actions:
  - id: scaffold-route
    fast:
      output:
        files:
          "apps/web/src/routes/api/{{ inputs.name | kebabCase }}/+server.ts": |
            import { json } from '@sveltejs/kit';
            import { handleGet } from './handler';

            // WIP: return the appropriate status once implemented.
            export const GET = () => json(handleGet(), { status: 501 });
          "apps/web/src/routes/api/{{ inputs.name | kebabCase }}/handler.ts": |
            import type { {{ inputs.name | pascalCase }}Response } from './types';

            export function handleGet(): {{ inputs.name | pascalCase }}Response {
              // WIP: fetch and return the route's data.
              return { message: 'WIP: implement {{ inputs.name | pascalCase }}' };
            }
          "apps/web/src/routes/api/{{ inputs.name | kebabCase }}/types.ts": |
            export interface {{ inputs.name | pascalCase }}Response {
              message: string;
              /* WIP: insert response fields here */
            }
```

The placeholder returns HTTP 501 until implemented. A richer workflow could
choose an authentication pattern, generate a handler from a brief, and then
write the related files in order.

## Authoring and reuse

1. Describe the task, conventions, inputs, and intended outputs in `SNIPPET.md`.
2. An agent generates `snippet.yaml`, including reusable actions and examples.
3. Validate the workflow and try its examples in an isolated directory.
4. Inspect the proposed files with `run --dry-run`, then run with real inputs.
5. Refine the instructions and recompile as the workflow improves.

`compile <id>` gives the calling agent the instructions, a snippet-authoring
skill, and the project layout. The agent writes and corrects the YAML using
`validate` and `eval`. The CLI provides the tools; the agent translates the
task into the workflow.

`SNIPPET.md` remains the human-readable source. `snippet.yaml` contains the
executable workflow and can also be edited by hand. Its `source` records
`sha256:<SHA-256 of the exact SNIPPET.md bytes>`. Validation detects changed
instructions and requires recompilation before execution.

## Inputs and templates

Inputs describe what varies between runs: a component name, destination,
source text, or a generation brief. Each input declares its type, whether it
is required, and an optional default. CLI flags use these input names; the
library API accepts the same values as an object.

Templates interpolate `{{ inputs.name }}`. The `kebabCase` and `pascalCase`
filters turn names such as `Dashboard Navbar` into `dashboard-navbar` and
`DashboardNavbar`. Templates work in output paths and file contents. Missing
inputs, invalid paths, and names that cannot produce valid filenames or
identifiers are validation errors.

Paths are relative to the directory where the snippet is installed, or an
explicit target directory supplied when running it. A new file needs no
existing input file to activate generation.

## Workflow actions

Actions run in the declared order and share inputs and a pending set of files.
Use the execution method suited to each action:

- **`fast`** renders templates and performs deterministic file, naming, or
  text operations. Use it when the result follows directly from the inputs.
- **`smart`** uses a decision model to choose a declared variant or workflow
  branch, such as which authentication scaffold fits a route. Each choice
  describes its applicability and the action or output it selects.
- **`deep`** calls an LLM to generate or transform content from instructions
  and context, such as implementing a component from a design brief. It
  returns proposed files. Optional `reasoning` is `low`, `medium`, or `high`.

These are ways to perform work. A template action finishes after rendering;
a model decision selects the next work; a generation action produces content.
The workflow explicitly defines any branching or fallback. An execution error
stops the run rather than silently selecting another implementation.

`output.files` maps destination paths to complete file contents. For `fast`,
paths and content can be templates; generation steps return concrete paths
and content. Later actions can use the pending files as context and refine
them before the workflow writes its final result.

Creation fails if any destination already exists. Transformations must
explicitly identify the existing files they intend to change. Output paths
stay within the target directory, and collision checks happen before writing.
`--dry-run` shows the proposed file tree and changes without applying them.

Failures identify the action, affected paths, intended effect, actual outcome,
and recovery guidance. Any files already written are reported so a retry can
account for them. Pending content from an unfinished action is not published.

## Verify a snippet

Examples demonstrate that the workflow produces the intended artifacts.
They live under `evals` in `snippet.yaml`; `eval <id>` runs them in temporary
directories and compares the generated files with the expected results.

For example, the UI snippet can include this collision case:

```yaml
evals:
  - name: Preserve existing component types
    inputs:
      name: Dashboard Navbar
    files:
      packages/ui/src/lib/dashboard-navbar/types.ts: "export interface ExistingProps {}\n"
    expect:
      error: output-conflict
```

Successful examples supply `expect.files` with concrete output paths and
contents, including the component and its props interface. Failure examples
supply `expect.error` and assert that existing files remain unchanged. The
agent validates both the ordinary task and relevant edge cases before reuse.
Model-backed actions use the configured provider during evals so their
outputs can be reviewed when instructions or models change.

## Get started (WIP)

Planned commands; these are not available yet:

```sh
npm install -D agent-snippets
npx agent-snippets init                         # configuration and local snippets
npx agent-snippets add ui-component             # install a reusable workflow
npx agent-snippets compile ui-component         # agent generates snippet.yaml
npx agent-snippets validate ui-component        # structure, inputs, and source hash
npx agent-snippets eval ui-component            # try the workflow's examples
npx agent-snippets run ui-component --name "Dashboard Navbar" --dry-run
npx agent-snippets run ui-component --name "Dashboard Navbar"
npx agent-snippets list                         # available snippets
npx agent-snippets docs                         # authoring and execution guidance
```

The CLI runs named workflows. Agents can invoke it while working; scripts or
other tools can invoke the same workflows through the CLI or package API.
Choose providers in `agent-snippets.config.yaml` for workflows that need
models, and keep credentials in `.env`.

## Sharing snippets

A reusable package has two files:

```text
.agents/snippets/<id>/
  SNIPPET.md
  snippet.yaml
```

Any project directory can have its own `.agents/snippets/`. Snippets are
available there and in child directories; local configuration can disable
specific inherited snippets. `add <id> [targetDir]` installs a community
snippet in the chosen directory, then gives the agent an adaptation brief.
`update` compares against the recorded library commit and preserves a local
copy if its upstream entry disappears.

Community snippets live in `library/<id>/` in `feuersteiner/agent-snippets`.
Contributions include the instructions, compiled workflow, credited author,
and examples showing the generated results and expected errors. Review checks
that the workflow does its stated task and contains no hidden or unrelated
instructions. Generated code is content; the runner does not execute it as
part of rendering a template.

Every snippet merged into `main` is available to install. Planned
`vote <id> [--up | --down]` records interest through a GitHub PR changing only
the signed-in user's own row in the snippet's `score.csv`. Votes do not gate
availability.

The planned repository layout is `core/` for the npm package, `docs/` for its
documentation, and `library/` for shared workflows. Agent-tests remains an
independent project for repository checks.

## Verify this README

No installation or build is required. Run `git diff --check`, parse the YAML
examples, and inspect the generated component and route code. The CLI commands
above become runtime verification commands once the CLI exists.

## License

[MIT](LICENSE).
