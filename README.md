# query_format

This document describes the JSON format that can be used to query REST APIs.
The document is stuctured as a guide + Q&A.

## 1. Request: Top Level

```
{
  whereAnd: ...
  xor
  whereOr: ...
}
```

#### Q: why not default `where` to `whereAnd` semantics?

**A:** explicit is better than implicit. It many cases it's more readable as `whereAnd` than just `where`.
Being explicit effectively prevents errors as a constant reminder of "what's going on here".

#### Q: can we provide both at the same time?

**A:** no, because it's not obvious how to combine them. And choosing non-obvious fallback
just for the case of _having a fallback_ was proven to be a design error multiple times like in JS and more.

## 2. Where Structure

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
