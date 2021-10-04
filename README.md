# Query Format

**WIP** This document describes a JSON format/schema convention to implement and query REST APIs.
It can also be used in GraphQL APIs, but, for the sake of brevity, this part is omitted for now.

The document provides a high-level overview of the format and is agnostic to exact command names like `eq vs equals vs =`
or other implementation details like CamelCase vs snake_case. The content is applicable regardless of your term/case/... preferences.
Such topics are partially touched below and might be elaborated later.

**Main benefits of having a flexible fixed schema:**
  - no need to (re)invent names and formats for each endpoint
  - potential for code generation / code autoresolving (based exclusively on endpoint schemas)

## Quick Example

#### JS code

```js
// or ["SEARCH", "/api/your-collection"] if this new verb is supported in your environment
fetchAPI(["POST", "/api/your-collection?search"], {
  whereAnd: [
    {eq: [field("location"), "UK"]}
    {notEq: [field("location"), "London, UK"]},
    {between: [field("salary"), [3000, 5000]]}
  ],
})
```

#### HTTP request

```json
// -> POST /api/your-collection?search
{
  "whereAnd": [
    {"eq": ["☆location", "UK"]}
    {"notEq": ["☆location", "London, UK"]},
    {"between": ["☆salary", [3000, 5000]]}
  ]
}

// <- 200 OK
...
```

Where `☆` marks the table/collection field and is used exclusively for the purpose of documentation.
It's not the real characted suggested in the format. More on that below.

## Types

Using TypeScript here to convey the idea.

```ts
const where : Where = [
  {eq: [field(location), "UK"]}     // : WhereItem
  {notEq: [field(location), "UK"]} // : WhereItem
]
```

```ts
type Request = {...} & ( // pagination, sorting, etc. aspects are skipped but belong here
  | {whereAnd : Where} 
  | {whereOr : Where}  
)  

export type Where = Condition[]

export type Condition =
  | EqCondition
  | NotEqCondition
  | GtCondition
  | LtCondition
  | GteCondition
  | LteCondition
  | RangeCondition
  | SearchCondition
  | NotCondition
  | OrCondition
  | AndCondition

type Comparable = number | string | boolean
type Sortable = number | string | boolean

type Field = string // TODO research "nominal type" approximations in TS

type EqCondition = {
  eq : [Field, Comparable] | [Field, Field]
}

type NotEqCondition = {
  notEq : [Field, Comparable] | [Field, Field]
}

type GtCondition = {
  gt : [Field, Sortable] | [Field, Field]
}

type LtCondition = {
  lt : [Field, Sortable] | [Field, Field]
}

type GteCondition = {
  gte : [Field, Sortable] | [Field, Field]
}

type LteCondition = {
  lte : [Field, Sortable] | [Field, Field]
}

type RangeCondition = {
  range : [Field, [number, number]] // simplified for now
}

type SearchCondition = {
  search : [string]
}

type NotCondition = {
  not : Condition[]
}

type OrCondition = {
  or : Condition[]
}

type AndCondition = {
  and : Condition[]
}
```

On BE you'll setup more precise types on per-endpoint basis:

```ts
// ENDPOINT SPECIFIC

type EqCondition = {
  eq : 
    | ["☆location", Comparable] 
    | ["☆location", "☆location"] 
    | ["☆profile_status", Comparable]}
    // ...all supported arguments for 'eq' condition
}

type NotEqCondition = {
  not_eq : 
    | ["☆location", Comparable] 
    | ["☆location", "☆location"] 
    | ["☆profile_status", Comparable]}
    // ...all supported arguments for 'not_eq' condition
}

// ...all supported commands (unique per endpoint)
```

^ The above will be declared using syntax of your API/validation BE framework/library or with imperative commands. 
The concepts of "tuples" and "literal vaues" must be supported.

## Guide

Like the name suggest, this page describes and explains the structure of `whereAnd / whereOr` query objects.
Implementation of `fetchAPI` and other helpers are out of the current scope.

### 1. Request: Top Level

```ts
{
  whereAnd: _,
  // x(or)
  whereOr: _,
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

```ts
{
  whereAnd: [_] 
}
// same for
{
  whereOr: [_]
}
```

#### Q: why not object here?

Like

```ts
{
  whereAnd: { 
    term1: value1,
    term2: value2,
  }  
}
```

**A:** because whatever `term1` is (field name, command name) it should support repetitions

```ts
{
  whereAnd: { 
    fieldname: value1,
    __fieldname__: value2, // key duplication!
  }  
}

