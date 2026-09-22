# shared-mise-tasks

Central repository of mise tasks, tool declarations and linter configuration,
consumed by many application repositories. One task definition serves both a
developer laptop and a GitLab CI pipeline.

Written for whoever picks this up cold, human or agent. It states what the
design is, why each part exists, what has been verified by running it, and the
traps that are easy to walk into.

## The core idea

A task is defined once, in `tasks/`. Everything the task needs travels with it:

- the **tool and its version**, declared inline in the task file header with
  `#MISE tools={}` using the `http` backend;
- the **linter configuration**, in `config/`, reached through a path relative to
  the task script;
- the **script logic** itself.

An application repository pins one git ref of this repository in its `mise.toml`.
That single pin decides the script, the tool and the config for **both** paths:

```
developer laptop:   mise run lint:ruff
GitLab CI:          mise run lint:ruff      (inside templates/mise-task.yml)
```

The CI job is a wrapper. It contains no tool name, no version and no lint flags,
so the pipeline cannot drift from what a developer runs locally.

## Repository layout

```
tasks/lint/ruff        mise file task; declares ruff via #MISE tools={} (http backend)
tasks/lint/python      older variant kept for comparison: tool stub + PATH fallback
bin/ruff               mise tool-stub used by tasks/lint/python
config/ruff.toml       shared ruff defaults
templates/mise-task.yml  GitLab CI/CD Catalog component that runs any task in tasks/
mise.toml              makes this repo run its own tasks (includes = ["tasks"])
tests/fixture.py       clean Python file the CI self-test lints
.gitlab-ci.yml         self-test + release-on-tag (publishes templates/ to the Catalog)
```

## How a consuming repository wires it up

`mise.toml` in the application repo:

```toml
[settings]
experimental = true          # remote git task includes are experimental in mise

[task_config]
includes = [
    "git::https://gitlab.example.com/platform/shared-mise-tasks.git//tasks?ref=v1.0.0",
    ".mise/tasks",           # local tasks, last entry wins, so a repo can override
]
```

`.gitlab-ci.yml` in the application repo:

```yaml
include:
  - component: gitlab.example.com/platform/shared-mise-tasks/mise-task@v1.0.0
    inputs:
      job_name: "lint:ruff"
      task: "lint:ruff"
      args: "--output-format=gitlab --output-file=gl-code-quality-report.json"

"lint:ruff":
  artifacts:
    when: always
    reports:
      codequality: gl-code-quality-report.json
```

The component version and the `ref=` in `mise.toml` are bumped together in one
merge request. GitLab-specific extras (reports, artifacts, coverage regexes) are
overrides in the consuming project, never in the task, so the task keeps working
on a laptop with no GitLab around.

## Invariants: do not break these

1. **The CI job runs exactly `mise run <task>`.** The moment a lint flag or a
   tool version appears in YAML, local and CI can differ, and the whole design
   is pointless.
2. **Pin a tag, never `ref=main`.** mise caches a remote include and does not
   re-fetch it (see Trap 1). CI containers are fresh every run, laptops are not,
   so `main` means the pipeline and the developer silently run different code.
3. **Tool versions live in the task header, not in the consuming repo.** A task
   that needs a tool declares it; a consuming repo declaring the same tool is a
   smell, and its version loses anyway (see Verified fact 3).
4. **Config resolution order is local-wins.** `$ROOT/.lint/<tool>.toml` if
   present, otherwise `$SHARED/config/<tool>.toml`.

## Anatomy of a task file

`tasks/lint/ruff`, trimmed:

```bash
#!/usr/bin/env bash
#MISE description="Lint Python code with ruff (inline http tool, no stub)"
#MISE tools={ "http:ruff" = { version = "0.15.22", platforms = { "macos-arm64" = { url = "...", checksum = "sha256:...", size = "..." }, ... } } }
set -euo pipefail

SHARED="$(cd "$(dirname "$0")/../.." && pwd)"
ROOT="${MISE_PROJECT_ROOT:-$PWD}"

if [ -f "$ROOT/.lint/ruff.toml" ]; then
    CONFIG="$ROOT/.lint/ruff.toml"
else
    CONFIG="$SHARED/config/ruff.toml"
fi

cd "$ROOT"
ruff check --config "$CONFIG" "$@" .
```

`SHARED` works because mise clones the **whole** repository into its include
cache. The `//tasks` fragment in the include URL only selects which directory is
scanned for tasks; `config/` and `bin/` are siblings on disk at runtime.

### Adding a tool to a task header

The header is TOML inside a comment, and it must be a single inline table on one
line. Get the checksums without downloading anything:

```bash
curl -s https://api.github.com/repos/<owner>/<repo>/releases/tags/<tag> \
  | python3 -c "import json,sys;[print(a['name'],a.get('digest'),a['size']) for a in json.load(sys.stdin)['assets']]"
```

In an air-gapped environment, point the URLs at the internal mirror instead and
compute checksums there.

## Traps, all of them hit for real in this repo

