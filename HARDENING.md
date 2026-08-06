<!-- markdownlint-disable -->

# Hardening Report: hoverkraft-tech--compose-action/v3.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **hoverkraft-tech--compose-action/v3.1.0** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct ${{ }} expression interpolation inside run: blocks. In __check-action.yml, three steps inject matrix-controlled expressions directly into shell commands:
1. `run: echo "absolute=${GITHUB_WORKSPACE}/${{ matrix.compose-file }}" >> "$GITHUB_OUTPUT"` — matrix.compose-file is interpolated directly into the shell command.
2. In the OCI publish step: `OCI_REPOSITORY="${{ matrix.compose-file }}"` and `docker compose -f "${{ matrix.source-compose-file }}" publish ...` — both matrix values are interpolated directly into the run block.
3. `run: ${{ matrix.assertion }}` — the entire shell command is a matrix-controlled expression, allowing arbitrary code execution.
In __check-dist.yml: `run: ${{ steps.setup-node.outputs.run-script-command }} package` — a step output is interpolated directly into the run: command string.

Locations:

- `.github/workflows/__check-action.yml:196`
- `.github/workflows/__check-action.yml:228`
- `.github/workflows/__check-action.yml:251`
- `.github/workflows/__check-dist.yml:24`

### github-env-injection (severity: high)

A matrix-controlled value (${{ matrix.compose-file }}) is written directly to $GITHUB_OUTPUT without the required sanitization step (printf '%s' ... | tr -d '\n\r'). The offending line is: `run: echo "absolute=${GITHUB_WORKSPACE}/${{ matrix.compose-file }}" >> "$GITHUB_OUTPUT"`. An attacker who controls the matrix.compose-file value (e.g. via a crafted workflow_call) could inject newlines to poison GITHUB_OUTPUT with arbitrary key-value pairs.

Locations:

- `.github/workflows/__check-action.yml:196`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all script-injection and github-env-injection findings:
1. __check-action.yml 'Resolve absolute compose path': moved matrix.compose-file to env.COMPOSE_FILE and sanitized with tr -d '\n\r' before writing to GITHUB_OUTPUT.
2. __check-action.yml OCI publish step: moved matrix.compose-file and matrix.source-compose-file to env vars MATRIX_COMPOSE_FILE and MATRIX_SOURCE_COMPOSE_FILE.
3. __check-action.yml assertion step: moved matrix.assertion to env.ASSERTION and executed via 'bash -e -- <(printf '%s' "$ASSERTION")' using process substitution.
4. __check-dist.yml 'Build dist/ Directory': moved steps.setup-node.outputs.run-script-command to env.RUN_SCRIPT_COMMAND and referenced as $RUN_SCRIPT_COMMAND in the run block.

### Iteration 2

**Fixes applied:** script-injection, hardcoded-credentials

**Notes:**

1. Fixed script-injection in .github/workflows/__check-dist.yml line 24: changed `run: $RUN_SCRIPT_COMMAND package` to `run: "$RUN_SCRIPT_COMMAND" package` to prevent shell metacharacters in the step output from being interpreted as shell commands. 2. Fixed hardcoded-credentials in .github/workflows/main-ci.yml line 47: replaced the literal Codecov badge token `90JXB7EIMA` in the URL with `${{ secrets.CODECOV_TOKEN }}` so the credential is stored as a repository secret rather than hardcoded in the workflow file.

