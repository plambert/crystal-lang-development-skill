# CLI Argument Parsing in Crystal

Paul's preferred style changes based on the size of the codebase.

For a very small utility with no subcommands and very few options, and which is not going to be more
than 1 source files, use the OptionParser class from the standard library.

For anything larger, use Shell::AutoComplete, also below.

## Small use case, no shard needed

Use OptionParser (`require option_parser`), with a synchronized `HELP` heredoc. This
makes argument handling explicit and avoids adding a shard, at the cost of having to keep the HELP
text manually in sync (the pattern below makes that discipline easy).

Be sure to have a `--version` flag that just outputs the `"#{PROGRAM_NAME} #{VERSION}"` and exits.
Be sure to update the `VERSION = "0.1.0"` line as described in the base file of this skill.

---

### OptionParser pattern

```crystal
require "option_parser"

HELP = <<-HELP
  Usage: my-tool [options] [arguments]

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
    parser = OptionParser.new do |parser|
      parser.banner = HELP
      parser.on("--output PATH", "Write output to PATH (default: stdout)") { |path| @output_path = path }
      parser.on("--format FORMAT", "Output format: text, json, yaml (default: text)") { |fmt| @format = fmt }
      parser.on("--verbose", "Enable verbose logging") { @verbose = true }
      parser.on("--dry-run", "Show what would happen without doing it") { @dry_run = true }
      parser.on("--help", "Show this help") do
        puts parser
        exit 0
      end
      parser.invalid_option { |opt| raise ArgumentError.new "#{opt}: unknown option" }
      parser.unknown_args { |args| @files.concat args }
    end
    parser.parse(opts)
  end
end
```

#### Key conventions

* `opts = ARGV.dup` as the default keeps `ARGV` intact and makes the constructor testable
  by passing a hand-built `Array(String)`. `parser.parse(opts)` consumes the copy, not `ARGV`.
* Each `parser.on` block sets a property directly; flags that take a value (`--output PATH`)
  receive it as the block argument, so OptionParser handles the "requires an argument" error.
* `parser.invalid_option` raises `ArgumentError` so unknown flags fail loudly rather than
  being silently skipped.
* `parser.unknown_args` collects the non-flag positional arguments into the accumulator.
* The `HELP` banner covers only what OptionParser can't generate — the usage line, the
  positional `Arguments`, and `Examples`. The `Options:` section is generated from the
  `on` descriptions, so `puts parser` prints the banner followed by the option list.

---

#### Testing the Parser

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

## Shell::AutoComplete

Add to `shard.yml`:

```yaml
dependencies:
  shell-auto_complete:
    github: plambert/shell-auto_complete.cr
```

Run `shards install` and then the `./lib/shell-auto_complete/SKILL.md` tells you how to use it.

### When to use

Use the shard when:

* You need subcommands
* You have more than 4 or 5 flags (not counting boolean negations)
* You want auto-generated help text from the flag definitions.
* The flags are complicated enough that shell completion is going to be useful
