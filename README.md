# nom

nom, eating data byte by byte

nom is a parser combinator library with a focus on safe parsing,
streaming patterns, and (as much as possible) zero copying.

The code is available on [Github](https://github.com/Geal/nom)

### Getting started

#### Minimal parser

Here is an example of the most basic parser you can write. It takes a bunch
of bytes and looks for the binary representation of the string "nom".

```rust
#[macro_use]
extern crate nom;

// Generates something like:
// fn cool_word(i: &[u8]) -> nom::IResult<&[u8], &[u8]> { ... }
named!(cool_word<&[u8]>, tag!("nom"));

fn main() {
    println!("{:?}", cool_word(b"nom is awesome"));
}

// Outputs: Done([32, 105, 115, 32, 97, 119, 101, 115, 111, 109, 101], [110, 111, 109])
```

The heart of this is the `named!` macro, which generates a function, which 
returns a `nom::IResult`. This is an alternative type for `std::Result`:

```rust,ignore
pub enum IResult<I, O, E = u32> {
   Done(I, O),          /* I holds the rest of the input, O holds the parsed output */
   Error(Err<E>),       /* Error holds the error. It's possible to have sub-errors. */
   Incomplete(Needed),  /* Incomplete is for parsing streams (via network) */
}
```

The last three bytes hold the value `nom` as bytes (nom only occured once in
our string). A function in nom can return a specific type (in this case, it 
just returned `IResult`). You can, for example use the following function to 
return an IResult of `str` from the parser, instead of `&[u8]`:

```rust,ignore
// Takes &str as input (first argument), produces an IResult of type &str (second argument)
named!(cool_word<&str, &str>, tag!("nom"));
// outputs: Done(" is awesome", "nom")
```

#### Repeated and alternative parsers

One of the core strengths of nom is the ability to combine sub-parsers into 
larger, mor complicated parsers. For this to work, we have to define two 
functions: One to parse a singular "nom" string, and one to expect that the 
"nom" is at least __once__ contained in our input:

```rust
# #[macro_use]
# extern crate nom;
# 
// same as before
named!(cool_word<&str, &str>, tag!("nom"));

// the many1 macro produes a vector of T
named!(many_cool_words <&str, Vec<&str>>,
   many1!(cool_word));

fn main() {
    println!("{:?}", many_cool_words("nomnomnomnomnomnom oh yes!"));
}

// Output: Done(" oh yes!", ["nom", "nom", "nom", "nom", "nom", "nom"])
```

The macros `many1!`, `many0` or `many`, `many_m_n` and `many_till` can be 
used for repeated content. `many1` for example has the restriction that the
given parser must match at least once, if the string did not contain the word
"nom", it would fail.

Alternative parsers can be written using the `alt!` macro. It tries the first
parser and if it fails, tries the second parser (seperated by a vertical bar).
You can even try more than two parsers, by chaining them via a `|`.

```rust
# #[macro_use]
# extern crate nom;
# 
named!(parse_many_as <&str, Vec<char>>,
   many1!(char!('a')));

named!(parse_many_bs <&str, Vec<char>>,
   many1!(char!('b')));

named!(many_as_or_bs <&str, Vec<char>>,
     alt!(parse_many_as | parse_many_bs));

fn main() {
    println!("{:?}", many_as_or_bs("aaaaaaaples"));
    println!("{:?}", many_as_or_bs("bbbaaaaples"));
}

// Done("ples", ['a', 'a', 'a', 'a', 'a', 'a', 'a'])
// Done("aaaaples", ['b', 'b', 'b'])
```

Here, we also introduced the `char!` macro for single characters.
Note that the order of parsers are important in the `alt!` macro. This will,
for example, not parse the "b"s in our string and stop at the first "failed" 
parsing result:

```rust,ignore

fn main() {
   println!("{:?}", many_as_or_bs("aaabbbaaaabbbples"));
}

// Done("bbbaaaabbbples", ['a', 'a', 'a'])
```

#### Capturing / serializing input to structs

Usually when using nom you have some sort of input that you want to serialize
to a data structure. `nom` is a zero-copy parser, meaning that it will build
references to the data you give it, not clone the data.

Serializing blocks of data that follow a specific order is done using the 
`do_parse!` macro. This macro takes a list of parsers, seperated by a `>>`
and runs them sequentially on your data. __Note:__ In older versions of `nom`,
the `>>` was a tilde character (`~`). This still works (same effect), but the
`>>` should be used.

```rust
# #[macro_use]
# extern crate nom;
# 
named!(many_as <&str, Vec<char>>,
   many1!(char!('a')));

named!(many_bs <&str, Vec<char>>,
   many1!(char!('b')));

named!(as_space_then_bs <&str, ()>, do_parse!(
    many_as >>
    char!(' ') >>
    many_bs >>
    () ));

fn main() {
    println!("{:?}", as_space_then_bs("aaa bbbbbbbbbb"));
}

// Done("", ())
```

`do_parse` uses the last seperated item as a return type that it returns from
the generated function. In the above case, this was simply `()`, a void type.
However, you can bind named fields to the parsers (using the `name: parser` syntax)
and then capture the output to a struct.

```rust
# #[macro_use]
# extern crate nom;
# 
# pub struct ImportantStruct {
#    a_vec: Vec<char>,
#    b_vec: Vec<char>,
# }
# named!(many_as <&str, Vec<char>>,
#   many1!(char!('a')));
#
# named!(many_bs <&str, Vec<char>>,
#    many1!(char!('b')));
# 
//[...]
// see previous example for the missing code

named!(serialize_a_b_struct<&str, ImportantStruct>, do_parse!(
    captured_as: many_as >>
    char!(' ') >>
    captured_bs: many_bs >>
    (ImportantStruct { 
           a_vec: captured_as, 
           b_vec: captured_bs
    }) ));

fn main() {
    let serialized_struct = serialize_a_b_struct("aaa bbbbbbbbbb").unwrap();
    println!("{:?}", serialized_struct.1.a_vec);
    println!("{:?}", serialized_struct.1.b_vec);
}

// ['a', 'a', 'a']
// ['b', 'b', 'b', 'b', 'b', 'b', 'b', 'b', 'b', 'b']
```

Notice: The captured struct has to be wrapped in `()` and the resulting type
has to be unwrapped (for this example), resulting in a `(&str, ImportantStruct)`.
So we have to use `.1` to get our struct.

#### Conditional parsing, serializing enums, convenience parsers

Now that you know how to assign fields, you can write conditional parsers using
`opt!` and `cond!`. `opt!` returns an `Option` with `Some(IResult)` if the given
parser succeded.

```rust
# #[macro_use]
# extern crate nom;
# 
use std::str::FromStr;
use nom::digit;

named!(negative_i32 <&str, i32>, do_parse!(
    neg: opt!(char!('-')) >>
    digs: cond_reduce!(neg.is_some(), 
        map_res!(digit, i32::from_str)) >>
    (-digs) ));

fn main() {
    println!("{:?}", negative_i32("1023"));
    println!("{:?}", negative_i32("-55"));
}

// Error(CondReduce)
// Done("", -55)
```

This is a parser that only parses negative numbers. It looks for the char "-" 
as the first input, then stores the result in `neg`. The `cond_reduce!` macro only 
executes if the first argument is `true` - otherwise it returns an Error. Of
course you could also use an `if` statement inside the return type.

The `nom::digit` parser is a convenience parser for basic types, provided by nom.
Nom has convenience parsers for integers (with various endianness) and strings.

You can also serialize directly to enums by using a closure or extract bytes
using the `take!`:

```rust
# #[macro_use]
# extern crate nom;
use nom::Endianness;

#[derive(Debug)]
struct Header<'a> {
  endianness:   Endianness,
  version:      &'a[u8],
}

named!(parse_header<&[u8], Header>, 
    do_parse!(
        endianness: alt!(
          tag!(b"v") => {|_| Endianness::Little } |
          tag!(b"V") => {|_| Endianness::Big } ) >>
        version: take!(3) >>
        (Header {
          endianness:   endianness,
          version:      version,
        })
    )
);

fn main() {
    println!("{:?}", parse_header(b"v244"));
}

// Done([], Header { endianness: Little, version: [50, 52, 52] })
```

#### Simple error handling

To handle errors with `nom`, you can use the `add_return_error!` macro when 
you want to make your own error code (all errors in `nom` are `i32`) and 
later on use pattern `error_to_list!` to get a `Vec` over the error codes.
Then you can implement error handling how you want. Or you can use the 
error_chain` crate to transform noms `ErrorKind` into a string for an error message. 

See the [Error management](error_management.html) page for more details

#### Simple CSV parser

This is an example of how to combine parsers to make a CSV parser. There are 
more complicated formats, but this should give you an idea of how parsers can
be combined to form bigger ones. Each sub-parser is a seperate function and 
can thus be tested on its own.

```rust
# #[macro_use]
# extern crate nom;
# 
named!(csv_escaped_char<&str, char>, alt!(
    tag!("\"\"") => { |_| '"' } |
        none_of!("\"")));

named!(csv_style_str<&str, String>, do_parse!(
    char!('"') >>
    chars: many0!(csv_escaped_char) >>
    char!('"') >>
    (chars.into_iter().collect()) ));

fn main() {
    println!("{:?}", csv_style_str(r#""some string" is cool"#));
    println!("{:?}", csv_style_str(
        r#""some string with "" internal quotes!" is also cool"#));
}
```



### Further Reading

- [Nom guides](home.html)
- [The design of nom](how_nom_macros_work.html)
- [How to write parsers](making_a_new_parser_from_scratch.html)
- [Error management](error_management.html)
- [Migrating from nom 1.0 to 2.0 / higher](upgrading_to_nom_2.html)
- [FAQ](FAQ.html)

Also, you can join us on `#nom` on our IRC channel at `irc.mozilla.org:6667`.
