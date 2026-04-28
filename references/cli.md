# CLI Argument Parsing in Crystal

Paul's preferred style: hand-rolled `while opts.shift?` loop in `initialize`, with a
synchronized `HELP` heredoc. This keeps argument handling explicit and avoids framework
magic, at the cost of having to keep help text manually in sync (the pattern below makes
that discipline easy).

---

## Standard Pattern

```crystal
require "option_parser"  # NOT used — this is the hand-rolled style

HELP = <<-HELP
  Usage: my-tool [options] [arguments]

  Options:
    --output PATH       Write output to PATH (default: stdout)
    --format FORMAT     Output format: text, json, yaml (default: text)
    --verbose           Enable verbose logging
    --dry-run           Show what would happen without doing it
    --help              Show this help

  Arguments:
    FILES               One or more input files to process

  Examples:
    my-tool --format json input.txt
    my-tool --output /tmp/out.txt --verbose a.txt b.txt
  HELP

class MyTool
  property output_path : String?
  property? verbose : Bool = false
  property? dry_run : Bool = false
  property format : String = "text"
  property files : Array(String) = [] of String

  def initialize(opts = ARGV.dup)
    while opt = opts.shift?
      case opt
      when "--output"
        @output_path = opts.shift? || raise ArgumentError.new "--output requires a PATH argument"
      when "--format"
        @format = opts.shift? || raise ArgumentError.new "--format requires a FORMAT argument"
      when "--verbose"
        @verbose = true
      when "--dry-run"
        @dry_run = true
      when "--help", "-h"
        puts HELP
        exit 0
      when /\A--/
        raise ArgumentError.new "#{opt}: unknown option"
      else
        @files << opt
      end
    end
  end
end
```

### Key conventions

- `opts = ARGV.dup` as the default keeps `ARGV` intact and makes the constructor testable
  by passing a hand-built `Array(String)`.
- Use `opts.shift?` (returns `String?`) in the `while` condition — when the array is empty
  it returns `nil` and the loop exits cleanly.
- When an option requires a value, use `opts.shift?` and raise `ArgumentError` on `nil`,
  so the error message names the flag.
- Unknown flags that start with `--` (or `-`) should raise rather than silently skip.
  Non-flag arguments (no leading `-`) go into a positional accumulator.
- The `HELP` heredoc is the canonical source of truth for option documentation. Keep it
  adjacent to the parser so it's easy to update both in the same edit.

---

## Keeping HELP in Sync

The heredoc approach doesn't enforce sync automatically. A few habits help:

1. Add each `when` arm and its `HELP` line in the same commit.
2. Order options alphabetically in both the `case` and the `HELP` block.
3. In code review (or ameba custom rules), verify that every `when "--foo"` arm has a
   corresponding line in HELP — this is currently a manual check.

---

## Subcommands

For tools with subcommands, parse the first non-flag argument as the command, then delegate
remaining `opts` to a subcommand-specific parser:

```crystal
def initialize(opts = ARGV.dup)
  while opt = opts.shift?
    case opt
    when "serve"
      @command = ServeCommand.new(opts)
      return
    when "migrate"
      @command = MigrateCommand.new(opts)
      return
    when "--help", "-h"
      puts HELP
      exit 0
    when /\A--/
      raise ArgumentError.new "#{opt}: unknown option"
    else
      raise ArgumentError.new "#{opt}: unknown subcommand"
    end
  end
end
```

Each subcommand class has the same `initialize(opts = ARGV.dup)` pattern and its own `HELP` constant.

---

## Testing the Parser

Because the constructor accepts `Array(String)`, it is trivially testable with Spectator:

```crystal
Spectator.describe MyTool do
  describe "#initialize" do
    it "parses --format" do
      tool = MyTool.new(["--format", "json", "file.txt"])
      expect(tool.format).to eq "json"
      expect(tool.files).to eq ["file.txt"]
    end

    it "raises on unknown option" do
      expect { MyTool.new(["--nope"]) }.to raise_error(ArgumentError, /unknown option/)
    end

    it "raises when --output has no argument" do
      expect { MyTool.new(["--output"]) }.to raise_error(ArgumentError, /requires a PATH/)
    end
  end
end
```

---

## When to Reach for OptionParser Instead

The stdlib `OptionParser` (`require "option_parser"`) is worth considering when:

- You need `--flag[=VALUE]` optional-value syntax (the hand-rolled pattern struggles with this).
- You want auto-generated help text from the flag definitions.

If you use `OptionParser`, still write a `HELP` constant for the banner section (description and
examples), since `OptionParser` only generates the flag table, not usage prose.
