---
layout: article
description:  "Pattern matching in Elixir & it's use cases"
title: "Cleaner flows with Pattern Matching in Elixir"
date:   2022-05-19
tags: [elixir, pattern-matching]
image: images/preview.png
---

Pattern matching is a powerful feature of most Functional Programming languages and allows for a huge improvement in readability.
At it's core, pattern match in Elixir relies on the match operator, the = sign. It tries to match the right hand side with the left hand side. 
> What difference would this have with the == or the === sign that many other languages have?

The match operator allows you to bind variables in addition to checking for a match. Let's understand this through an example

```console
iex(1)> %{ first_name: "smith", last_name: last_name } = %{ first_name: "smith", last_name: "john"}
%{first_name: "smith", last_name: "john"}

iex(2)> last_name
"john"
```

Whenever a variable is present on the LHS of a match expression, the value of the expression will be assigned into the variable. In the above example,  it will give a successful match on any name which has the first_name value as `smith` and it will bind the last_name ("john") in the `last_name` variable. If the `first_name` does not match `smith` , it will raise a MatchError.

## Pattern Match in Function Parameters

Let's start with an example 

```elixir
def print(1), do: IO.puts("one")         # print(1) => "one"
def print(2), do: IO.puts("two")         # print(2) => "two"
def print(_n), do: IO.puts("noop")       # print(3) => "noop"
```

The above example has three implementations defined for print
The overloaded print function will start pattern matching on the arguments with each definition until it finds a match or it would raise a `FunctionClauseError`
Functions are matched for parameters from top to bottom in the order of definition

Looking at a more complicated example, 

```elixir
def transform(html, :italics), do: "<i> #{html} </i>"
def transform(html, :bold), do: "<b> #{html} </b>"
def transform(html, :strikethrough), do: "<s> #{html} </s>"
```

Here the HTML can be transformed to add an italics/bold/strikethrough surrounding it, by 
changing the way we call transform with the second argument.

You could extract variables inside different types such as tuples, maps and lists using pattern matching in function parameters as well

```elixir
def switcheroo({x, y}), do: {y, x}
# switcheroo({1,2}) => {2, 1}

def extract_name(%{name: name}), do: name
# extract_name(%{name: "john", ...}) => "john"

def extract_head([head | tail]), do: head
# extract_head([1,2,3]) => 1

def name_ends_with_john?(%{first_name: first_name, last_name: "john"}), do: true
def name_ends_with_john?(_), do: false
# name_ends_with_john(%{first_name: "smith", last_name: "john"}) => true
```

## Cleaner flows with Pattern Matching

Here is an example for the merge function in Merge Sort, which merges two already sorted arrays. Note how the pattern match in the function arguments combined with the `|`  operator improves readability.

```elixir
def merge([], list_b), do: list_b

def merge(list_a, []), do: list_a

def merge(list_a = [head_a | rest_a], list_b = [head_b | rest_b]) do
	if head_a < head_b do
		[head_a | merge(rest_a, list_b)]
	else
		[head_b | merge(list_a, rest_b)]
	end
end
```

## Doing more with Guards

Guard clauses restrict the the parameters when pattern matching in functions. Consider a function to check whether the elements in a given list doubles with every element.

```elixir
  def is_doubling([x | tail = [y | _]]) when x == 2 * y, do: is_doubling(tail)
  def is_doubling([_]), do: true
  def is_doubling(_), do: false
```
In addition to matching on the function parameters, `when x==2*y` should also be true for the function clause to match. 

Consider the following example for checking of valid parantheses. 
The `<<"(", rest::binary>>` format allows for us to pattern match on strings, where the end of the string is allowed to be of variable length and is stored in `rest`.

```elixir
def is_balanced(s), do: is_balanced(s, 0, 0)

def is_balanced(<<"(", rest::binary>>, open_count, close_count) do
  is_balanced(rest, open_count+1, close_count)
end

def is_balanced(<<")", rest::binary>>,
		open_count,
		close_count)
	when close_count < open_count do
  is_balanced(rest, open_count, close_count+1)
end

def is_balanced("", count, count), do: true
def is_balanced(_, _, _), do: false
```

All in all, pattern matching is a very useful feature enhancing the readability.