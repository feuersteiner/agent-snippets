# agent-snippets

Turn instructions into reusable workflows that generate content.

An agent compiles `SNIPPET.md` into one `snippet.yaml`: triggers select inputs,
actions generate or transform content, and evals verify the results. The
framework mirrors [agent-tests](https://github.com/feuersteiner/agent-tests).

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

The [specification](SPEC.md) defines compilation, scope, execution levels,
expected outputs, and the community library.

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

## Verify this specification

No installation or build is required. Run `git diff --check` and review the
YAML examples against [SPEC.md](SPEC.md). Runtime validation and eval commands
above become verification commands once the CLI exists.

## License

[MIT](LICENSE).
