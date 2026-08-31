# Spandrel

> Published automatically as the README for
> [spandrel-php/spandrel](https://github.com/spandrel-php/spandrel),
> Spandrel's release repo (the `spandrel.phar` and `spandrel/spandrel`
> Composer package live there) — source, issues, and pull requests
> live in [spandrel-php/spandrel-src](https://github.com/spandrel-php/spandrel-src).

Spandrel checks that your codebase's actual dependencies match the
architecture you've written down — a Markdown file defines named
**layers** (groups of namespaces) and **rules** (allowed/forbidden
dependencies between them), and `analyse` reports every dependency
edge that breaks a rule.

This guide is task-oriented: how to run Spandrel and how to write a
ruleset. For the reasoning behind the design, see Spandrel's own
[`docs/architecture.md`](https://github.com/spandrel-php/spandrel-src/blob/main/docs/architecture.md),
a real ruleset dogfooded on every run.

## Requirements

- PHP 8.5+

## Quick start

Install Spandrel into the project you want to analyse:

```sh
composer require --dev spandrel/spandrel
```

This adds `vendor/bin/spandrel`, backed by a signed `spandrel.phar` —
see [CHANGELOG.md](https://github.com/spandrel-php/spandrel/blob/main/CHANGELOG.md)
for release history.

Or download and run the phar directly, without Composer:

```sh
curl -LO https://github.com/spandrel-php/spandrel/raw/main/spandrel.phar
php spandrel.phar analyse
```

Every release is signed — verify `spandrel.phar.asc` against the
bundled public key before trusting a download (fingerprint
`824682B3BE7DE81A8108FA98CE6235F34D7E5C43`; check it against what
`gpg --import` itself reports, in case this key is ever rotated —
also published on [keys.openpgp.org](https://keys.openpgp.org)):

```sh
curl -LO https://github.com/spandrel-php/spandrel/raw/main/spandrel-bot.gpg.asc
curl -LO https://github.com/spandrel-php/spandrel/raw/main/spandrel.phar.asc
gpg --import spandrel-bot.gpg.asc
gpg --verify spandrel.phar.asc spandrel.phar
```

`init` and `spandrel.yaml`/`architecture.md` auto-discovery are both
relative to the **current working directory** — run these from the
project root:

```sh
vendor/bin/spandrel init
# edit the generated architecture.md to match your namespaces, then:
vendor/bin/spandrel analyse
```

`init` refuses to overwrite existing files unless you pass `--force`.
Without `init`, `--ruleset=path/to/architecture.md` and a `paths`
argument work the same way.

`analyse`'s exit code is `0` (no violations), `1` (violations found), or
`2` (the ruleset or config itself is invalid) — `lint` and the `debug:*`
commands never exit `1`, since they don't evaluate violations.

### Running from source instead

To work on Spandrel itself, or run an unreleased commit, see the
[main README](https://github.com/spandrel-php/spandrel-src/blob/main/README.md)
(Docker, or a local PHP 8.5+/Composer install) and substitute
`bin/spandrel.php` for `vendor/bin/spandrel` above — pointed at the
project you want to analyse, since `init`/`spandrel.yaml` discovery is
relative to the working directory, not this clone. If you're running
via the project's own `docker compose` setup, note the shipped
`compose.yaml` only mounts this clone's own directory into the
container — to analyse a project outside this repo that way, add your
own bind mount for it (or a temporary one via
`docker compose run -v /path/to/project:/target ...`).

## Writing a ruleset

A ruleset is a Markdown file (`architecture.md` by default) with up to
three `##` sections the parser understands — everything else (prose,
extra headings) is ignored, so narrative and rules can live side by
side:

```markdown
# My Project Architecture

## Meta

- Every layer must be used in a rule.

## Layers

- **Domain**: `App\Domain\**`
- **Infrastructure**: `App\Infrastructure\**`

## Rules

- `Domain` must not depend on `Infrastructure`
```

### Layers reference

| Form | Syntax | Use for |
|---|---|---|
| Explicit | `` - **Name**: `pattern`[, `pattern`...] [except ...] `` | A named namespace pattern (or several), the common case. |
| Placeholder | `` - `App\{Name}\**` `` | Derives one layer per distinct namespace segment found in the code — no per-layer bullet needed. Two `{Name}` captures derive a 2D set plus one group layer per axis value. |
| Group | `` - **Name** groups `A` and `B` [except ...] `` | Names a union of already-declared layers (leaf or group). |
| External | `` - **Name** matches `pattern`[, `pattern`...] [except ...] `` | Names a **vendor/third-party** namespace pattern — no `Element` behind it, so it needs nothing added to `source.paths`. Works everywhere a layer name does, except element-kind filters. |

Pattern syntax: backtick-quoted, `\`-separated; `*` matches exactly one
namespace segment, `**` matches zero or more. A pattern with no
wildcard matches exactly one FQCN.

Every element in the Code Graph matches **at most one leaf** layer
(groups and external layers are exempt from this check) — no match is
silently excluded from analysis, more than one match is a load error.

### Rules reference

Grammar: `` - `<Subject>` <verb> `<Object>` ``, one bullet per rule.
Either side can be a [list](#lists) or an [inline pattern](#operands)
instead of a declared layer name.

| Verb | Semantics |
|---|---|
| `must not depend on` | Blacklist — any edge `Subject → Object` is a violation. |
| `may only depend on` | Whitelist — only edges to layers named in a `may only depend on` bullet for this subject are allowed. |
| `depends on nothing` | Sugar for `must not depend on` every other declared layer. No object. |
| `may depend on anything` | Explicitly marks a layer unconstrained (a no-op, but visible — distinct from "nobody wrote a rule for this yet"). No object. |

A layer may carry at most one unscoped verb (of the four above).
Kind-scoped rules (below) can coexist with it — they narrow, never
widen, whatever the unscoped rule already allows.

**Kind-scoped verbs** — restrict which edge *kind* a rule considers,
instead of any dependency:

| Verb pair | Edge kind | Reserved object |
|---|---|---|
| `must not call functions defined in` / `may only call functions defined in` | function calls | `core functions` (PHP builtins) |
| `must not instantiate objects from` / `may only instantiate objects from`, or bare `must not instantiate objects` (no object) | `new` | `core classes` (PHP builtins) |
| `must not throw` / `may only throw` | `throw` | `core classes` |

**`subtypes of`** — object-position only, matches the transitive
`extends`/`implements` closure instead of a namespace:
`` `Domain` may only throw subtypes of `App\Exception\Base` ``. Rejected
on call-scoped rules (functions have no type hierarchy).

**Element-kind filters** — restrict *what the target is*, not which
edge kind: `` `App\**` may only depend on interfaces in `Symfony\**` ``.
Grammar: `(classes|interfaces|traits|enums)[, ...] in <operand>`, valid
on either side of a rule. Needs a real parsed `Element` for the
target — doesn't work against vendor code unless that vendor path is
in `source.paths` (an [external layer](https://github.com/spandrel-php/spandrel-src/blob/main/docs/ruleset.md#external-layers)
alone isn't enough for this one case).

#### Lists

```
- `A`, `B`, or `C` must not depend on `D` and `E`
```

Expands to the Cartesian product of subjects × objects as individual
pairwise rules. `and`/`or` are interchangeable — both just mean "this
set of layers," not logical conjunction/disjunction.

#### Operands

Either side of a rule (or one item in a list) can be:
- a declared layer name (`` `Domain` ``),
- a raw inline pattern, same glob syntax as a layer (`` `App\Foo\**` ``) — matched directly, not registered as a named layer,
- `core functions` / `core classes` (call/instantiate/throw scopes only),
- `subtypes of \`X\`` (object position only).

The **object** (never the subject) can carry a trailing `except`,
itself a list of one or more things to subtract —
`` except `Layer`[, `pattern`...] `` — but not nestable: an excepted
operand can't itself carry another `except`.

### `## Meta`

Optional. Fixed, exact-match sentences that set ruleset-wide policy —
equivalent CLI flags always override what's declared here:

| Sentence | Equivalent to |
|---|---|
| `Any class not in a layer violates rules.` | `analyse --strict` (element-coverage half) |
| `A file that fails to parse violates rules.` | `analyse --strict` (parse-error half) |
| `Every layer must be used in a rule.` | `lint --strict-layers` |

## CLI commands

| Command | Purpose |
|---|---|
| `analyse` (alias `analyze`) | Run the pipeline, report violations. `--report=FORMAT[:OUTPUT]` (console/json/sarif/mermaid), `--baseline`/`--generate-baseline`, `--strict`. |
| `init` | Scaffold a starter `architecture.md` + `spandrel.yaml`. |
| `lint` | Validate the ruleset itself (grammar, layer-resolution sanity) without evaluating violations. `--strict-layers` catches an unused/stale layer. |
| `debug:layers` | Show every declared layer, its pattern(s), and how many elements matched — the first stop when a rule "isn't catching anything." |
| `debug:ruleset` | Print the ruleset with every placeholder/group/list/nullary-predicate expanded to its explicit equivalent. |
| `cache:clear` | Empty the extraction cache. |

Run any command with `--help` for its full option list.

## Project configuration (`spandrel.yaml`)

Discovered automatically at the project root (like `phpstan.neon`),
overridable with `--config`:

```yaml
source:
  paths: ["src"]
  exclude: ["src/Generated"]
ruleset: architecture.md
cache:
  directory: .spandrel-cache
baseline: spandrel-baseline.json
```

| Key | Meaning |
|---|---|
| `source.paths` | Directories Spandrel parses. Only code here gets a real `Element` — everything else (including `vendor/`) is invisible to layer/kind-filter matching, though a plain pattern rule still catches it (see [external layers](https://github.com/spandrel-php/spandrel-src/blob/main/docs/ruleset.md#external-layers)). |
| `source.exclude` | Paths to skip within `source.paths`. |
| `ruleset` | Default ruleset file, overridable with `--ruleset`. |
| `cache.directory` | Where parsed-file results are cached between runs. |
| `baseline` | Default baseline file — suppresses already-known violations so an existing codebase can adopt Spandrel without fixing everything at once. |

## Where to go next

- [`docs/ruleset.md`](https://github.com/spandrel-php/spandrel-src/blob/main/docs/ruleset.md) — the Ruleset file, `## Layers`, and `## Meta` in depth.
- [`docs/cli.md`](https://github.com/spandrel-php/spandrel-src/blob/main/docs/cli.md) — every command and option, in depth.
- [`docs/report.md`](https://github.com/spandrel-php/spandrel-src/blob/main/docs/report.md) — what each `--report` format actually contains.
- [`docs/architecture.md`](https://github.com/spandrel-php/spandrel-src/blob/main/docs/architecture.md) — Spandrel's own ruleset, a real example including a group layer and an external layer.
