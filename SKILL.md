---
name: crystal-development
description: >
  Best practices, style rules, and tooling workflow for Crystal programming language development.
  Use this skill whenever writing, reviewing, or refactoring Crystal source code, working with
  shards, setting up tests, handling CLI argument parsing, working with time/duration types,
  or configuring ameba. Also use when the user mentions .cr files, shard.yml, crystal spec,
  or asks about Crystal idioms, nillability, enums, properties, or stdlib types.
---

# Crystal Development

This skill covers Paul's preferred style and tooling for Crystal (targeting 1.19+/1.20).
For topics marked **→ see reference**, load the relevant file from `references/`.

Quick notes:

* Like ruby, crystal supports a trailing `if` and `unless`, but unlike ruby, it does _not_ support a
  trailing `while` or `until`.

## First Time Working on a Project

When you first start working on a Crystal project that is one of Paul's own — meaning it has **no
git remote at all**, or **every** git remote points at one of these — run the three setup steps
below:

* `yakko:` or `yakko.plambert.net:`
* `github:plambert/*`

Check with `git remote -v`. If there are no remotes, or all of them match the patterns above, the
project is Paul's; if any remote points elsewhere, skip these steps (it's a third-party project).

1. **Pin the VERSION constant.** Wherever `VERSION` is set under `src/**`, replace it with the
   macro form so the version is read from `shard.yml` at build time:

   ```crystal
   {% begin %}
   VERSION = {{ `shards version`.strip.stringify }}
   {% end %}
   ```

   See [Version Number](#version-number) for why the `begin`/`end` wrapper is required.

2. **Install the ameba config.** If `~/.ameba.yml` exists, copy it to `.ameba.yml` at the project
   root. See [Linting with ameba](#linting-with-ameba) for the fallback when it doesn't exist.

3. **Format the Markdown.** If `rumdl` is on the `PATH`, run `rumdl fmt *.md` from the project root.

If the repository has **no commits at all** (`git rev-parse HEAD` fails), commit these changes once
they're made with the message `initial commit`.

## Tooling Workflow

### Formatting

Always format before committing:

```text
crystal tool format src/path/to/file.cr
```

Format the whole project at once with `crystal tool format` from the root (no path argument).

### Linting with ameba

Do _not_ install ameba as a development dependency; use the system-installed version instead,
because it is a more recent build with more features and capabilities, and the build time for ameba
is very long. It should already be in the shell's `PATH`; otherwise it is at
`~/.nix-profile/bin/ameba` on every host.

Always run ameba after formatting. Fix everything it reports **except**
`Metrics/CyclomaticComplexity` on long `case` statements, which should instead be suppressed with a
comment:

```crystal
# ameba:disable Metrics/CyclomaticComplexity
def dispatch(command : Command) : Nil
  case command
  in .start?  then start
  in .stop?   then stop
  # ...
  end
end
```

Every repo needs `.ameba.yml` at the root. Create it if absent, copying the file from ~/.ameba.yml
if it exists, otherwise using:

```yaml
Metrics/CyclomaticComplexity:
  MaxComplexity: 20
```

### Building and testing

```bash
shards install                                            # after clone
shards update                                             # after editing shard.yml
shards build --no-debug --error-trace                     # fast dev build
shards build --no-debug --error-trace --release           # production release build
crystal build --error-trace --release \
              -o bin/foo scratch/foo.cr                   # one-off
crystal spec -v --error-trace                             # run tests, never with `--release`
```

* Create exploratory programs under `scratch/`; promote them to `spec/` tests when they stabilize.
* Don't commit `scratch/` to the main branch of a repo; it's fine in a work branch that will
  eventually be removed, but it should only be for ephemeral code.
* Prefer **Spectator** over stdlib Spec. → see `references/testing.md`

---

## Version Number

The default crystal scaffold has:

```crystal
  VERSION = "0.1.0"
```

The version number should instead be pulled from the `shard.yml`:

```crystal
  {% begin %}
  VERSION = {{ `shards version`.strip.stringify }}
  {% end %}
```

The `shards` command looks for the `shard.yml` file in the current directory, and then searches
upward until it finds it. Don't assume this won't work when not in the root directory.

The begin/end macros are needed so that when `crystal docs` is run, it doesn't show the macro as
the value of the VERSION constant, but instead the stringified version.

Make this change on every crystal project.

## Properties and Attribute Macros

Always use a macro (`property`, `getter`, `setter`, and their variants) — never bare `@ivar`
declarations.

| Macro | Use when |
|---|---|
| `property foo : T` | Read + write |
| `getter foo : T` | Read-only |
| `setter foo : T` | Write-only |
| `property? closed : Bool` | Bool; generates `#closed?` getter |
| `property! foo : T` | Starts nil, raises on nil access; also creates `#foo?` nilable getter |

`property!` is for values that are set before first use but not during `initialize`. It is unsafe if
the value could be mutated by another Fiber. Use it only when the lifecycle is clear.

---

## Nillability

Never use `.not_nil!`. Use one of these patterns instead.

### `if` block — for brief, local use

```crystal
if thing = self.thing
  thing.do_something   # compiler knows thing is not nil here
end
```

This only works when the value cannot be `false` (i.e., not `Bool | Nil`).

### `Guard` — for methods that hinge on a nilable attribute

Add to `shard.yml`:

```yaml
dependencies:
  guard:
    github: plambert/guard.cr
```

Include in any class/module/struct/enum: `include Guard`

```crystal
# Return nil if @thing is nil or false:
def process?(params)
  thing = guard self.thing
  thing.do_work(params)
end

# Raise if @thing is nil or false:
def process(params)
  thing = guard self.thing { RuntimeError.new "@thing must not be nil" }
  thing.do_work(params)
end

# Allow false values through (only nil triggers the guard):
def process(params)
  thing = guard! self.thing
  thing.do_work(params)
end
```

---

## Naming

* **No single-letter variable names** — not even in small blocks. `i`, `j`, `x`, `n` are all banned.
  Use descriptive names: `item_index`, `column_count`, `byte_value`.
* Exception: `a` and `b` in `sort` block parameters are fine.
* Do not use `_name` with a leading underscore to suppress "unused variable" warnings without a
  comment explaining why the variable needs to be there.

---

## String Interpolation

Interpolation calls `#to_s` implicitly. Never write `"value is #{thing.to_s}"` —
write `"value is #{thing}"`. Using `#inspect` inside interpolation is fine when appropriate.

---

## Exhaustive `case` Statements

Always prefer `case … in … end` (no `else`) over `case … when … end` for type unions and enums.
The compiler will enforce that all cases are covered.

```crystal
# Union type
def format_amount(amount : String | Int32) : String
  case amount
  in String then amount
  in Int32  then " " * amount
  end
end

# Enum — use predicate methods, not == comparisons
case output_format
in .text?       then format_text(io)
in .ansi_color? then format_ansi_color(io)
in .json?       then format_json(io)
end
```

A `Symbol` can be passed where a non-union enum parameter is expected (`:text` for
`OutputFormat::Text`).

---

## In-Place Array/Collection Mutation

When you have an ephemeral intermediate collection (one that was just created by a method call and
isn't shared), prefer in-place mutation methods over allocating a second copy:

```crystal
# ✓ Preferred: sort! mutates the Array returned by .values — no second allocation
some_hash.values.sort!

# ✗ Avoid: .sort allocates a new Array
some_hash.values.sort

# ✓ Also applies to map!, select!, reject!, uniq!, reverse!, shuffle!
records.map! { |record| transform(record) }
```

This is safe when: (a) the array was just created for local use, (b) no other reference holds it,
and (c) the element type is the same before and after (required for `map!`).

---

## Block Arguments in Method Definitions

If a method yields, declare it with `&` as the final parameter. Add a type signature where it
aids clarity without sacrificing flexibility:

```crystal
# Yields an Item, return value ignored
def each_valid(items : Array(Item), & : Item ->) : Nil
  items.each { |item| yield item }
end

# Yields an Item, expects a Bool back (used to filter)
def select_valid(items : Array(Item), & : Item -> Bool) : Array(Item)
  items.select { |item| yield item }
end
```

---

## Time and Duration (Crystal 1.19+)

`Time.monotonic` is **deprecated**. Use `Time::Instant` instead.

```crystal
# Measuring elapsed time
start = Time.instant
do_work
elapsed = start.elapsed   # => Time::Span

# Measuring a block
elapsed = Time.measure { do_work }   # => Time::Span

# Sleep always takes a Time::Span, never a bare number
sleep 500.milliseconds
sleep 2.seconds
sleep 1.5.minutes

# Storing a deadline or start point: use Time::Instant, not Time::Span
property started_at : Time::Instant = Time.instant
```

`Time::Instant` represents a point on the monotonic timeline.
`Time::Span` represents a duration. They are no longer the same type.

---

## CLI Tools

→ see `references/cli.md`

---

## Testing with Spectator

→ see `references/testing.md`

---

## `shard.yml` Dependencies Reference

Common shards used in Paul's projects:

```yaml
dependencies:
  guard:
    github: plambert/guard.cr

development_dependencies:
  spectator:
    gitlab: arctic-fox/spectator
    version: "~> 0.11"
```
