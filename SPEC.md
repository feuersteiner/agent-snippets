# Nippet specification

Nippet is short for snippet. It is a CLI for turning instructions written in ordinary language into executable, testable actions.

An AI agent uses the CLI to translate a user's `nippet.md` into a workflow and its step definitions. The generated files can be inspected, tested, and reused.

## Package boundaries

- Nippet owns actions: creating or transforming files, folders, names, and text, and sequencing those operations.
- Agent-tests owns checks: evaluating conventions and reporting whether a target satisfies them.
- Neither CLI requires the other. Nippet still validates its inputs and tests its own actions; these are execution and development requirements, not a standalone repository-checking feature.

Potential shared code includes model-provider adapters, schema-loading utilities, and error serialization. Extract a shared library only when both implementations demonstrate the same need. Shared infrastructure must not introduce a dependency between the CLIs. Its packaging remains undecided.

## Package

```text
create-component/
  nippet.md
  workflow.yaml
  action-create-component.yaml
  <test file>
  templates/
  fixtures/
```

- `nippet.md`: the user's description, in the user's own words, of what the action should do. Similar in purpose to `SKILL.md`.
- `workflow.yaml`: the executable translation of that description. Sequences and combines the steps.
- `action-<kebab-case-name>.yaml`: defines a step that performs an operation.
- Test file: supplies test inputs and expected outcomes to validate the nippet, as a normal `.test.ts` file would. The exact format remains undecided.
- Templates and fixtures: optional supporting files.

## Workflow

The workflow describes what runs and in what order. Step implementation details belong in the separate step files.

Illustrative syntax, pending schema design:

```yaml
steps:
  - action: create-component
```

This entry refers to `action-create-component.yaml`. Workflow syntax must remain readable without implementation knowledge.

## Step levels

Each step can define up to three levels:

| Level | Implementation |
| --- | --- |
| 1 | Deterministic actions: templates, filesystem operations, text transformations, and ordinary code. |
| 2 | Jev / decision models to choose how an action is performed. |
| 3 | An LLM call to generate or transform content for an action. |

Not every step needs all three levels. Model configuration and level implementations belong in the step definition. Conditions and model decisions support an action; standalone checks belong to agent-tests.

The level selection policy is not yet defined. The existence of three levels does not imply automatic fallback. Selection, escalation, and handling of unavailable levels must be specified before implementation.

## Supported work

- Create folders and files, choose filenames, and apply naming conventions such as kebab-case, PascalCase, and camelCase.
- Render text snippet templates, including existing plain-text snippets, default Svelte components, and props.
- Transform text using deterministic code, decision models, or LLM calls.
- Create a proposed file structure.
- Combine and sequence actions into a reusable workflow.

## Authoring and validation

1. The user describes the intended action in `nippet.md`.
2. An AI agent uses the CLI to produce `workflow.yaml`, step files, and tests.
3. The CLI validates the generated package and runs its tests.
4. The agent corrects failures until the package satisfies the expected outcomes.
5. The CLI executes the resulting workflow against supplied inputs.

Every nippet must include tests with explicit expected outcomes. These tests verify the action itself: returned values, generated text, filenames, directory structures, and expected failures. They do not turn the nippet into a reusable repository check. The test isolation strategy and treatment of model calls remain to be specified.

## Failure reporting

Failures must help an AI agent recover: identify the step and level, affected input or file, intended effect, actual outcome, and a useful explanation. Distinguish invalid inputs from an inability to execute the action.

Report any actions already completed before failure. Machine-readable error formatting and exit codes remain to be specified.

## Remaining decisions

- Exact workflow and step schemas, including inputs, outputs, references, and branching.
- How levels are selected and whether escalation is allowed.
- Test file format and how decision-model and LLM behavior is tested.
- CLI commands, model configuration, file collision behavior, and retry semantics.
