# agent-snippets

Turn recurring agent tasks into reusable file generators.

Describe a convention once. An agent turns it into templates. Run them with new
inputs to create components, API routes, or project scaffolds with the same
structure and naming every time. The agent authors the recipe; the runner renders
the files without a model call.

**Status:** WIP specification. No CLI or API is implemented or published yet.

## Example

Every Svelte component needs a folder, a component file, and a props interface.
Describe the convention in `.agents/snippets/ui-component/SNIPPET.md`:

```md
Create a Svelte component under src/lib. Use a kebab-case folder and filename.
Put its PascalCase props interface in types.ts beside it. Leave implementation
placeholders. Take the component name as input.
```

An agent writes `snippet.yaml` beside it:

```yaml
inputs: [name]
files:
  'src/lib/{{ name | kebabCase }}/types.ts': |
    export interface {{ name | pascalCase }}Props {
      /* WIP: add props */
    }
  'src/lib/{{ name | kebabCase }}/{{ name | kebabCase }}.svelte': |
    <script lang="ts">
      import type { {{ name | pascalCase }}Props } from './types';
      let props: {{ name | pascalCase }}Props = $props();
    </script>

    <div>WIP: implement {{ name }}</div>
```

The planned command:

```sh
npx agent-snippets run ui-component --name "Dashboard Navbar"
```

creates:

```text
src/lib/dashboard-navbar/
  dashboard-navbar.svelte
  types.ts
```

Run it with another name to create another component using the same convention.

## The v1 format

A snippet lives in the repository-root `.agents/snippets/<id>/`. The folder name
is its ID; `SNIPPET.md` describes the task and `snippet.yaml` defines the output.
YAML has just two fields:

- `inputs`: a list of required string input names; use `[]` for none.
- `files`: a map of output paths to complete file contents.

Paths and contents accept `{{ name }}`, `{{ name | kebabCase }}`, and
`{{ name | pascalCase }}`. For `Dashboard Navbar`, the filters produce
`dashboard-navbar` and `DashboardNavbar`. All other text is literal.

Output paths are relative to the repository root. Missing or unknown inputs,
invalid templates, paths outside the root, duplicate output paths, and existing
destinations fail before writing. Parent directories are created as needed.
`--dry-run` shows the rendered paths and contents without writing.

## Author and run (planned)

```sh
npx agent-snippets init                         # create .agents/snippets/
# Write .agents/snippets/ui-component/SNIPPET.md, then:
npx agent-snippets compile ui-component         # give the agent authoring guidance
npx agent-snippets validate ui-component        # check the YAML and templates
npx agent-snippets run ui-component --name "Dashboard Navbar" --dry-run
npx agent-snippets run ui-component --name "Dashboard Navbar"
```

`compile` supplies the instructions and project context to the calling agent,
which writes the YAML. Review the preview before reuse. YAML can also be written
by hand. Identical templates and inputs produce identical paths and contents.

V1 covers local file generation. Composition, existing-file edits, commands,
model-generated content during execution, and a community library come later.

Planned layout: `core/` for the package and CLI, `docs/` for documentation, and
`library/` for future community snippets.

[Discord](https://discord.gg/E7hsgN7pdW) · [MIT license](LICENSE)