**Trap 1: the remote include cache is never refreshed.** mise clones the shared
repo to `~/Library/Caches/mise/remote-git-tasks-cache/<hash>/` (Linux:
`~/.cache/mise/...`). With `ref=main`, a push to the shared repo is invisible
locally until that directory is deleted. Symptom: a task you just edited and
pushed behaves like the old version, or a new task does not appear in
`mise tasks`. Fix while developing: `rm -rf` the cache directory. Fix for users:
pin tags and bump them.

**Trap 2: the task header is rendered by mise's template engine before the
backend sees it.** An `http` backend URL containing the backend's own
`{{version}}` placeholder fails with:

```
error: Variable `version` is not defined. Available variables: config_root, cwd, env, mise_bin, ...
```

Escape it: `{% raw %}{{version}}{% endraw %}`. `mise.toml` needs no escape; only
task headers do.

**Trap 3: do not spread the tool across dotted keys.** This parses to nothing,
installs nothing and reports no error, only a `KdlError` warning:

```
#MISE tools."http:ruff".version = "0.11.0"
```

Use one inline table. A quoted key inside the inline table is fine.

**Trap 4: tool stubs plus a PATH check are fragile.** `tasks/lint/python` does
`command -v ruff` and falls back to `bin/ruff`. In a project with a mise shim on
PATH but no ruff version configured, `command -v ruff` succeeds and the task dies
with `mise ERROR No version is set for shim: ruff`. The inline-header approach in
`tasks/lint/ruff` has no such failure mode. Prefer it; `tasks/lint/python` is kept
only as the comparison case.

**Trap 5: `MISE_DATA_DIR` inside the project directory gets linted.** CI sets
`MISE_DATA_DIR=$CI_PROJECT_DIR/.mise` so GitLab can cache installs. `config/ruff.toml`
therefore has `exclude = [".mise"]`. A consuming repo with its own
`.lint/ruff.toml` must exclude it too.

**Trap 6: private shared repo needs auth for the clone inside CI.** The component
rewrites the URL in `before_script`:

```bash
git config --global url."https://gitlab-ci-token:${CI_JOB_TOKEN}@${CI_SERVER_HOST}/".insteadOf "https://${CI_SERVER_HOST}/"
```

The consuming project must be on this project's job token allowlist
(Settings > CI/CD > Job token permissions). Alternatives: make the shared repo
internal, or use a deploy key in a masked CI/CD variable.

## Verified facts

Run on macOS arm64, mise 2026.9.11, against the real repository.

1. An `http` backend tool with a per-platform `url`/`checksum` map **does** work
   in a `#MISE tools={}` header. `mise run lint:ruff` installs
   `http:ruff@0.15.22`, verifies the checksum and runs it.
2. Config resolution works both ways: with no local config the task reports
   `config: <shared clone>/config/ruff.toml`; with `.lint/ruff.toml` present it
   reports the local path, and the local `line-length` is the one enforced.
3. **The task header wins over the consuming repo's `[tools]`.** The application
   repo declares `ruff = "0.12.0"` and has it installed; the task still ran
   `~/.local/share/mise/installs/http-ruff/0.15.22/ruff`, `ruff 0.15.22`.
4. The whole repository lands in the include cache, so `config/` is reachable
   from a task at `$(dirname "$0")/../..`.
5. `mise run` auto-trusts its active config in normal mode, so CI needs no
   `mise trust`. (`mise tasks` does not; that is why interactive exploration
   prompts for trust and `mise run` does not.)

Not verified: the non-macos-arm64 platform entries in the task headers. Their
checksums come from the GitHub release API `digest` field, not from a download.

## Verifying changes

Locally, against the working copy:

```bash
mise run lint:ruff            # in this repo, lints tests/fixture.py
```

From a consuming repo, after pushing a change here, remember Trap 1:

```bash
rm -rf ~/Library/Caches/mise/remote-git-tasks-cache/*
mise tasks
mise run lint:ruff
```

For the pipeline, `glci` renders and executes the pipeline locally without a
GitLab server:

```bash
glci show                             # pipeline graph, offline
glci run lint:ruff --confirm-docker   # real execution in Docker
```

A `component:` include cannot be resolved by glci without a `GITLAB_TOKEN`; it
prints `skipping component include ... no GitLab token configured` and renders
zero jobs. To exercise the component body locally, inline it into a scratch
`.gitlab-ci.yml` (that is what the local proof run does).

## Adding a new shared task

1. Write `tasks/<group>/<name>`, executable, with `#MISE description` and, if it
   needs a tool, a one-line `#MISE tools={}` header (escape `{{version}}`).
2. Put any config file in `config/` and resolve it local-wins in the script.
3. Lint it: the script must survive `set -euo pipefail` and pass shellcheck.
4. Run it in this repo (`mise run <task>`), then from a consuming repo with the
   include cache cleared.
5. Tag a new version. Consumers bump the component version and the `mise.toml`
   `ref=` in the same merge request.
