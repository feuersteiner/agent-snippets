# agent-tests

Turn recurring agent mistakes and review lessons into reusable repository checks.

Learn a rule during an agent session, save it as a check, and verify future changes against it. Each result should explain what was checked, what was found, and what needs attention.

**Status:** design stage. No CLI implementation is available yet. The behavior below describes the intended product; file formats and command syntax remain undecided.

## Scope

Agent-tests is a CLI for checks only. Checks inspect a repository, selected files, or a proposed change without modifying the target.

Examples:

- Component filenames follow PascalCase.
- Files under billing do not import UI utilities.
- API handlers validate inputs before using them.
- A component directory contains the required files.
- User-facing text is written in French.
- A change preserves a rule learned during an earlier review.

Checks return findings and recovery guidance. Creating files, rewriting content, and applying fixes belong to actions outside agent-tests.

## Workflow

1. An agent or reviewer identifies a rule worth preserving.
2. An agent translates the rule into a reusable check with examples and expected outcomes.
3. The check is validated against those examples before reuse.
4. Relevant checks are selected for a repository area, task, or change.
5. The CLI executes them and reports results with supporting evidence.
6. New review lessons become additional checks or improvements to existing ones.

The intended CLI covers installing, validating, running, and reporting checks. Optional sharing through a GitHub registry remains under consideration.

## Check definitions

A check needs:

- A stable name and a plain-language description of the rule.
- An applicability scope, such as paths, repository areas, or types of changes.
- The inputs and execution method needed to evaluate it.
- Explicit pass, fail, and any warning criteria.
- Evidence to include in a finding and guidance for addressing a failure.
- Test fixtures with expected results, including passing and failing examples.

The exact package layout and definition format remain open.

## Execution

Checks may use deterministic logic such as regex, text matching, structural inspection, or scripts; Jev / decision models; or an LLM evaluator. A check only needs the methods appropriate to its rule.

Deterministic regex construction from structured rules and literal regex written by an agent are both supported design goals. Either form needs examples that validate its behavior.

Model-based checks need explicit evaluation criteria. Model configuration, method selection, escalation, and handling uncertain results remain to be specified; automatic fallback is not assumed.

Existing test frameworks may supply evidence through adapters or commands where their execution can preserve the read-only target contract.

## Results

| Result | Meaning |
| --- | --- |
| Passed | The check applied and the target met the rule. |
| Failed | The check applied and found a violation. |
| Warning | The check applied and found an explicitly defined advisory issue. |
| Irrelevant | The check did not apply to the target or change. |

An execution error is separate from these results. Missing inputs, unavailable providers, and invalid evaluator responses must not be reported as passing checks. Uncertainty must be reported explicitly rather than treated as success or irrelevance.

Results should be available to people and agents. Findings identify the check, affected paths or locations, expected behavior, observed evidence, and useful correction guidance. Output schemas and exit codes remain undecided.

## Relationship to nippet

**Agent-tests checks; nippet acts.** Agent-tests evaluates conventions and changes. Nippet creates or transforms files, folders, names, and text.

Neither CLI requires the other. Both may eventually consume shared infrastructure for model adapters, YAML/schema utilities, and structured errors. No shared library exists yet; extraction should follow demonstrated implementation needs.

## Setup and verification

There are no installation, build, or test commands yet. Executable setup and verification commands will be documented here when implementation begins.
