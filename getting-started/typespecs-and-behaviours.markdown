---
section: getting-started
layout: getting-started
title: Typespecs and behaviours
---

## Types and specs

Elixir is a dynamically typed language, so all types in Elixir are checked at runtime. Nonetheless, Elixir comes with **typespecs**, which are a notation used for:

1. declaring typed function signatures (also called specifications);
2. declaring custom types.

Typespecs are useful for code clarity and static code analysis (for example, Erlang's [Dialyzer](http://www.erlang.org/doc/man/dialyzer.html) tool).

### Function specifications

Elixir provides many [built-in types](https://hexdocs.pm/elixir/typespecs.html#built-in-types), such as `integer` or `pid`, that can be used to document function signatures.  For example, the `round/1` function, which rounds a number to its nearest integer. As you can see [in its documentation](https://hexdocs.pm/elixir/Kernel.html#round/1), `round/1`'s typed signature is written as:

```elixir
round(number()) :: integer()
```

The syntax is to put the function and its input on the left side of the `::` and the return value's type on the right side. Be aware that types *may* omit parentheses.

In code, function specs are written with the `@spec` attribute, typically placed immediately before the function definition. Specs can describe both public and private functions. The function name and the number of arguments used in the `@spec` attribute must match the function it describes.

Elixir supports compound types as well. For example, a list of integers has type `[integer]`, or maps that define keys and types (see the example below).

You can see all the built-in types provided by Elixir [in the typespecs docs](https://hexdocs.pm/elixir/typespecs.html).

### Defining custom types

Defining custom types can help communicate the intention of your code and increase its readability. Custom types can be defined within modules via the `@type` attribute.

A simple example of a custom type implementation is to provide a more descriptive alias of an existing type. For example, defining `year` as a type makes your function specs more descriptive than if they had simply used `integer`:

```elixir
defmodule Person do
   @typedoc """
   A 4 digit year, e.g. 1984
   """
   @type year :: integer

   @spec current_age(year) :: integer
   def current_age(year_of_birth), do: # implementation
end
```

The `@typedoc` attribute, similar to the `@doc` and `@moduledoc` attributes, is used to document custom types.

You may define compound custom types, e.g. maps:

```elixir
@type error_map :: %{
   message: String.t,
   line_number: integer
}
```

[Structs](https://elixir-lang.org/getting-started/structs.html) offer similar functionality.

Let's look at another example to understand how to define more complex types. Say we have a `LousyCalculator` module, which performs the usual arithmetic operations (sum, product, and so on) but, instead of returning numbers, it returns tuples with the result of an operation as the first element and a random remark as the second element.

```elixir
defmodule LousyCalculator do
  @spec add(number, number) :: {number, String.t}
  def add(x, y), do: {x + y, "You need a calculator to do that?!"}

  @spec multiply(number, number) :: {number, String.t}
  def multiply(x, y), do: {x * y, "Jeez, come on!"}
end
```

Tuples are a compound type and each tuple is identified by the types inside it (in this case, a number and a string). To understand why `String.t` is not written as `string`, have another look at the [typespecs docs](https://hexdocs.pm/elixir/typespecs.html#the-string-type).

Defining function specs this way works, but we end up repeating the type `{number, String.t}` over and over. We can use the `@type` attribute to declare our own custom type and cut down on the repetition.

```elixir
defmodule LousyCalculator do
  @typedoc """
  Just a number followed by a string.
  """
  @type number_with_remark :: {number, String.t}

  @spec add(number, number) :: number_with_remark
  def add(x, y), do: {x + y, "You need a calculator to do that?"}

  @spec multiply(number, number) :: number_with_remark
  def multiply(x, y), do: {x * y, "It is like addition on steroids."}
end
```

Custom types defined through `@type` are exported and are available outside the module they're defined in:

```elixir
defmodule QuietCalculator do
  @spec add(number, number) :: number
  def add(x, y), do: make_quiet(LousyCalculator.add(x, y))

  @spec make_quiet(LousyCalculator.number_with_remark) :: number
  defp make_quiet({num, _remark}), do: num
end
```

If you want to keep a custom type private, you can use the `@typep` attribute instead of `@type`. The visibility also affects whether or not documentation will be generated by tools like [ExDoc](https://hexdocs.pm/ex_doc/readme.html), Elixir's documentation generator.

### Static code analysis

Typespecs are not only useful to developers as additional documentation. The Erlang tool [Dialyzer](http://www.erlang.org/doc/man/dialyzer.html), for example, uses typespecs in order to perform static analysis of code. That's why, in the `QuietCalculator` example, we wrote a spec for the `make_quiet/1` function even though it was defined as a private function.

## Behaviours

Many modules share the same public API. Take a look at [Plug](https://github.com/elixir-lang/plug), which, as its description states, is a **specification** for composable modules in web applications. Each *plug* is a module which **has to** implement at least two public functions: `init/1` and `call/2`.

Behaviours provide a way to:

* define a set of functions that have to be implemented by a module;
* ensure that a module implements all the functions in that set.

If you have to, you can think of behaviours like interfaces in object oriented languages like Java: a set of function signatures that a module has to implement. Unlike Protocols, behaviours are independent of the type/data.

### Defining behaviours

Say we want to implement a bunch of parsers, each parsing structured data: for example, a JSON parser and a MessagePack parser. Each of these two parsers will *behave* the same way: both will provide a `parse/1` function and an `extensions/0` function. The `parse/1` function will return an Elixir representation of the structured data, while the `extensions/0` function will return a list of file extensions that can be used for each type of data (e.g., `.json` for JSON files).

We can create a `Parser` behaviour:

```elixir
defmodule Parser do
  @doc """
  Parses a string.
  """
  @callback parse(String.t) :: {:ok, term} | {:error, atom}

  @doc """
  Lists all supported file extensions.
  """
  @callback extensions() :: [String.t]
end
```

Modules adopting the `Parser` behaviour will have to implement all the functions defined with the `@callback` attribute. As you can see, `@callback` expects a function name but also a function specification like the ones used with the `@spec` attribute we saw above. Also note that the `term` type is used to represent the parsed value. In Elixir, the `term` type is a shortcut to represent any type.

### Implementing behaviours

Implementing a behaviour is straightforward:

```elixir
defmodule JSONParser do
  @behaviour Parser

  @impl Parser
  def parse(str), do: {:ok, "some json " <> str} # ... parse JSON

  @impl Parser
  def extensions, do: [".json"]
end
```

```elixir
defmodule CSVParser do
  @behaviour Parser

  @impl Parser
  def parse(str), do: {:ok, "some csv " <> str} # ... parse CSV

  @impl Parser
  def extensions, do: [".csv"]
end
```

If a module adopting a given behaviour doesn't implement one of the callbacks required by that behaviour, a compile-time warning will be generated.

Furthermore, with `@impl` you can also make sure that you are implementing the **correct** callbacks from the given behaviour in an explicit manner. For example, the following parser implements both `parse` and `extensions`. However, thanks to a typo, `BADParser` is implementing `parse/0` instead of `parse/1`.

```elixir
defmodule BADParser do
  @behaviour Parser

  @impl Parser
  def parse, do: {:ok, "something bad"}

  @impl Parser
  def extensions, do: ["bad"]
end
```

This code generates a warning letting you know that you are mistakenly implementing `parse/0` instead of `parse/1`.
You can read more about `@impl` in the [module documentation](https://hexdocs.pm/elixir/main/Module.html#module-impl).

### Using behaviours

Behaviours are useful because you can pass modules around as arguments and you can then _call back_ to any of the functions specified in the behaviour. For example, we can have a function that receives a filename, several parsers, and parses the file based on its extension:

```elixir
@spec parse_path(Path.t(), [module()]) :: {:ok, term} | {:error, atom}
def parse_path(filename, parsers) do
  with {:ok, ext} <- parse_extension(filename),
       {:ok, parser} <- find_parser(extension, parsers),
       {:ok, contents} <- File.read(filename) do
    parser.parse(contents)
  end
end

defp parse_extension(filename) do
  if ext = Path.extname(filename) do
    {:ok, ext}
  else
    {:error, :no_extension}
  end
end

defp find_parser(extension, parsers) do
  if parser = Enum.find(parsers, fn parser -> ext in parser.extensions() end) do
    {:ok, parser}
  else
    {:error, :no_matching_parser}
  end
end
```

Of course, you could also invoke any parser directly: `CSVParser.parse(...)`.

Note you don't need to define a behaviour in order to dynamically dispatch on a module, but those features often go hand in hand.
