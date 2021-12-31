---
title: From recursive descent to LR parsing
---

[Recursive descent parsers](https://en.wikipedia.org/wiki/Recursive_descent_parser) are attractive for their simplicity. A collection of functions, one for each part of your language, looks at the next token to decide what to parse next. The structure and control flow of the program matches the structure of the grammar, so you can write a parser by hand in your favorite language and use familiar tools for testing and debugging.

Infix expressions are a common pain point for recursive descent parsers, but there is a common solution that mostly preserves this direct style: [Pratt parsing](https://en.wikipedia.org/wiki/Operator-precedence_parser#Pratt_parsing) or "top-down operator precedence." My goal here is to take your familiarity with recursive descent and Pratt parsing to build an intuition for the more-general but often-mysterious [LR parsing](https://en.wikipedia.org/wiki/LR_parser).

> If you're new to Pratt parsing, I'll cover some of it in this post, but here are some introductions I've appreciated:
> * [Pratt Parsers: Expression Parsing Made Easy](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/)
> * [Simple but Powerful Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html)

### Parser control flow and pushdown automata

There are two important kinds of control flow in a recursive descent parser:
* Within a function, the sequence of checks, branches, and loops corresponds to the right-hand sides of the rules in the language's grammar.
* Calls between functions correspond to occurrences of nonterminal symbols (symbols on the left-hand sides of rules) in the language's grammar.

For example, consider this grammar (extended with regular expressions, just for fun) for JSON numbers and arrays, and pseudo Rust code for a recursive descent parser:

```
value = NUMBER
      | array
array = "[" (value ("," value)*)? "]"
```

```rust
fn value() -> Json {
    match peek() {
        NUMBER => { Json::Number(bump(NUMBER)) }
        L_BRACKET => { array() }
        _ => { /* Unexpected token */ }
    }
}

fn array() -> Json {
    let mut values = vec![];

    bump(L_BRACKET);
    while peek() != R_BRACKET {
        values.push(value());
        if !eat(COMMA) { break; }
    }
    bump(R_BRACKET);

    Json::Array(values)
}
```

We can also express this more abstractly as a [pushdown automaton](https://en.wikipedia.org/wiki/Pushdown_automaton), or a finite state machine whose edges can match input symbols *and* push and pop elements from a stack. There are a lot of ways to construct a PDA for a grammar, but our approach here will mirror the control flow and call stack of our recursive descent pseudocode.

To begin with, here are the function-local control flow graphs:

<img src="/images/lr-control-flow/json-cfg.svg" height="210" alt="JSON parser control flow">

Solid edges match input symbols, while dashed edges match nonterminals. Final states have a double border. To turn this into a PDA, we'll need to convert the dashed edges into explicit stack operations. Edges are now labeled `x ; A / B`, to read an input symbol `x`, pop a stack symbol `A`, and push a stack symbol `B`, with &#x3F5; ("epsilon") standing for the empty string. Parsing is complete when we reach a final state *and* have an empty stack.

<img src="/images/lr-control-flow/json-pda.svg" height="210" alt="JSON parser PDA">

This is a bit of a rat's nest, but if you pick it apart you can see that any state at the source of a dashed edge now has a new edge to push a symbol and transition to the called function, and the end of that function has a new edge to pop the symbol and return to the target of the dashed edge.

For example, parsing the input `[1, [], 3]` looks like this:

| Input        | Stack                | Action                                 |
| ------------ | -------------------- | -------------------------------------- |
| `[1, [], 3]` |                      | Push `V`, read `[`                     |
| `1, [], 3]`  | `V`                  | Push `A`, read `1`                     |
| `, [], 3]`   | `A V`                | Pop `A`, read `,`                      |
| `[], 3]`     | `V`                  | Push `A`, push `V`, read `[`           |
| `], 3]`      | `V A V`              | Read `]`                               |
| `, 3]`       | `V A V`              | Pop `V`, pop `A`, read `,`             |
| `3]`         | `V`                  | Push `A`, read `3`                     |
| `]`          | `A V`                | Pop `A`, read `]`                      |
|              | `V`                  | Pop `V`, done                          |

In the general case, a function might be called from multiple places. That's why we have multiple distinct stack symbols---they tell us where to return to, just like the return address on the call stack of the recursive descent parser.

### Pratt parsing

Pratt parsing solves two distinct problems faced by recursive descent parsers:
* Naive recursive descent doesn't support left recursion (rules which start with their own nonterminal symbol, like `expr = expr "+" NUMBER`), which are the most natural way to describe infix operators.
* Naive recursive descent needs to know which rule it is parsing with only one token of lookahead, but infix operators don't appear until arbitrarily far into the expression.

> I say "naive" because Pratt parsing isn't the only place these problems are solved---most recursive descent parsers deal with them somehow. It's just a nice illustrative example where they're all solved in one place.

In this JSON PDA, we can already see signs of the first problem: the automaton must take some number of &#x3F5; edges that just push or pop stack symbols (in recursive descent this means calling or returning from functions) before seeing all the possible inputs it needs to consider. For example, from the first state it may immediately read a `NUMBER`, or it may push a `V` before reading a `"["`.

This is not actually an issue for JSON, where this extra exploration is limited enough that we can hard-code it at the call sites of `array` and `value` by mechanically following those &#x3F5; edges ahead of time. But consider this grammar and PDA for left-associative addition expressions:

```
expr = expr "+" NUMBER
     | NUMBER
```

<img src="/images/lr-control-flow/add-pda.svg" height="150" alt="sum parser PDA">

Using the same approach of mechanically following the &#x3F5; edges to find a set of tokens to read next, we can see that the first state must read a `NUMBER`, and the last state must read a `"+"` or the end of the input. And indeed, we expect an alternating sequence of `NUMBER` and `"+"` tokens. This PDA even makes it obvious that the sequence must start with a `NUMBER`.

However, we no longer have enough information to determine what to do with the stack. Seeing a `NUMBER` might mean we should parse the `expr = NUMBER` rule, or it might mean we should push an `E` to start parsing the `expr = expr "+" NUMBER` rule. And if we pick the second option, we're back where we started and need to make the same decision again! For this to work out, the number of `E` symbols we should push depends on the number of unprocessed `"+"` tokens in the input.

In these terms, Pratt parsing tackles this problem by relaxing the association between the parser's call stack and the PDA's stack. We can't know up front how many calls to make, so instead we'll just *pretend* we made exactly the right number and then discover what that number was as we go---each time we parse a complete `expr`, if we see another `"+"`, it's as if we have just returned from one more `expr()` call and are ready to continue the `expr = expr "+" NUMBER` rule:

```rust
fn expr() -> Expr {
    // Make some unknown number of imaginary "calls" to `expr()`, then parse the
    // `expr = NUMBER` rule.
    let mut lhs = Expr::Number(bump(NUMBER));

    // Each time there is a `"+"`, "return" from one more imaginary "call," then
    // continue parsing a `expr = expr "+" NUMBER` one level up.
    while eat(PLUS) {
        let rhs = Expr::Number(bump(NUMBER));
        lhs = Expr::Add(lhs, rhs);
    }

    // When there are no more `"+"` tokens, return from the top-level "call."
    // Depending on how many `"+"` tokens we read, the result may be either a
    // number or an add expression.
    lhs
}
```

Despite this stack trickery, the resulting pseudocode looks just like what we would write from scratch, without thinking about PDAs.

In general, Pratt parsing supports multiple operators. If they have different precedence levels, we can handle that using actual function calls like normal. But at the same precedence level, we run into the second problem---consider what happens when we add subtraction to the grammar:

```
expr = expr "+" NUMBER
     | expr "-" NUMBER
     | NUMBER
```

<img src="/images/lr-control-flow/sub-pda.svg" height="230" alt="subtraction parser PDA">

Not only do we need to take an arbitrary number of &#x3F5; edges up front, we also need to decide for each one whether it corresponds to addition or subtraction, so the automaton can return to the right place.

Pratt parsing resolves this by exploiting the fact that, while the two rules share a common prefix (which is why the PDA needs to choose `P` vs `M` up front), they diverge in the middle with different operators. We can use the operator to decide which rule to "return" into:

```rust
fn expr() -> Expr {
    let mut lhs = Expr::Number(bump(NUMBER));

    loop {
        match peek() {
            PLUS => {
                bump(PLUS);
                let rhs = Expr::Number(bump(NUMBER));
                lhs = Expr::Add(lhs, rhs);
            }
            MINUS => {
                bump(MINUS);
                let rhs = Expr::Number(bump(NUMBER));
                lhs = Expr::Sub(lhs, rhs);
            }
            _ => { break }
        }
    }

    lhs
}
```

The full version of Pratt parsing uses two more tricks, which make it nicer to work with but aren't directly relevant to this post. A quick summary for completeness:
* Infix operator rules have a common suffix as well as a common prefix, so Pratt parsing can combine them instead of handling them separately per-operator.
* Naive recursive descent would use a separate function for each operator precedence level, but they all use the same control flow, so Pratt parsing combines them into one function with a parameter for the current precedence.

### LR parsing

PDAs may or may not be the most intuitive way to think about Pratt parsing, but they are a good way to understand LR parsing, which gets its power from the same two tricks! In fact, I would argue that the name "top-down operator precedence" is a misnomer, and that Pratt parsing is actually *bottom-up* just like LR: it parses the leaves of the AST first, then combines them into larger and larger trees.

LR parsing is usually presented as a table-based algorithm, constructing a PDA similar to ours ahead of time, then parsing by looking up its edges based on the current state and input token. It's also possible to generate code that runs the PDA directly, which is sometimes called [recursive ascent](https://en.wikipedia.org/wiki/Recursive_ascent_parser). (I tried [writing one of these by hand](/blog/2018/04/08/recursive-ascent) once.) This is one of the benefits of thinking in terms of PDAs---you can apply what you figure out to multiple implementation styles.

There are three major differences between the PDAs we constructed above (which mirror the control flow of recursive descent) and the automata used for LR parsing. One is a tweak to how the stack is used, and the other two are fairly standard state machine transformations that implement the Pratt parsing tricks.

#### The LR stack

Above, we used the PDA stack to keep track of where to continue after finishing a rule, with an edge back to each call site. LR uses a different scheme, which is simpler in some ways but doesn't match recursive descent as closely.

Instead of per-call site return edges, an LR automaton pushes each state it enters onto the stack, and when it reaches the end of a rule it pops the group of states from that rule all at once, in a "reduce" action. This means rules need to have fixed lengths so we know how many states to pop, but it also means the edge labels no longer need to include stack operations. We also bring back the nonterminal edges, so we know where to go after reduce actions.

The automaton for the JSON grammar (now lightly refactored to give the rules fixed lengths) looks like this:

```
value = NUMBER
      | array
array = "[" elems "]"
      | "[" "]"
elems = elems "," value
      | value
```

<img src="/images/lr-control-flow/json-lr-epsilon.svg" height="230" alt="JSON LR parser with &#x3F5; edges">

At this point, the dashed &#x3F5; edges still match recursive descent's function calls, which has one subtle complication: to make the fixed-length rule scheme work, you would need to avoid pushing states reached via those &#x3F5; edges. The actual LR algorithm doesn't need to worry about this, though, because of how it implements the first Pratt parsing trick:

#### &#x3F5; closure

The first trick was to skip "call" edges and rely only on the "return" edges (or reduce actions) edges to keep track of what rule we're in. A finite state machine with &#x3F5; edges can always be converted to an equivalent state machine without them, which will bake this trick directly into the automaton.

This is done by finding the "&#x3F5; closure" of each state, or the set of all states reachable by following &#x3F5; edges. Then we can introduce shortcut edges past them like this:

<img src="/images/lr-control-flow/json-lr-closure.svg" height="280" alt="JSON LR parser without &#x3F5; edges">

This is, again, a bit of a rat's nest. The main thing to notice here is that, except for the `value` rule, all the rules' initial states have disappeared. We skip past them every time we enter a new rule, directly from its "call" site.

#### Determinization

The second trick was to merge common prefixes between rules. A nondeterministic finite state machine, or one with multiple edges leaving a state with the same symbol, can always be converted to an equivalent deterministic state machine, which again will bake this trick directly into the automaton.

This is done by introducing a new state for each *combination* of states that the nondeterministic state machine might be in at the same time. For our JSON grammar, this actually shrinks the automaton a bit:

<img src="/images/lr-control-flow/json-lr-deterministic.svg" height="240" alt="Deterministic JSON LR parser">

The result in this case is still probably too messy to be worth writing by hand, and in my opinion these two transformations are what make LR parsers so opaque---this state machine no longer corresponds directly to the grammar. On the other hand, in the world of PDAs they are the same transformations we used for Pratt parsing, where applying them selectively is a powerful tool.

### LR(k) parsing

What we've built so far is an LR(0) automaton---one that uses 0 tokens of lookahead to determine whether to reduce. This is sufficient for the grammars we have seen so far, but it is not enough to handle operator precedence. Let's take a look at one final grammar, and the automaton we get from &#x3F5; closure and determinization:

```
expr = expr "+" term
     | term
term = term "*" NUMBER
     | NUMBER
```

<img src="/images/lr-control-flow/expr-lr-conflict.svg" height="210" alt="Expression LR parser with a conflict">

Something funny is going on with the final state with the two incoming `term` edges. Imagine parsing the expression `1 + 1`. After taking the first `NUMBER` edge, we reduce back to the first state and take the `term` edge. Without lookahead, we have no way of knowing whether we are in the middle of a `term = term "*" NUMBER` rule, or have just finished an `expr = term` rule!

This is called a "shift/reduce" conflict---an ambiguity between reading more input (which LR calls "shifting" a token) or reducing the state stack. LR's clever approach to the PDA stack has come back to bite us. What we want to do, and what Pratt parsing does, is peek at the next token when faced with this choice. If it is a `"*"`, then we are in the middle of a `term = term "*" NUMBER` rule and we continue at the same precedence level. Otherwise, it must be a `"+"` and we can "return" up to an `expr = expr "+" term` rule.

> There is a lot of variety in how LR parsers pull this off, which is outside the scope of this post. So here are some places you might start if you want to learn more:
>
> "Canonical" LR(k) parsing resolves this conflict by tracking which strings of k tokens we expect after each "call" to a rule. (k is almost always 1, and in fact any grammar that LR(k) supports can be made to work with LR(1). In the other direction, there is an extension called "LR-regular" where these strings are generalized into [regular languages](https://en.wikipedia.org/wiki/Regular_language)!) Each "call" with a different set of expected lookahead tokens gets its own separate copy of the states for the associated rules, which can propagate down through any additional "calls."
>
> All these copies can lead to a much larger state machine, so there are several flavors of LR(k) that give up some of this disambiguation power to avoid making quite so many copies. "[Simple LR](https://en.wikipedia.org/wiki/Simple_LR_parser)" or "SLR" does not copy any states, and disambiguates purely based on whether the lookahead token can follow the rule's nonterminal anywhere in the grammar. "[Lookahead LR](https://en.wikipedia.org/wiki/LALR_parser)" or "LALR" is slightly more precise, and propagates the same information as Canonical LR but merges lookahead information rather than copying states. There are also algorithms like "lane tables" or "IELR" that can detect when copying will help, and fall back to LALR otherwise.
>
> Yet another approach is to accept these ambiguities in the automaton, and process them all. This class of algorithms can even parse ambiguous grammars, producing a parse "forest" instead of a tree.

I hope this made sense! Thinking in terms of abstract state machines like this is a great tool for understanding and refactoring all kinds of programs at a high level.
