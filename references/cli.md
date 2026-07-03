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

HELP_BANNER = <<-HELP_BANNER
  Usage: my-tool [options] [arguments]
  HELP_BANNER

HELP_FOOTER = <<-HELP_FOOTER
  Arguments:
    FILES               One or more input files to process

  Examples:
    my-tool --format json input.txt
    my-tool --output /tmp/out.txt --verbose a.txt b.txt
  HELP_FOOTER

class MyTool
  property output_path : String?
  property? verbose : Bool = false
  property? dry_run : Bool = false
  property format : String = "text"
  property files : Array(String) = [] of String

  def initialize(opts = ARGV.dup)
    parser = OptionParser.new do |parser|
      parser.banner = HELP_BANNER
      parser.on("--output PATH", "Write output to PATH (default: stdout)") { |path| @output_path = path }
      parser.on("--format FORMAT", "Output format: text, json, yaml (default: text)") { |fmt| @format = fmt }
      parser.on("--verbose", "Enable verbose logging") { @verbose = true }
      parser.on("--dry-run", "Show what would happen without doing it") { @dry_run = true }
      parser.on("--help", "Show this help") do
        puts parser
        puts HELP_FOOTER
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
* The `HELP_BANNER` sets the banner written by the `OptionParser#to_s` method. That method shows the
  banner followed by the options it knows about. What OptionParser can't generate — the usage line,
  the positional `Arguments`, and `Examples` are in the HELP_FOOTER. The `Options:` section is
  generated from the `on` descriptions, so `puts parser` prints the banner followed by the option
  list.

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

## Output to a pipe

When a CLI is run with its output sent via a pipe to a command like `less` or `more`, and the pager
or other destination exits before the CLI has written all the output, writes to STDOUT/STDERR will
raise an IO::Error about a "Broken pipe".

It's important to rescue this near each place that output is written to STDOUT/STDERR (or where it
could be writing to STDOUT/STDERR) and handle it cleanly. The CLI should not produce a stack trace
and exit with an error code when this happens.

If there are no side effects that happen after writing the output, then exiting cleanly when
catching this is acceptable:

```crystal
def run
  # do work...
  write_output STDOUT
end

def write_output(io = STDOUT)
  1_000.times do
    io.puts "w00t!"
  end
rescue e : IO::Error
  raise e unless e.message =~ /Broken pipe/
  exit 0
end
```

However, if there are side effects that may happen after the output is written, then it's important
to still allow them to happen:

```crystal
def run
  # do work...
  write_output STDOUT
  File.delete @tmpfile
end

def write_output(io = STDOUT)
  1_000.times do
    io.puts "w00t!"
  end
rescue e : IO::Error
  raise e unless e.message =~ /Broken pipe/
end
```

For this reason it can be advantageous to organize the tool so that writing output happens after any
side effects. This is not always the right thing to do, though, so consider the consequences. For
example, if output is small (a few hundred kilobytes) then generating the dataset in memory and then
writing it at the end of execution can be good, but if it's possible the dataset it too large to
make sense keeping in memory, or writing a progress update of a long running task, then the failed
writes should just be silently ignored.
