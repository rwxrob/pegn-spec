# PEGN: Parsing Expression Grammar Notation

PEGN is a language for defining languages. More precisely, it is a
universal notation for expressing any grammar --- including natural
language --- in a way that is easy to parse cognitively and
programmatically without any specific application or implementation in
mind. It builds on the best of existing meta and data structure
languages such as PEG, ABNF, EBNF, and JSON.

## Motivation

As technology increases in complexity and the need for better
human-computer interaction becomes more pronounced, creating language
grammars quickly and simply has become a critical need. PEGN is designed
to meet this need. By allowing any data to be represented as a grammar
and breaking it down into a universal form data can be combined,
composed, and analyzed in remarkable ways. 

Whether it be simply counting all the words in a document, creating a
simple query language to make searching logs easier, coding a
human-friendly interface to an otherwise complicated web API,
simplifying the parsing of a form of a common Markup, implementing a
full programming language that leverages the LLVM to
quickly create a highly-performant compiler, or developing a binary
language for moisture evaporators, PEGN addresses these needs by
prioritizing the creation of other language grammars without weighing
them down with any specific bias about how they should be implemented. In fact,
PEGN is so flexible it can be used to define spoken and written natural
languages and musical notation as well.

### PEG Grammars are Exploding, But Inconsistent

Since 2004 PEG grammars have exploded in popularity but the only thing
that remains consistent is the wide variety of differences in their
implementation and interpretation. Bryan Ford's "example" PEG grammar is
all but ignored as people continue to build their own syntaxes that have
very little resemblance to the original and are more implementation code
than PEG. This is demonstrated by many projects that contain *both* a
grammar file for becoming acquainted with the syntax and another
virtually identical file containing additional implementation specific
code added to it so that a highly specialized code generator can use it.
This redundancy and specialization are not only less sustainable but also
highly rigid and counter-productive.

PEGN is a language grammar specification that *does not allow*
implementation code  so that the resulting grammar specifications stand
on their own allowing the creation of any variety of linters and code
generators in different language implementations, even different design
variations in the *same* implementation language (AST, event callbacks,
etc.)

### Original PEG Lacks Specificity

For years ABNF and EBNF provide excruciating levels of specificity in
their grammars but lack the obvious advantages of ordered priority and
the simplicity of the *original* ASCII PEG grammar. For example, PEGN
adds `Count` and `MinMax` to provide limits and adds Unicode tokens.

The hope is that the PEGN language itself can become a more explicit,
better performing, and readable replacement for many grammar
meta-languages and inline regular expressions. Code generators
producing parsers of different types and in different implementation
languages can be created from the same grammar specification expressed
in PEGN. PEGN parsers and standard libraries can even provide highly
optimized handling of PEGN grammars included directly in code as strings
and constants much like compiled regular expressions are handled today
but with much greater clarity and efficiency.

## Examples