{
  whereOr: { 
    commandX: value1,
    __commandX__: value2, // key duplication!
  }  
}
```

An array naturally allows duplication:

```ts
{
  whereAnd: [ 
    {fieldname: value1},
    {fieldname: value2} // ok to repeat!
  ]
}

{
  whereOr: [
    {commandX: value1},
    {commandX: value2}, // ok to repeat!
  ]
}
```
 
### 3. Where Array Structure

Let's focus on `whereAnd` cause `whereOr` will be the same.

There are two "obvious" ways to structure commands:

#### 1. Field-First

```ts
{
  whereAnd: [ 
    {fieldname: {op1: value1}},
    {fieldname: {op2: value2}},
  ]
  
  // or
  
  whereAnd: [ 
    {fieldname: [op1, value1]},
    {fieldname: [op2, value2]},
  ]
}
```

#### 2. Command-First

```ts
{
  whereAnd: [ 
    {op1: {fieldname: value1}},
    {op2: {fieldname: value2}},
  ]
  
  // or
  
  whereAnd: [ 
    {op1: [fieldname, value1]},
    {op2: [fieldname, value2]},
  ]
}
```

Command first is a superior alternative.

#### Q. how command-first is better that field-first?

**A:** field-first is arguably more _naturally_ readable in some cases

```ts
whereAnd: [ 
  {foo: {eq: bar} // FOO eq BAR
]

// vs

whereAnd: [ 
  {eq: [foo, bar] // eq(FOO, BAR)
]
```

Command-first is LISP-like and requires some time to get used to. But it's objectively better because:

1. **Consistency**

Command-first is consistent with nested `and`, `or`, `not` (and top level `whereAnd`, `whereOr` which follow command-first approach!).

```ts
whereAnd: [
  {foo: {eq: bar}}, 
  {not: [              // not/and/or use a different order
    {foo: {eq: bar2}}, // than eq and it's unavoidable
  ]} 
]

// vs

whereAnd: [
  {eq: [foo, bar]},
  {not: [              // all commands follow 
    {eq: [foo, bar2]}, // the same structure
  ]}
]
```

2. **Arity**

Command-first has no problems arities. Like `foo OP bar` is a binary-only syntax. For unary commands it should be `OP foo` and for ternary commands it's `foo OP1_1 bar OP1_2 bazz`. Thy syntax for ternaries is so ineffective we see very few such commands in reality (both math and programming). Both JS and Python support only one kind
of ternary – a conditional operator (they became synonymous though they naturally aren't).

The same translates to JSON format. There seems to be no seamless way to express unary, ternary and commands with more arity with the field-first approach:

```ts
whereAnd: [
  {foo: {eq: V},                       // binary, ok
  {foo: exists},                       // exists is a command? or a value?
  {foo: {cond: {then: V1, else: V2}}}, // @_@ a mess
]
```

Unlike the command-first approach where everything is fine:

```ts
whereAnd: [
  {unary: [foo]},
  {binary: [foo, bar]},
  {ternary: [foo, bar, baz]},
  {nAry: [foo, bar, baz, ...]},
]
```

3. **Rich Semantics**

For now we imagined `field OP value` cases exclusively. But there are `field1 OP field2` and even more complex cases involving multiple fields and values.
The whole "readability" argument of field-first approach fails shortly for that as we have to mark **values**

```ts
// SQL
// WHERE location = "UK"
// AND   location != "London, UK"
// AND   location != location2

// QUERY
whereAnd: [
  {location: {eq: "UK"}},
  {location: {notEq: "London, UK"}},
  {location: {eq: "...?..."}},       
]
```

We can try `{location: {eq: "@location2"}}` but it's inconsistent with the first `location` having no extra characters.

With command-first approach, if we don't want to rely on the order of fields/value (and we don't)
there are several workaround like using sentinel values:

```ts
// PG-SQL
// WHERE "location" = 'UK'          -- field1 OP value1 
// AND   "location" != 'London, UK' -- field1 OP value2
// AND   "location" != "location2"  -- field1 OP field2

// QUERY
whereAnd: [
  {eq: ["@location", "UK"]},
  {notEq: ["@location", "London, UK"]}, // or {not: [ {eq: ["@location", "London, UK"]} ]}
  {notEq: ["@location", "@location2"]},       
]
```

In PG we differ values and fields by the quote type which doesn't translate well to JSON.

In Mongo we mark commands with `$` so one more possiblity is like:

```ts
// QUERY
whereAnd: [
  {$eq: [{$field: "location"}, "UK"]},             // "location" = 'UK'
  {$notEq: [{$field: "location"}, "London, UK"]}, // "location" != 'London, UK'
  {$notEq: [{$field: "location"}, "location2"]},  // "location" != "location2"      
]
```

Generally speaking, this version relies on the fact that you control object keys much better than string values, at least on average.
Most objects in the code come from the code itself unlike string values. Some objects originate from DB but you'll
rarely have to deal with end-user constructed datastructures. It's possible to ensure by convention that your data keys 
don't start with sentinels. You can't expect the same guarantees from string values which come from everywhere and can take whatever shapes.

Regardless, it's significantly more noisy than the previous approach, at least in my opinion. More control characters, more nesting.

### 4. How to Escape: fields vs values

String values may contain almost any sequence of bytes (especially if consider binary data)
and field names are (or can be) naturally limited to alphanumerics and underscores by convention.

Given that, it's much easier to escape and unescape values than fields. In the first case we just prepend or append 
any character _that the field can not start_ with to our string values. In the second case we can't do the same for fields
as a valid value _could_ start with that character (we need to apply some string transformation to BOTH to make them recognizable).

Considering that there're much more potential values than fields in code (think of objects being values with each string value
being necessary to escape) it still seems more straightforward to escape fields instead of values. 

```ts
{
  whereAnd: [
    {eq: [field("location"), "UK")]},
    {notEq: [field("location"), field("location2")]},
  ]
}
```

Where `field` is the escaping function. But is it possible to select a sentinel value which wouldn't collide with strings.
Naive versions like `@` or `$` will fail shortly:

```ts
{
  whereAnd: [
    {eq: ["@location", "UK")]},
    {notEq: ["@location"), "@location2"]},
    // then "suddenly"
    {eq: ["@twitter", "@ivankleshnin")]}, // @_@ fiasco: the second one was meant to be a value, not a field
  ]
}
```

Check the [Specials_(Unicode_block)](https://en.wikipedia.org/wiki/Specials_(Unicode_block)) page on **Wikipedia** first.

**Unicode.org** has a detailed review of values you may try as [sentinels](http://www.unicode.org/faq/private_use.html#sentinels).
None of which is perfect but `\uFFFF` seems good enough unless some legacy clients will use your API.
It's officially suggested to be used as sentinel in Unicode 4.0. It's a [non-character](http://www.unicode.org/faq/private_use.html)
and should not appear in blobs(!), let alone human texts.

So `field` can be implemented like:

```ts
function field(str : string) : string {
  return "\uFFFF" + str
}
```

Now on Backend you just `fieldOrValue.startsWith("\uFFFF") ? _fieldName_ : _stringValue_`.

TODO verify in:

- JS (utf16) <-> JSON (utf8) <-> Python (utf8)
- JS (utf16) <-> JSON (utf8) <-> JS (utf16)
- and possibly other cross-platform, cross-UTF exchanges...

### 5. snake_case vs camelCase

Some APIs expect and return object data in snake_case instead of camelCase following the conventions of
their languages (Python, Rust, etc). One option is to use snake_case directly in your code but it's prone
to expand beyond the fetching layer and "contaminate" forms, etc. with unconventional names.

It's trivial to add a layer of convertion between the data
and the fetching code:

```ts
fetchAPI(["POST", "/api/your-collection?search"], {
  whereAnd: formatWhere([
    {notEq: [field("location"), "London, UK"]},
  ])
})
```

to create a JSON in expected format:

```json
{
  "whereAnd": [
    {"notEq": ["☆location", "London, UK"]}
  ]
}
```

In the above example `formatWhere` can be implemented like (TypeScript):

```ts
const formatWhere = (where : Where = []) : Where => {
  return snakifyKeys(where) as Where
} // check Where definition above
```

and `snakifyKeys` is:

```ts
import {pipe, fromCamelCase, fromSnakeCase, toCamelCase, toSnakeCase, convertData} from "kecasn"

const snakifyStr = pipe(fromCamelCase, toSnakeCase)
const camelizeStr = pipe(fromSnakeCase, toCamelCase)

const snakifyKeys = convertData(snakifyStr, {keys: true})
```

Using [kecasn](https://github.com/ivan-kleshnin/kecasn) case conversion library as one of many tools that do the job.
