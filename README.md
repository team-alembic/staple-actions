# staple-actions

A collection of reusable GitHub Actions and workflows for Elixir projects,
maintained by [Alembic](https://github.com/team-alembic). Each action wraps a
common Mix task (compile, test, credo, dialyzer, docs, hex.publish, …) with
sensible defaults and aggressive caching of `deps/`, `_build/`, `HEX_HOME`,
and `MIX_HOME`.

## Repository Layout

```
actions/     # individual composite actions
workflows/   # opinionated end-to-end workflows that wire the actions together
```

## Two Flavors: Self-Contained vs. Composable

Every task action is published twice:

- `actions/<name>` — **self-contained**. Installs Erlang/Elixir, restores
  caches, runs `mix deps.get`, compiles, then runs the task. Safe to use on
  its own in a single-job workflow.
- `actions/<name>-composable` — **composable**. Runs only the task itself
  with the correct `MIX_ENV`, `HEX_HOME`, and `MIX_HOME`. Expects setup and
  compilation to have already run earlier in the same job. Use these when
  chaining several checks in one job to avoid re-paying the install/compile
  cost.

Reach for the self-contained form when each check gets its own job (parallel
CI fan-out). Reach for the composable form when you are stacking checks
inside a single job.

## Available Actions

Setup & dependency management:

- `install-elixir` — installs Erlang + Elixir per `.tool-versions` via
  [`erlef/setup-beam`](https://github.com/erlef/setup-beam).
- `mix-deps-get` / `mix-deps-get-composable` — fetch Hex deps with a
  `mix.lock`-keyed cache.
- `mix-deps-unlock` / `-composable` — `mix deps.unlock --check-unused`.
- `mix-hex-audit` / `-composable` — `mix hex.audit`.

Compile & quality checks:

- `mix-compile` / `-composable` — `mix compile` with a `_build` cache.
- `mix-format` / `-composable` — `mix format --check-formatted`.
- `mix-credo` / `-composable` — `mix credo --strict`.
- `mix-doctor` / `-composable` — `mix doctor --full --raise`.
- `mix-check` / `-composable` — `mix check` (runs an aggregated check
  pipeline).
- `mix-sobelow` / `-composable` — `mix sobelow --config`.
- `mix-dialyzer` / `-composable` — `mix dialyzer` with PLT caching.
- `mix-dialyzer-plt` / `-composable` — build only the Dialyzer PLT.

Test, docs, release:

- `mix-test` / `-composable` — `mix test`.
- `mix-docs` / `-composable` — `mix docs` (e.g. for publishing to GitHub
  Pages).
- `mix-hex-publish` / `-composable` — `mix hex.build` + `mix hex.publish
  --yes` using `hex-api-key`.
- `git-ops` / `-composable` — `mix git_ops.release` for automated release
  tagging.
- `conventional-commit` / `-composable` — `mix git_ops.check_message`
  against the PR head commit.

Generic escape hatch:

- `mix-task` / `-composable` — run an arbitrary `mix <task>` (optionally
  compiling first) when no dedicated action exists.

Deploy:

- `gigalixir-deploy` — deploy to [Gigalixir](https://www.gigalixir.com/) and
  optionally run `mix ecto.migrate`.

## Common Inputs

Most actions accept the same set of inputs:

| Input               | Purpose                                                                  | Default                |
| ------------------- | ------------------------------------------------------------------------ | ---------------------- |
| `mix-env`           | Mix environment for the step (usually `test` or `dev`)                   | required               |
| `working-directory` | Directory to run commands from                                           | `${{ github.workspace }}` |
| `use-cache`         | Toggle `actions/cache` usage                                             | `true`                 |
| `cache-prefix`      | Prefix for cache keys (useful in monorepos with multiple apps)           | `""`                   |
| `git-token`         | Token used to rewrite `github.com` URLs so private git deps can resolve  | `${{ github.token }}`  |

Individual actions add their own inputs — e.g. `mix-hex-publish` requires
`hex-api-key`, `gigalixir-deploy` requires email/password/app-name,
`mix-task` requires `task`.

## Usage

Pin to `@main` (what the repo currently uses) or to a tagged release.

### Single-job fan-out (self-contained actions)

```yaml
jobs:
  credo:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: team-alembic/staple-actions/actions/mix-credo@main
        with:
          mix-env: test
```

### Chained checks in one job (composable actions)

```yaml
jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: team-alembic/staple-actions/actions/mix-compile@main
        with:
          mix-env: test
      - uses: team-alembic/staple-actions/actions/mix-format-composable@main
        with:
          mix-env: test
      - uses: team-alembic/staple-actions/actions/mix-credo-composable@main
        with:
          mix-env: test
      - uses: team-alembic/staple-actions/actions/mix-test-composable@main
        with:
          mix-env: test
```

## Ready-Made Workflows

`workflows/` contains end-to-end pipelines you can copy into
`.github/workflows/` of a consuming project:

- [`elixir_lib.yml`](workflows/elixir_lib.yml) — CI for an Elixir library:
  deps, audit, compile, format, credo, doctor, test, dialyzer, conventional
  commits, sobelow, unused deps, docs to GitHub Pages, and `git_ops.release`
  on `main`.
- [`elixir_lib_release.yml`](workflows/elixir_lib_release.yml) — publishes
  the package to Hex on GitHub release. Requires a `HEX_API_KEY` secret.
- [`staple.yml`](workflows/staple.yml) — CI + Gigalixir deployment for an
  Elixir application (includes a Postgres service for tests). Requires
  `GIGALIXIR_EMAIL`, `GIGALIXIR_PASSWORD`, `GIGALIXIR_APP_NAME`, and a
  `PRODUCTION_URL` environment URL.

## Requirements in Consuming Projects

- A `.tool-versions` file at the working directory root declaring `erlang`
  and `elixir` versions (consumed by `erlef/setup-beam`).
- A `mix.lock` (for cache keying).
- For actions that depend on extra Mix tasks: the corresponding dev
  dependency installed — e.g. `credo`, `dialyxir`, `doctor`, `ex_check`,
  `git_ops`, `sobelow`, `ex_doc`.

## License

See [LICENSE](LICENSE).
