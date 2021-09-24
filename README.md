# query_format

This document describes the JSON format that can be used to query REST APIs.
The document is stuctured as a guide + Q&A.

### 1. Request: Top Level

```
{
  whereAnd: ...
  xor
  whereOr: ...
}
```

#### Q: maybe default `where` to `whereAnd` semantics?

**A:** explicit is better than implicit. It many cases it's more readable as `whereAnd` than just `where`.
Being explicit effectively prevents errors as a constant reminder of "what's going on here".

#### Q: should we accept both values in the same request?

**A:** no, because it's not obvious how to combine them. And choosing non-obvious fallback
just for the case of _having a fallback_ was proven to be a design error multiple times like in JS and more.
Sending both should result in a `400` error.

### 2. Where Structure

```
{
  whereAnd: [...]  
}
same for
{
  whereOr: [...]
}
```

#### Q: why not object here?

Like

```
{
  whereAnd: { 
    term1: value1,
    term2: value2,
  }  
}
```

**A:** because whatever `term1` is (field name, command name) it should support repetitions

```
{
  whereAnd: { 
    fieldname: value1,
    __fieldname__: value2, ! key duplication !
  }  
}

{
  whereOr: { 
    commandX: value1,
    __commandX__: value2, ! key duplication !
  }  
}
```

An array naturally allows duplication:

```
{
  whereAnd: [ 
    {fieldname: value1},
    {fieldname: value2} ! fine to repeat !
  ]
}

{
  whereOr: [
    {commandX: value1},
    {commandX: value2}, ! fine to repeat !
  ]
}
```
 
### 3. Where Array Structure

Let's focus on `whereAnd` cause `whereOr` will be the same.

There are two "obvious" ways to structure commands:

#### 1. Field-First

```
{
  whereAnd: [ 
    {fieldname: {op1: value1}},
    {fieldname: {op2: value2}},
  ]
  
  _or_
  
  whereAnd: [ 
    {fieldname: [op1, value1]},
    {fieldname: [op2, value2]},
  ]
}
```

#### 2. Command-First

```
{
  whereAnd: [ 
    {op1: {fieldname: value1}},
    {op2: {fieldname: value2}},
  ]
  
  _or_
  
  whereAnd: [ 
    {op1: [fieldname, value1]},
    {op2: [fieldname, value2]},
  ]
}
```

Command first is a superior alternative.

#### Q. how command-first is better that field-first?

**A:** field-first is arguably more _naturally_ readable in some cases

```
whereAnd: [ 
  {foo: {eq: bar} -- FOO eq BAR
]

_vs_

whereAnd: [ 
  {eq: [foo, bar] -- eq(FOO, BAR)
]
```

Command-first is LISP-like and requires some time to get used to. But it's objectively better because:

1. **Consistency**

Command-first is consistent with nested `and`, `or`, `not` (and top level `whereAnd`, `whereOr` which follow command-first approach!).

```
whereAnd: [
  {foo: {eq: bar}}, 
  not: [               -- not/and/or use a different order
    {foo: {eq: bar2}}, -- than eq and it's unavoidable
  ] 
]

_vs_

whereAnd: [
  {eq: [foo, bar]},
  not: [               -- all commands follow 
    {eq: [foo, bar2]}, -- the same structure
  ] 
]
```

2. **Arity**

Command-first has no problems arities. Like `foo OP bar` is a binary-only syntax. For unary commands it should be `OP foo` and for ternary commands it's `foo OP1_1 bar OP1_2 bazz`. Thy syntax for ternaries is so ineffective we see very few such commands in reality (both math and programming). Both JS and Python support only one kind
of ternary – a conditional operator (they became synonymous though they naturally aren't).

The same translates to JSON format. There seems to be no seamless way to express unary, ternary and commands with more arity with the field-first approach:

```
whereAnd: [
  {foo: {eq: V},                      -- binary, ok
  {foo: exists}                       -- exists is a command? or a value?
  {foo: {cond: {then: V1, else: V2}}} -- @_@ a mess
]
```

Unlike the command-first approach where everything is fine:

```
whereAnd: [
  {unary: [foo]},
  {binary: [foo, bar]},
  {ternary: [foo, bar, baz]},
  {n_ary: [foo, bar, baz, ...]},
]
```
