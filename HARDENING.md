<!-- markdownlint-disable -->

# Hardening Report: hoverkraft-tech--compose-action/v3.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **hoverkraft-tech--compose-action/v3.0.0** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct ${{ }} expression interpolation inside run: blocks. In __check-action.yml, three steps inject matrix-controlled expressions directly into shell commands: (1) `run: echo "absolute=${GITHUB_WORKSPACE}/${{ matrix.compose-file }}" >> "$GITHUB_OUTPUT"` — matrix.compose-file is interpolated directly; (2) a multiline run block containing `OCI_REPOSITORY="${{ matrix.compose-file }}"` and `docker compose -f "${{ matrix.source-compose-file }}" publish ...` — both matrix values are interpolated directly into shell; (3) `run: ${{ matrix.assertion }}` — the entire shell command is a matrix-controlled expression, allowing arbitrary command execution. In __check-dist.yml, `run: ${{ steps.setup-node.outputs.run-script-command }} package` interpolates a step output directly into the run command.

Locations:

- `.github/workflows/__check-action.yml:218`
- `.github/workflows/__check-action.yml:237`
- `.github/workflows/__check-action.yml:239`
- `.github/workflows/__check-action.yml:265`
- `.github/workflows/__check-dist.yml:20`

### github-env-injection (severity: high)

A run: block writes a value derived from an untrusted matrix input directly to $GITHUB_OUTPUT without sanitization. In __check-action.yml, the step 'Resolve absolute compose path' runs: `echo "absolute=${GITHUB_WORKSPACE}/${{ matrix.compose-file }}" >> "$GITHUB_OUTPUT"`. The value `${{ matrix.compose-file }}` is matrix-controlled (workflow-controllable) and is written to $GITHUB_OUTPUT without applying the required `printf '%s' ... | tr -d '\n\r'` sanitization step.

Locations:

- `.github/workflows/__check-action.yml:218`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all four injection points across two workflow files:

1. `.github/workflows/__check-action.yml` - 'Resolve absolute compose path' step: moved `matrix.compose-file` to env var `COMPOSE_FILE` and added `printf '%s' "$COMPOSE_FILE" | tr -d '\n\r'` sanitization before writing to $GITHUB_OUTPUT (fixes both script-injection and github-env-injection).

2. `.github/workflows/__check-action.yml` - 'Arrange: publish compose application as OCI artifact' step: moved `matrix.compose-file` and `matrix.source-compose-file` to env vars `COMPOSE_FILE` and `SOURCE_COMPOSE_FILE`, referenced as plain shell variables.

3. `.github/workflows/__check-action.yml` - assertion step: moved `matrix.assertion` to env var `ASSERTION` and used `eval "$ASSERTION"` instead of direct `run: ${{ matrix.assertion }}` interpolation.

4. `.github/workflows/__check-dist.yml` - 'Build dist/ Directory' step: moved `steps.setup-node.outputs.run-script-command` to env var `RUN_SCRIPT_COMMAND` and referenced as `$RUN_SCRIPT_COMMAND package` in the shell script.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:
1. hardened/action/.github/workflows/__check-dist.yml line 26: Quoted `$RUN_SCRIPT_COMMAND` → `"$RUN_SCRIPT_COMMAND"` to prevent unquoted variable expansion of a workflow-controllable steps output.
2. hardened/action/.github/workflows/__check-action.yml line 280: Replaced `eval "$ASSERTION"` with writing the assertion to a mktemp file via `printf '%s' "$ASSERTION"` and executing it with `bash`, eliminating the eval-based code injection risk from the workflow-controllable `matrix.assertion` context.

