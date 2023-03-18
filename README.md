# genped

A **minimalist** parser generator for Rust.

Selling points:

- No action code
- Only one file needed
- No need to learn a new language
- Easy to use your own lexer (in fact, that's the only option!)

## Example

Rather than specify a BNF grammar, you specify
a concrete syntax tree (CST).

The syntax of the CST specification langauge
is nearly identical to Rust.

The only parts that should look unfamiliar to you
are the terms that involve the `$` symbol.
We will explain this later.

Input (e.g., `arithmetic.genped`):

```rust
#[derive(Debug, Clone)]
enum Sum {
    Product(Product),
    Sum(Box<Sum>, $Plus, Product),
}

#[derive(Debug, Clone)]
enum Product {
    Term(Term),
    Product(Box<Product>, $Asterisk, Term)
}

#[derive(Debug, Clone)]
enum Term {
    Number($Number),
    Parenthesized($LParen, Box<Sum>, $RParen),
}

// Terminals

use super::lexer::TaggedToken;

$ = enum TaggedToken {
    Plus(()),
    Asterisk(()),
    LParen(()),
    RParen(()),
    Number(String),
}
```

Output (e.g., `arithmetic.rs`):

```rust
use super::lexer::TaggedToken;

#[derive(Debug, Clone)]
pub enum Sum {
    Product(Product),
    Sum(Box<Sum>, (), Product),
}

#[derive(Debug, Clone)]
pub enum Product {
    Term(Term),
    Product(Box<Product>, (), Term)
}

#[derive(Debug, Clone)]
pub enum Term {
    Number(String),
    Parenthesized((), Box<Sum>, ()),
}

impl Sum {
    pub fn parse_tokens<I>(s: I) -> Result<Self, TaggedToken> where I: IntoIterator<Item = TaggedToken> {
        // <auto-generated parser code...>
    }
}
```

In the next section, we'll explain the `$`.

## Lexer integration

As you probably could guess from the signature
of `Sum::parse`, genped's parser does not directly parse `str`s--it instead parses a sequence of
_tokens_.

_You_ are in control of defining what a token is.
The only restriction is that the token type must have the form:

```rust
enum SOME_NAME {
    TAG_NAME_1(TYPE_1),
    TAG_NAME_2(TYPE_2),
    // ...
    TAG_NAME_N(TYPE_N),
}
```

genped will replace all types of the form
`TAG_NAME_i` with the corresponding `TYPE_i`.

You must tell genped what type to use as your
token type by writing a `$ =` statement.

### Example:

Consider the previous example.
We specified the token type to be `TaggedToken`,
by writing:

```rust
$ = enum TaggedToken {
    Plus(()),
    Asterisk(()),
    LParen(()),
    RParen(()),
    Number(String),
}
```

Then, in the output, we saw

```rust
#[derive(Debug, Clone)]
pub enum Sum {
    Product(Product),
    Sum(
        Box<Sum>,
        /* previously `$Plus` */ (),
        Product,
    ),
}

#[derive(Debug, Clone)]
pub enum Product {
    Term(Term),
    Product(
        Box<Product>,
        /* previously `$Asterisk` */ (),
        Term,
    ),
}

#[derive(Debug, Clone)]
pub enum Term {
    Number(/* previously `$Number` */ String),
    Parenthesized(
        /* previously `$LParen` */(),
        Box<Sum>,
        /* previously `$RParen` */(),
    ),
}
```

As you can see, each `TAG_NAME_i` is replaced
with its corresponding inner type `TYPE_i`.

For example, since we wrote

```rust
$ = enum TaggedToken {
    // ...
    Number(String),
}
```

...every instance of `$Number` was replaced with `String`.

Similarly, since we wrote

```rust
$ = enum TaggedToken {
    // ...
    Plus(()),
}
```

...every instance of `$Plus` was replaced with `()`.

## Pros:

Incredibly simple.

- No action code
- Only one file needed
- No need to learn a new language

## Cons:

- Concrete syntax tree (CST) specification
  is not as pleasant to read as a BNF grammar.
  - HOWEVER: You would have had to write a
    syntax tree in Rust _anyway_, so you
    might as well derive the grammar from it
    for no additional scost.
- The lack of action code support means that
  you will probably have to perform more
  manipulation elsewhere in your code.
  - HOWEVER: We believe this is actually a
    _benefit_. genped aims to do One Thing Well--parse.
