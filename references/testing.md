# Testing with Spectator

Paul's preferred test framework is **Spectator** (RSpec-style, feature-rich), not stdlib Spec.

- GitLab: https://gitlab.com/arctic-fox/spectator
- GitHub mirror: https://github.com/icy-arctic-fox/spectator

---

## Setup

### `shard.yml`

```yaml
development_dependencies:
  spectator:
    gitlab: arctic-fox/spectator
    version: "~> 0.11"
```

Run `shards install` after adding this.

### `spec/spec_helper.cr`

```crystal
require "spectator"
require "../src/my_project"
```

Do **not** `require "spec"` — Spectator replaces it entirely.

### Running tests

```bash
crystal spec -v --error-trace
```

The `-v` flag shows each example name; `--error-trace` gives full backtraces on failures.

---

## Structure

Every file must wrap its top-level block with `Spectator.describe`. Nested `describe` and
`context` blocks drop the `Spectator.` prefix.

```crystal
require "./spec_helper"

Spectator.describe MyClass do
  describe "#method_name" do
    context "when condition is true" do
      it "does the expected thing" do
        obj = MyClass.new
        expect(obj.method_name).to eq "expected"
      end
    end

    context "when condition is false" do
      it "returns nil" do
        obj = MyClass.new(active: false)
        expect(obj.method_name).to be_nil
      end
    end
  end
end
```

---

## Matchers

Spectator uses `expect(actual).to matcher` syntax. Common matchers:

```crystal
expect(value).to eq(42)
expect(value).not_to eq(42)
expect(value).to be_nil
expect(value).to be_truthy
expect(value).to be_a(String)
expect(value).to be > 10
expect(value).to be_close(3.14, 0.001)  # float proximity
expect(array).to contain("foo")
expect(array).to contain_exactly("a", "b", "c")   # order-independent
expect(array).to have_attributes(size: 3)
expect(string).to match(/pattern/)
expect(string).to start_with("prefix")
expect(string).to end_with("suffix")
expect { risky_call }.to raise_error(ArgumentError)
expect { risky_call }.to raise_error(ArgumentError, /message pattern/)
expect { value_changes }.to change { obj.count }.by(1)
expect { value_changes }.to change { obj.state }.from(:idle).to(:running)
```

---

## Setup and Teardown Hooks

```crystal
Spectator.describe MyService do
  before_all  { MyService.init_db }
  after_all   { MyService.teardown_db }

  before_each { MyService.reset }
  after_each  { MyService.cleanup }

  around_each do |example|
    MyService.transaction do
      example.run
    end
  end
end
```

---

## `let` and `subject`

`let` defines a memoized value that is re-evaluated fresh for each example.
`subject` is a special `let` for the object under test.

```crystal
Spectator.describe Parser do
  let(input) { "--verbose --output /tmp/out.txt" }
  subject(parser) { Parser.new(input.split) }

  it "enables verbose" do
    expect(parser.verbose?).to be_true
  end

  it "sets the output path" do
    expect(parser.output_path).to eq "/tmp/out.txt"
  end
end
```

Note: `let` variables are **not** available in `before_all`/`after_all` hooks — use instance
variables via `before_each` for shared mutable state that needs setup.

---

## Pending Examples

```crystal
it "handles edge case"   # no block = pending, shown in output but not failed
pending "not implemented yet" do
  # ...
end
```

---

## Mocking and Doubles

Spectator provides doubles and mocks. Prefer abstract class interfaces over `inject_mock`
(which alters production type behavior).

```crystal
# Define an abstract interface
abstract class Store
  abstract def get(key : String) : String?
  abstract def set(key : String, value : String) : Nil
end

# In tests, create a double
double(MockStore, Store) do
  stub def get(key : String) : String?
    nil
  end

  stub def set(key : String, value : String) : Nil
  end
end

Spectator.describe MyService do
  let(store) { MockStore.new }
  subject(service) { MyService.new(store: store) }

  it "reads from the store" do
    allow(store).to receive(:get).with("config").and_return("production")
    expect(service.config_mode).to eq "production"
  end
end
```

---

## File Layout

Mirror your `src/` tree under `spec/`:

```
src/
  my_project/
    parser.cr
    processor.cr
spec/
  spec_helper.cr
  my_project/
    parser_spec.cr
    processor_spec.cr
```

Each `_spec.cr` file requires `./spec_helper` (not `../spec_helper`) — Spectator handles
the relative path correctly via the standard Crystal spec runner.

---

## Comparison with stdlib Spec

| Feature | Spectator | stdlib Spec |
|---|---|---|
| Syntax | `expect(x).to eq y` | `x.should eq y` |
| `let` / `subject` | ✓ | ✗ |
| Mocking/doubles | ✓ | ✗ (need external shard) |
| Hooks | `before_each`, `before_all`, `around_each` | `before_each`, `before_all` |
| Pending examples | ✓ | ✓ |
| Run command | `crystal spec -v --error-trace` | same |

The stdlib `Spec` module is incompatible — do not `require "spec"` alongside Spectator.
