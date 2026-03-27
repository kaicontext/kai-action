# Kai Selective CI — GitHub Action

Run [Kai](https://github.com/kailayerhq/kai) semantic change detection in GitHub Actions. Only test what changed.

## Quick Start

```yaml
name: Selective Tests
on: pull_request

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - uses: kailayerhq/kai-action@v1
        id: kai
        with:
          args: '--git-range ${{ github.event.pull_request.base.sha }}..${{ github.sha }} --out plan.json'

      - name: Run selected tests
        if: steps.kai.outputs.mode == 'selective'
        run: |
          echo "Running ${{ steps.kai.outputs.targets-count }} test targets"
          # Use plan.json to drive your test runner
```

## Inputs

| Input     | Required | Default  | Description                                          |
|-----------|----------|----------|------------------------------------------------------|
| `version` | no       | `latest` | Kai version to install (e.g. `v0.3.0`)              |
| `command` | no       | `plan`   | `kai ci` subcommand: `plan`, `ingest`, or `report`  |
| `args`    | **yes**  | —        | Arguments passed to the subcommand                   |

## Outputs

| Output          | Description                                  |
|-----------------|----------------------------------------------|
| `plan-file`     | Absolute path to the generated `plan.json`   |
| `mode`          | Plan mode: `selective`, `full`, or `skip`    |
| `confidence`    | Confidence score (0.0–1.0)                   |
| `targets-count` | Number of test targets selected              |

Outputs are only populated when `command` is `plan`.

## Examples

### Selective RSpec (Rails)

```yaml
- uses: kailayerhq/kai-action@v1
  id: kai
  with:
    args: '--git-range ${{ github.event.pull_request.base.sha }}..${{ github.sha }} --out plan.json'

- name: Run RSpec
  if: steps.kai.outputs.mode != 'skip'
  run: |
    if [ "${{ steps.kai.outputs.mode }}" = "selective" ]; then
      TARGETS=$(jq -r '.targets.run[]' plan.json | tr '\n' ' ')
      bundle exec rspec $TARGETS
    else
      bundle exec rspec
    fi
```

### Selective Jest (Node.js)

```yaml
- uses: kailayerhq/kai-action@v1
  id: kai
  with:
    args: '--git-range ${{ github.event.pull_request.base.sha }}..${{ github.sha }} --out plan.json'

- name: Run Jest
  if: steps.kai.outputs.mode != 'skip'
  run: |
    if [ "${{ steps.kai.outputs.mode }}" = "selective" ]; then
      TARGETS=$(jq -r '.targets.run[]' plan.json | paste -sd '|' -)
      npx jest --testPathPattern="$TARGETS"
    else
      npx jest
    fi
```

### Shadow Mode

Run Kai alongside your full test suite to validate accuracy before switching to selective mode:

```yaml
- uses: kailayerhq/kai-action@v1
  id: kai
  with:
    args: '--git-range ${{ github.event.pull_request.base.sha }}..${{ github.sha }} --out plan.json'

- name: Full test suite (always runs)
  run: bundle exec rspec --format json --out results.json

- name: Ingest coverage
  uses: kailayerhq/kai-action@v1
  with:
    command: ingest
    args: '--coverage coverage/.resultset.json --results results.json'
```

### Coverage Ingestion

After tests complete, ingest coverage data to improve future plans:

```yaml
- name: Ingest results
  uses: kailayerhq/kai-action@v1
  with:
    command: ingest
    args: '--coverage coverage/.resultset.json --results results.json'
```

## How It Works

1. Downloads the pre-built `kai` binary from [GitHub Releases](https://github.com/kailayerhq/kai/releases)
2. Runs `kai ci <command>` with your arguments
3. Parses `plan.json` and sets GitHub Action outputs for downstream steps

No Docker build, no Node.js runtime — just a single binary.

## License

Apache 2.0 — see [LICENSE](LICENSE).
