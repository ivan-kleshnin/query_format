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

3. **Rich Semantics**

For now we imagined `field OP value` cases exclusively. But there are `field1 OP field2` and even more complex cases involving multiple fields and values.
The whole "readability" argument of field-first approach fails shortly for that as we have to mark **values**

```
SQL
  WHERE location = "UK"
  AND   location != "London, UK"
  AND   location != location2

QUERY
  whereAnd: [
    {location: {eq: "UK"}},
    {location: {not_eq: "London, UK"}},
    {location: {eq: "...?..."}},       
  ]
```

We can try `{location: {eq: "@location2"}}` but it's inconsistent with the first `location` having no extra characters.

With command-first approach, if we don't want to rely on the order of fields/value (and we don't)
there are several workaround from which marking fields with extra chars like `@` seems the easiest:

```
PG-SQL
  WHERE "location" = 'UK'          -- field1 OP value1 
  AND   "location" != 'London, UK' -- field1 OP value2
  AND   "location" != "location2"  -- field1 OP field2

QUERY
  whereAnd: [
    {eq: ["@location", "UK"]},
    {not_eq: ["@location", "London, UK"]}, -- or {not: [ {eq: ["@location", "London, UK"]} ]}
    {not_eq: ["@location", "@location2"]},       
  ]
```

In which case string values have to be escaped like `{eq: ["@twitter", "\@ivan_kleshnin"]}` which seems not to be difficult
both to encode (on FE) and decode (on BE). In PG we differ values and fields by the quote type which doesn't translate well to JSON.

In Mongo we mark commands with `$` so one more possiblity is like:

```
QUERY
  whereAnd: [
    {$eq: [{$field: "location"}, "UK"]},             -- "location" = 'UK'
    {$not_eq: [{$field: "location"}, "London, UK"]}, -- "location" != 'London, UK'
    {$not_eq: [{$field: "location"}, "location2"]},  -- "location" != "location2"      
  ]
```

But it's significantly more noisy than the previous, at least in my opinion. More control characters, more nesting.
