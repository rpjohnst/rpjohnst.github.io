---
title: Writing a recursive ascent parser by hand
---

I've been exploring various ways to write parsers. For a long time, I've used hand-written [recursive descent](https://en.wikipedia.org/wiki/Recursive_descent_parser) for its straightforwardness, flexibility, and performance. There is another way---parser generators like [Menhir](http://gallium.inria.fr/~fpottier/menhir/), [LALRPOP](https://github.com/lalrpop/lalrpop), or the venerable [Bison](https://www.gnu.org/software/bison/) use the bottom-up [LR algorithm](https://en.wikipedia.org/wiki/LR_parser).

While LR parser generators have several nice properties---using the grammar directly as the parser's source code, automatic ambiguity checking and other algorithmic guarantees---they can be a pain. They involve an extra build step, their output is often an opaque state table, and it's hard to tweak their behavior.

So I decided I would try an experiment: write an LR parser by hand, and see how readable I could make it. This means writing functions instead of filling out a state table, giving things meaningful names, and avoiding too much jargon. This should build some intution for how LR parsers work and why you might use one.

The code in this article is written in [Rust](https://www.rust-lang.org). It should be easy to follow if you're familiar with a language with sum types and pattern matching.

## The LR algorithm

An LR parser is a kind of *shift-reduce* parser. It works by *shifting* tokens from the input onto a stack until it recognizes a rule from the grammar, when it *reduces* those tokens into the thing they represent. Unlike recursive descent, which processes the grammar top-down, LR is bottom-up: it reads the complete body of a rule before it decides which rule to use.

How does the parser know when it's seen the body of a rule? Intuitively, it defines a state for each position in a grammar rule, tracks a current state based on the tokens it shifts, and triggers a reduce action when it reaches a state for the end of a rule. It also pushes states onto the stack alongside their corresponding tokens, so it can return to where it was before starting into that rule.

A table-driven LR implementation is fairly direct---an actual stack, a lookup table from the current state and token to an action. A recursive implementation calls a function to shift a token and change states, and returns when it's time to reduce. This is [recursive ascent](https://en.wikipedia.org/wiki/Recursive_ascent_parser)!

That was one thing that confused me early on. The recursion in "recursive descent" happens when the parser "descends" into a nested rule. I kept expecting the function calls of recursive ascent to "ascend" somehow. This is not how it works! The stack grows with function calls on shift actions and shrinks with returns on reduce actions.

## A small grammar

Here is a simple example grammar to try this out. It matches a subset of JSON---possibly-nested arrays of numbers.

```
value = NUMBER | array
array = "[" "]" | "[" elements "]"
elements = value | elements "," value
```

Parsing the input `[1, [], 3]`, using the shift-reduce approach described above, should look like this:

| Input        | Stack                | Action                                 |
| ------------ | -------------------- | -------------------------------------- |
| `[1, [], 3]` |                      | Shift `[`                              |
| `1, [], 3]`  | `[`                  | Shift `1`                              |
| `, [], 3]`   | `[ 1`                | Reduce `value = NUMBER`                |
| `, [], 3]`   | `[ value`            | Reduce `elements = value`              |
| `, [], 3]`   | `[ elements`         | Shift `,`                              |
| `[], 3]`     | `[ elements ,`       | Shift `[`                              |
| `], 3]`      | `[ elements , [`     | Shift `]`                              |
| `, 3]`       | `[ elements , [ ]`   | Reduce `array = "[" "]"`               |
| `, 3]`       | `[ elements , array` | Reduce `value = array`                 |
| `, 3]`       | `[ elements , value` | Reduce `elements = elements "," value` |
| `, 3]`       | `[ elements`         | Shift `,`                              |
| `3]`         | `[ elements ,`       | Shift `3`                              |
| `]`          | `[ elements , 3`     | Reduce `value = NUMBER`                |
| `]`          | `[ elements , value` | Reduce `elements = elements "," value` |
| `]`          | `[ elements`         | Shift `]`                              |
|              | `[ elements ]`       | Reduce `array = "[" elements "]"`      |
|              | `array`              | Reduce `value = array`                 |
|              | `value`              | Done                                   |

Notice the runs of reduce actions: the first time, it works its way up from `1` to `value` to `elements`; later, from `[ ]` to `array` to `value` and from `elements , value` to `elements`; finally, from `[ elements ]` to `array` to `value`.

## Tokens and nonterminals

The stack holds two types of objects: tokens and nonterminals. A token is the lowest-level piece of input produced by the lexer---in this case, numbers and punctuation. A nonterminal is the name on the left-hand side of a rule, and the output of a reduce action---`value`, `array`, or `elements`.

We need a way to represent both of them. They will just be local variables, instead of going on an explicit stack, so separate `Token` and `Nonterminal` types will be fine. `Token` can look like this:

```rust
struct Lex { ... }

impl Lex {
    fn token(&mut self) -> Token { ... }
}

enum Token {
    Number(f64),
    LeftBracket,
    RightBracket,
    Comma,
    EndOfFile,
}
```

One important point is that while nonterminals can form a *parse tree*, they do not necessarily form an *abstract syntax tree*, which is usually what we actually want. For example, an `array` AST node should contain a flat list of values, but the process of building an `array` nonterminal involves several nested partial results, in the form of `elements` nonterminals.

The way parser generators handle this is by associating a user-defined *action* with each rule, which it runs at each reduction. This way it never actually has to build the parse tree explicitly, and it leaves the question of how (or whether) to build the AST up to the user. Nonterminal types are thus wrappers around the user-defined actions' types:

```rust
struct Value(JsonValue);
struct Array(Vec<JsonValue>);
struct Elements(Vec<JsonValue>);
```

```rust
enum JsonValue {
    Number(f64),
    Array(Vec<JsonValue>)
}
```

Finally, we need a way to report parse errors. We can use an empty struct for now, but eventually it might contain location information from the lexer:

```rust
struct ParseError;
```

## Shifting and reducing

The parser's initial state is just before a `value`. We can express this with a made-up rule called `goal` built from a `value` and the end of the input, written as `$`. We can write out this state using a `*` to mark the current position:

```
goal = * value $
```

Because recursive ascent is bottom-up, it cares primarily about the lowest-level units it might see next---tokens. Thus, being at the beginning of a `goal` means the parser is also at the beginning of all its sub-rules. This is the opposite of top-down recursive descent, where the parser makes a function call each time it steps into a sub-rule. The full state looks like this:

```
goal = * value $
value = * NUMBER
value = * array
array = * "[" "]"
array = * "[" elements "]"
```

We'll name each state's function using the nonterminal it's in the middle of, followed by how far it's gotten---in this case `goal_start`. There are two possible tokens to shift: `NUMBER` or `"["`. Anything else is a syntax error. The function `goal_start` looks like this:

```rust
// goal = * value $
fn goal_start(lex: &mut Lex) -> Result<Value, ParseError> {
    let token = lex.token();
    match token {
        // The next token is a number; go to the state for `value = NUMBER *`.
        Token::Number(number) => value_number(lex, number)?,

        // The next token is a "["; go to the state for `array = "[" * ...`.
        Token::LeftBracket => {
            let array = array_open(lex)?;

            // Now we have an `array`; go to the state for `value = array *`.
            value_array(lex, array)?
        }

        // The next token is something else; abort with an error.
        _ => return Err(ParseError),
    }

    ...
}
```

The next state knows something about the top of the stack, and this shows up in its function signature---it takes the number as an argument. It also knows something about the grammar---there's nowhere else in the grammar to go with a plain `NUMBER` token, so it's time to reduce. The function `value_number` looks like this:

```rust
// value = NUMBER *
fn value_number(lex: &mut Lex, number: f64) -> Result<Value, ParseError> {
    // Run the "user-defined" action on the token being reduced:
    let value = JsonValue::Number(number);

    // Reduce those tokens to a `value` nonterminal:
    Ok(Value(value))
}
```

The next state after a `"["` is more complicated, but we do know it reduces to an `array`, if it succeeds. This takes the parser to the end of the other `value` rule:

```rust
// value = array *
fn value_array(lex: &mut Lex, array: Array) -> Result<Value, ParseError> {
    // Run the "user-defined" action on the nonterminal being reduced:
    let Array(array) = array;
    let value = JsonValue::Array(array);

    // Reduce that nonterminal to a `value` nonterminal:
    Ok(Value(value))
}
```

Whichever token it shifted first, the parser winds up back in `goal_start` with a `value` nonterminal. Now the parser can advance past the `value` in `goal` to the final state:

```rust
// goal = * value $
fn goal_start(lex: &mut Lex) -> Result<Value, ParseError> {
    let token = lex.token();
    let value = match token {
        ...
    };

    // Now we have a `value`; go to the state for `goal = value * $`.
    Ok(goal_value(lex, value)?)
}

...

// goal = value * $
fn goal_value(lex: &mut Lex, value: Value) -> Result<Value, ParseError> {
    let token = lex.token();
    match token {
        // The next token is the end of the input; reduce to the goal.
        Token::EndOfFile => Ok(value),

        // There's more input, but we've parsed a complete value; report an error.
        _ => return Err(ParseError),
    }
}
```

## Arrays

When the parser sees a `"["` instead of a `NUMBER`, things get more complicated in several ways. Expanding the next state to include any sub-rules is a bit more involved:

```
array = "[" * "]"
array = "[" * elements "]"
elements = * value
elements = * elements "," value
value = * NUMBER
value = * array
array = * "[" "]"
array = * "[" elements "]"
```

The first token `array_open` might shift is a `"]"`. Unlike `NUMBER` or `"["`, this doesn't reduce to a nonterminal that this state can handle. Instead, it reduces to an `array` for `array_open`'s caller---the state that shifted the corresponding `"["`. This means `array_open` has to return immediately when it shifts a `"]"`:

```rust
// array = "[" * "]"
// array = "[" * elements "]"
fn array_open(lex: &mut Lex) -> Result<Array, ParseError> {
    let token = lex.token();
    let value = match token {
        ...

        // The next token is a "]"; go to the state for `array = "[" "]" *` and return.
        Token::RightBracket => return Ok(array_open_close(lex)?),

        ...
    }
    ...
}

// array = "[" "]" *
fn array_open_close(lex: &mut Lex) -> Result<Array, ParseError> {
    // Run the "user-defined" action on the tokens being reduced:
    let array = vec![];

    // Reduce the tokens to an `array` nonterminal:
    Ok(Array(array))
}
```

If the array is non-empty, the parser needs to parse an `elements`. In keeping with the bottom-up style, it starts with a `value` and reduces it using the first `elements` rule:

```rust
// array = "[" * "]"
// array = "[" * elements "]"
fn array_open(lex: &mut Lex) -> Result<Array, ParseError> {
    let token = lex.token();
    let value = match token {
        // The states for these tokens eventually reduce to a `value`.
        Token::Number(number) => value_number(lex, number)?,
        Token::LeftBracket => {
            let array = array_open(lex)?;
            value_array(lex, array)?
        }

        ...
    };

    // Now we have a `value`; go to the state for `elements = value *`.
    let elements = elements_value(lex, value)?;

    ...
}

// elements = value *
fn elements_value(lex: &mut Lex, value: Value) -> Result<Elements, ParseError> {
    // Run the "user-defined" action on the nonterminal being reduced:
    let Value(value) = value;
    let array = vec![value];

    // Reduce that nonterminal to an `elements` nonterminal:
    Ok(Elements(array))
}
```

## Left recursion

Another complication is that the second `elements` rule is left recursive. If you're used to writing grammars for top-down parsers, you've probably avoided left recursion because it doesn't let the parser make any progress. In bottom-up parsers, left recursion lets the parser reduce as it goes rather than all at once at the end. (Take another look at the [example parse](#a-small-grammar) above!)

When the parser moves past the first `elements`, it runs into another complication. If the next token is a `","`, it will parse a `value` and reduce using the rule for `elements "," value`. If the next token is a `"]"`, it will shift it and reduce using the rule for `"[" elements "]"`. But just like the last it shifted a `"]"`, it can't return directly to `array_open`.

This means the next state can't always return the same nonterminal. Instead, it returns an `Either<Elements, Array>` so that `array_open` can decide whether to keep going. This shows up in its state specification, which has rules for two different nonterminals but no way to advance over one:

```
array = "[" elements * "]"
elements = elements * "," value
```

(Unfortunately, this breaks our naming scheme. I've picked `array_open_elements` based on the outer-most rule it might use to reduce, but I'd love to find something better.)

```rust
// array = "[" * "]"
// array = "[" elements "]"
fn array_open(lex: &mut Lex) -> Result<Array, ParseError> {
    ...

    let mut elements = elements_value(lex, value)?;
    loop {
        match array_open_elements(elements) {
            Either::Left(e) => elements = e,
            Either::Right(array) => return Ok(array),
        }
    }
}

// array = "[" elements * "]"
// elements = elements * "," value
fn array_open_elements(lex: &mut Lex, elements: Elements) ->
    Result<Either<Elements, Array>, ParseError>
{
    let token = lex.token();
    match token {
        Token::Comma => {
            let elements = elements_elements_comma(lex, elements)?;
            Ok(Either::Left(elements))
        }

        Token::RightBracket => {
            let array = array_open_elements_close(lex, elements)?;
            Ok(Either::Right(array))
        }

        _ => return Err(ParseError),
    }
}
```

The rest of the parser follows naturally based on the same techniques as what we've seen so far. If the parser sees a `","`, it parses another value and reduces it into a new `elements`:

```rust
// elements = elements "," * value
fn elements_elements_comma(lex: &mut Lex, elements: Elements) ->
    Result<Elements, ParseError>
{
    let token = lex.token();
    let value = match token {
        Token::Number(number) => value_number(lex, number)?,
        Token::LeftBracket => {
            let array = array_open(lex)?;
            value_array(lex, array)?
        }
        _ => return Err(ParseError),
    };

    Ok(elements_elements_comma_value(lex, elements, value)?)
}

// elements = elements "," value *
fn elements_elements_comma_value(lex: &mut Lex, elements: Elements, value: Value) ->
    Result<Array, ParseError>
{
    // Run the "user-defined" action on the nonterminals and token being reduced:
    let Elements(array) = elements;
    let Value(value) = value;
    array.push(value);

    // Reduce them to an `elements` nonterminal:
    Ok(Elements(array))
}
```

On the other hand, if the parser sees a `"]"`, it reduces it into a new `array`:

```rust
// array = "[" elements "]" *
fn array_open_elements_close(lex: &mut Lex, elements: Elements) ->
    Result<Array, ParseError>
{
    // Run the "user-defined" action on the tokens and nonterminal being reduced:
    let Elements(array) = elements;
    let array = JsonValue::Array(array);

    // Reduce them to an `array` nonterminal:
    Ok(Array(array))
}
```

## Recursive ascent

That's a recursive ascent parser! It uses mutually-recursive function calls to build up sequences of tokens, eagerly combining them into nonterminals as it recognizes them. This is fairly different from recursive descent, but still possible to hand-write.

One thing to point out is that this grammar is LR(0)---the states only care about what's already on the stack. More complicated grammars require lookahead, which makes them LR(1). A recursive ascent parser for such a grammar would need to read a token in the states that perform reduce actions. It wouldn't actually shift that token, so it would need to save it somewhere for the next shift action to use. For example, LALRPOP generates parsers that return `(Token, Nonterminal)` tuples.

Another issue is that the parser has some amount of code duplication. The states `goal_start`, `array_open`, and `elements_element_comma` each start with a `match` on the tokens of a `value`. (The copy in `array_open` also includes an arm for `Token::RightBracket`.)

```rust
let token = lex.token();
let value = match token {
    Token::Number(number) => value_number(lex, number)?,
    Token::LeftBracket => {
        let array = array_open(lex)?;
        value_array(lex, array)?
    }
    _ => return Err(ParseError),
};
```

In a full JSON grammar, this gets even larger, because it also includes arms for strings, booleans, null, and objects. In a table-driven parser this doesn't make much of a difference, because each state takes up a full row in the table regardless, and the table is generated. In a hand-writen parser, it's kind of a pain. It may be less of an issue in some grammars, and it could be factored into a macro, but factoring it into a function is trickier because it is often slightly different, like in `array_open`.

On the other hand, bottom-up parsers are (slightly) more powerful than top-down parsers. Intuitively, this is because they don't decide which grammar rule to use until they've already seen its entire body. The ability to hand-write, rather than generate, a bottom-up parser means it can be applied to a piece of a larger grammar, even if the rest of the parser is top-down.

I've expanded the code here into a full [JSON parser](https://github.com/rpjohnst/json-parser/blob/recursive-ascent/src/parse.rs), along with a supporting lexer. It's not production quality, but it shows how to apply recursive ascent in a few more ways. You can also look at the output of [LALRPOP](https://github.com/lalrpop/lalrpop) in recursive ascent mode.
