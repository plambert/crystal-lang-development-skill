---
name: crystal-development
description: Follow best practices when working with the crystal programming language.
---

## Crystal language code requirements

### Formatting Crystal language files with `crystal tool format`

Code written in Crystal must always be formatted with `crystal tool format src/to/file.cr` before being committed to git.

### Checking Crystal language files for style and good practices with `ameba`

Code written in Crystal should pass ameba tests; these can be run with `ameba` in the root directory of the repository. Ameba is configured by a `.ameba.yml` file in the root of the repository. It should be created with the below setting if it does not exist.

#### Ameba's `.ameba.yml` file

Any repository with Crystal code that does not have a file `.ameba.yml` in the root of the repository should have one created with the following content:

```YAML
Metrics/CyclomaticComplexity:
  MaxComplexity: 20
```

#### Disabling ameba warnings

Methods that have a long case statement can be prepended with the comment below to disable the ameba warning:

```
# ameba:disable Metrics/CyclomaticComplexity
```

### Building and testing code

Run `shards install` to install any dependencies in the `shard.yml` file; run `shards update` when the `shard.yml` file is updated.

Always run `Spec` tests with `crystal spec -v --error-trace`.

Always build crystal programs with `shards build --error-trace`, and test programs with `crystal build --error-trace -o bin/test_program test_program.cr`.

Create test programs in the repository under `examples/`, and whenever possible, turn them into `Spec` tests under `spec/`.

### Nillable values

In Crystal it's very important to handle nillable attribute values well. There are two approaches to this, and the approach to use depends on the expected significance of a `nil` value in the attribute.

#### Avoid `.not_nil!`

Avoid using the `.not_nil!` method; there are very few cases where it is the correct approach. Instead use one of the approaches below to create a local value that the compiler can be certain is not `nil`.

#### Using an `if` block for nillable values

When a value might be `nil` and is only used in one or two statements, it is often best to assign it to a local variable and test for `nil`. When the value cannot be `Bool`, and more specifically `false`, it is idiomatic to write this, for example, to use the `@thing` attribute when it might be `nil`:

```Crystal
if thing = self.thing
  # use `thing` without needing to worry about it being `nil`
end

```

#### Gating an entire method on a nillable value using `Guard`

Often, we want to create a method that will return `nil` when an attribute is `nil`, but otherwise perform many tasks with it. Or else, we want the method to raise an exception if the attribute is `nil`.

For this, we use the `Guard` module. Add this to the `shard.yml` file and run `shards` to install the module; this only needs to be done once.

```YAML
dependencies:
  guard:
    github: plambert/guard.cr
```

Then in any class, module, enum, or struct, put `include Guard` at the start.  Then use one of these two forms:

##### Using `Guard` to return `nil` from a method when an attribute is `nil`

```Crystal
class Foo
  include Guard

  def a_method?(params)
    # return `nil` if @thing is `nil` or `false`, otherwise proceed
    thing = guard self.thing
    # proceed here knowing that `thing` cannot be `nil`.
  end
end
```

##### Using `Guard` to raise an `Exception` from a method when an attribute is `nil`

```Crystal
class Foo
  include Guard

  def a_method(params)
    # raise an exception if @thing is `nil` or `false`, otherwise proceed
    thing = guard self.thing { RuntimeError.new "expected @thing to not be nil or false" }
    # proceed here knowing that `thing` cannot be `nil`.
  end
end
```

##### Allowing `false` values with `Guard`

Use `guard!` (with the exclamation point) if you want `false` values to be allowed; otherwise it is identical to `guard`.


#### Using `property!` for values that are definitely not `nil` at the time a method is run

In the rare case when a value starts out `nil` during initialization, but we know that a method won't be called until after it is definitely given a not-nil value, we can change `property` to `property!`.

For example, `property! thing : String | Int32` will create `@thing` as nillable, but `self.thing` will raise an Exception if `@thing` is nil, so we can use it anywhere we are certain it is not `nil`.

In addition it creates a `self.thing?` method that will return `@thing` directly, as a nillable value, which is convenient for tests:

```Crystal
if self.thing?
  self.thing.method # we know it isn't `nil`
end
```

There is a **MAJOR CAVEAT** with this approach, which is that an attribute value can be changed at any time by another `Fiber` of execution, so it is only safe to use this if we know the value will never be changed in another `Fiber` of execution.

### Using `property?` for `Bool` attributes

Instead of using `property closed : Bool`, we should use `property? closed : Bool`. This will create the getter as `#closed?` with the question mark. That makes it much clearer that it is a boolean value. Do not use this for `Bool?` or other nillable values.

### Always use a macro to specify getters and setters for attributes of a class or struct.

While it's possible to do this:

```Crystal
class Foo
  @thing : String | Int32 = "foo"
  # ...
end
```

It is always better to use `property`, `property!`, or `property?`.

If only a getter is needed, use `getter`, `getter!`, or `getter?`.  And if only a setter is needed, use `setter`, `setter!`, or `setter?`.

### String interpolation implies `.to_s`.

When interpolating a value into a `String` literal, do not use `#to_s` explicitly, as it is called during interpolation.  In other words do not write `"we have a #{thing.to_s}"` but instead merely `"we have a #{thing}"`. It _is_ OK to use `#inspect` if it's appropriate.

### Use descriptive variable names

Do not use `i` or `j` as a variable name, even in a small block. Always use a more descriptive name, so that future additions don't leave it indecipherable.

For example, instead of `things.each_with_index { |thing, i| STDOUT.printf "%3d %s\n", i, thing }` we should write `things.each_with_index { |thing, thing_index| STDOUT.printf "%3d %s\n", thing_index, thing }`.

One exception is for the parameters to a block passed to `sort`; these can be `a` and `b`.

### Include block arguments in method definitions

If a method will `yield`, it should be defined with `&` as the final argument. Where it helps clarity and does not limit flexibility, specify a `Type` for the block, like `def method(something, something_else, & : Int32 -> Bool` if an `Int32` will be yielded, and a `Bool` is expected as the return value.

### Exhaustive `case` statements

#### Value types

When using a `case` statement to check the type of a value, use `case ... in ... end` instead of `case ... when ... end`.  For example:

```Crystal
def indent(amount : String | Int32) : String
  case amount
  in String
    amount
  in Int32
    " " * amount
  end
end
```

This form does _not_ allow an `else` statement, and requires that all possible types be included.

#### Enum types

When using a `case` statement to check an `Enum` value, use `case ... in ... end` to allow the compiler to enforce an exhaustive check.  For example:

```Crystal
enum OutputFormat
  Text
  AnsiColor
  Markdown
  HTML
  JSON
  YAML
end

def format(io, output_format : OutputFormat) : Nil
  case output_format
  in .text?
    format_text(io)
  in .ansi_color?
    format_ansi_color(io)
  in .markdown?
    format_markdown(io)
  in .html?
    format_html(io)
  in .json?
    format_json(io)
  in .yaml?
    format_yaml(io)
  end
end
```

This ensures that if a new value is added to the `OutputFormat` enum, it _must_ be added to the case statement in order to successfully compile.

Note that an enum defines test methods like those above (`#lower_case_value?`) to make case statements like this and tests for equality to be easier to write. `color.blue?` is much simpler and clearer than `color == ::Some::Namespace::Color::BLUE`.

Also note that the compiler allows a `Symbol` to be passed as a parameter in place of an `enum` when the expected value type is not a union.

For example, using the `OutputFormat` enum and `#format` method from above:

```Crystal
format io, :text
```