Perhaps the best example is the PEGN grammar itself, which is [specified
in PEGN](https://raw.githubusercontent.com/pegn/spec/master/grammar.pegn).

Here's another example, the JSON specification in PEGN:

```pegn
# JSON-RFC-8259 (v1.0.0) github.com/pegn/grammars/json

Grammar <-- ws* Value ws*

Value    <- ws* (Object / Array / Number / String 
          / True / False / Null) ws*

Object  <-- '{' ws* Member (ws* ',' ws* Member)* ws* '}'
Array   <-- '[' ws* Value (ws* ',' ws* Value)* ws* ']'
Number  <-- MINUS? Integer (DOT DIGIT+)? ('e' / 'E') sign DIGIT+ 
String  <-- DQ (Escaped / [x20-x21] / [x23-x5B] / [x5D-x10FFFF])* DQ
True    <-- 'true'
False   <-- 'false'
Null    <-- 'null'

Member  <-- String ':' Value
Integer  <- '0' / [1-9] DIGIT*
Escaped  <- BKSLASH ('b' / 'f' / 'n' / 'r' / 't' / 'u' hex{4}
          / DQ / BKSLASH / SLASH)
```

## Abstract Syntax Tree

PEGN includes a strict JSON AST serialization format which is itself
included in the [grammar
specification](https://raw.githubusercontent.com/pegn/spec/master/grammar.pegn). PEGN defines an
optional human-friendly version as well as the core compressed version. 

### AST (Short-Form)

[[PEGN Specification of AST
Short-Form](https://github.com/pegn/spec/tree/master/ast/short)]

The compressed, short-form of the JSON AST uses integers to identify
node types (as opposed to words as found in the long form) and is
optimized for the best parsing speeds and most efficient storage and
transfer. It *must* be used whenever an AST is stored or transferred
over the network in any way. 

### AST (Long-Form)

[[PEGN Specification of AST
Long-Form](https://github.com/pegn/spec/tree/master/ast/long/)]

The human-friendly JSON long-form AST is used only for debugging and
documentation and need never be parsed itself (despite it being JSON as
well).

Here is a small bit of the resulting abstract syntax tree in JSON (long
form):

```json
["Grammar", [
  ["Meta", [
    ["Lang", "JSON"],
    ["LangExt", "RFC-8259"],
    ["MajorVer", "1"],
    ["MinorVer", "0"],
    ["PatchVer", "0"],
    ["Home", "github.com/pegn/grammars/json"],
    ["EndLine", "\n"]
  ]],
  ["EndLine", "\n"],
  ["SchemaDef", [
    ["CheckId", "Grammar"],
    ["Expression", [
      ["Sequence", [
        ["Plain", [
          ["ClassId", [
            ["ResClassId", "ws"]
          ]],
          ["MinZero", "*"]
        ]],
...
```

### Parent and Terminal Nodes

There are two meta-types of PEGN AST Nodes *parents* and *terminals*.
Terminals contain nothing but a literal value. Parents contain nothing
but other parent and terminal Nodes. This constraint prevents
unnecessary complications. 

#### Thinking Differently About Grammar Design

The parent and terminal limitations can present cognitive challenges
when designing a PEGN grammar for those coming to PEGN from other
methodologies. Ultimately, however, what appear to be challenges result
in a more comprehensible grammar specification.

Consider the following example using a problem with the `Copyright` node
when it was first added.

```pegn
# humm ...
Copyright <-- '# Copyright ' (!EndLine any)+ EndLine
EndLine   <-- LF / CRLF / CR
```

Given the following sample data the error becomes apparent:

```
# Copyright 2020 Robert S Muhlestein (rwx@robs.io)
```

Notice the AST below does not contain a node containing any relevant
text from the `Copyright` making it virtually useless.

```json
["Copyright", [
  ["EndLine", "\n"]
]]
```

This problem occurs because both `Copyright` and `EndLine` are
"significant" (`SchemaDef`) nodes and therefore the literal text is
discarded from the AST when the `EndLine` definition triggers the parser
of `Copyright` to become a *parent* instead of a *terminal* node. Since
parents have no literal value we need more to capture the text. PEGN's
node tree model is deliberately minimal requiring us to improve our
grammar by making it more explicit.

To fix this problem and preserve the literal text from `Copyright` we
must define a significant *terminal* node type to contain it. Turns out
we already have one, the `Comment` `SchemaDef` defined elsewhere in the
grammar.

```pegn
# better!
Copyright <-- '# Copyright ' Comment EndLine
Comment   <-- (!EndLine any)+
EndLine   <-- LF / CRLF / CR
```

Having corrected the problem we now see that after parsing the AST
the relevant text of the `Copyright` is contained in the `Comment`
terminal node.

```json
["Copyright", [
  ["Comment", "2020 Robert S Muhlestein (rwx@robs.io)"],
  ["EndLine", "\n"]
]]
```

Another fix would be to have made `EndLine` insignificant (`CheckDef`), but
that would break other requirements of this specific grammar.

### No AST Node Attributes

Some rooted node tree data structure models allow for attributes. PEGN's
AST does not since attributes can more efficiently and precisely be
indicated by adding an additional Parent or Terminal Node type.

## Origins and History

PEGN was conceived and originally developed by [Rob
Muhlestein](https://robs.io) while creating grammars needed for the RWX
KnowledgeNet specification including Ezmark, Datamark, BaseQL, and
others. During Rob's [live-coding streams](https://twitch.com/rwxrob)
others jumped in to contribute providing valuable suggestions and merge
requests. PEGN's design has always been driven by practical needs making
it useful for other projects. It is designed for creating easy,
sustainable, readable grammars for any purpose. If you are planning
anything involving any sort of grammar you might consider taking a look
at PEGN.

## PEGN Related Tools and Resources

* [Vim Plugin](https://github.com/pegn/pegn-syntax)
* [Emacs Plugin](https://github.com/pegn/pegn-mode)
* VSCode Extension
* Language Syntax Server
* `pegn` Linter
* *Create Your First Language*
* Searchable Grammar Registry

## Other Efforts

PEGN combines the best of [PEG](https://bford.info/pub/lang/peg/), ABNF,
EBNF, and JSON. It's closely based on the PEG example but with essential
additions including ABNF specificity, limit syntax, generator hints, and
a specific AST notation in JSON. Unlike other PEG approaches, 
PEGN grammars embrace any implementation language since they do not
contain language-specific code such as the following:

* [Peg/Leg](https://www.piumarta.com/software/peg/)
* [Go PEG](https://github.com/pointlander/peg)
* [Python](https://medium.com/@gvanrossum_83706/peg-parsers-7ed72462f97c)
* [Pegen](https://github.com/we-like-parsers/pegen)
* [Pidgeon](https://github.com/mna/pigeon)
* [Pest.rs](https://pest.rs)
* [PEGGED](https://github.com/PhilippeSigaud/Pegged)
* [ANTLR](https://www.antlr.org/)

## Creating Documentation

Formal PEGN grammar specifications are documented in an accompanying
`README.md` file that contains headings matching the definition
identifiers which in turn may contain one or more Markdown code blocks
containing examples that double as input tests against other code blocks
containing the expected AST in JSON long form (inspired by the
CommonMark project). This documentation can then be easily combined with
code generators using the PEGN Markdown grammar parser built into the
`pegn` utility (or any other tool capable of parsing Markdown).

### Linking Documentation with Definitions

The `README.md` file need only contain headings (of any depth) that
specifically match the identifier from the formal PEGN syntax file
(`<grammar>.pegn`). Headings must be surrounded by backticks for further
distinction and so that they have better rendered formatting and do not
trigger spelling errors. Nothing else is required to make the
association. This allows for maximum flexibility when authoring the
documentation including as much additional content unrelated to the
specific definitions. It is, however, a good idea to group definition
documentation into a single section. The suggested name is *Grammar
Definition Descriptions* but this is entirely up to the author.

## Legal Considerations

Wide adoption of any open source project requires a pristine legal
pedigree demonstrating that use of the software and all related content
under its licensed terms can be trusted to never put those using it at
legal risk --- especially legal entities other than individuals that are
more frequently targeted for such litigation. Given the origins of PEGN
outside of academia it seems prudent to provide a higher degree of
attention to these legal considerations that academic publishing might
have otherwise provided. The details of this section are designed to meet
those needs with the hope that more will choose to use PEGN openly,
collaborative, and privately.

## MIME Type

Currently the official MIME type associated with the `.pegn` suffix
related to PEGN grammars is `text/x-pegn` with the hope of eventually
obtaining `text/pegn` from IANA.

### Trademarks

Whether marked with ™ or not, PEGN, `pegn`, and the PEGN logo are
trademarks of Robert S Muhlestein who maintains an exclusive right to
authorize or control the use of these marks primarily to provide
assurance to the PEGN community of consistency with respect to content
of related tools and services. Use of PEGN trademarks is governed by
the same guidelines applied to the [Linux trademarks by the Linux
Foundation](https://www.linuxfoundation.org/trademark-usage/). This
enables the most permissive use without losing trademark control. Until
a more formal trademark usage guideline document can be drafted and
legally reviewed please use Linux Foundation guideline to
assist when making decisions about use of the PEGN trademarks.

### Copyright

Copyright (c) 2020 Robert S Muhlestein (rwx@robs.io)

Everything in this document and the rest of this
[repository](https://github.com/pegn/spec) falls under this copyright.
All contributions to the project also fall under this copyright per the
terms outlined in the Developer Certification of Origin
which every contributor accepts by making any commit or merge request to
the project git repository.

### Licenses

PEGN is licensed under both of the following licenses:

* [Apache License, Version 2](https://www.apache.org/licenses/LICENSE-2.0.html)
* [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/)

Any content of this document or repository that falls under the legal
definition of "software" is covered by the Apache-2 software license.
All other content is covered under CC-BY-4. This is particularly
important since the categorization of a meta-language such as PEGN's
formal grammar falls somewhere between the two and does not yet have
much legal precedent. Most such meta-languages (PEG, ABNF, EBNF, BNF)
are released through academic channels. Unlike other standards bodies no
one will ever be required to pay for access to read any PEGN standard
document.

### Patents

Any and all patent rights associated with PEGN and anything in this
repository are addressed under the Apache-2 license which contains
explicit language regarding the granting of patent rights.

### Contributing

Contributions are governed under the same DCO terms as those of the
Linux and Gitlab projects. Any commit or merge request to this project
Git repository legally constitutes acceptance of the terms of the
Developer Certificate of Origin and the release of all
rights by the contributor to the project and its owners.

### IETF RFC Consideration

As the PEGN standard continues to receive practical adoption an IETF
RFC may eventually be considered to further codify the standard in a way
that provides further assuages potential legal concerns.
