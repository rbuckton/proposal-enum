# Proposal for ECMAScript enums

A common and oft-used feature of many languages is the concept of an [Enumerated Type][], or `enum`. Enums provide a
finite domain of constant values that are regularly used to indicate choices, discriminants, and bitwise flags. As a
popular and heavily used feature of TypeScript, this proposal seeks the adoption of a compatible form of TypeScript's
`enum` declaration. Where the syntax or semantics of this proposal differ from that of TypeScript, it is with the full
knowledge of the TypeScript development team and represents either a change that TypeScript is willing to adopt, or
represents a limited subset of functionality that TypeScript expands upon.

> NOTE: This proposal has been heavily reworked from it's prior incarnation, which can now be found at
> https://github.com/rbuckton/proposal-enum/tree/old

## Status

**Stage:** 0  
**Champion:** Ron Buckton (@rbuckton)

_For more information see the [TC39 proposal process](https://tc39.github.io/process-document/)._

## Authors

* Ron Buckton (@rbuckton)

# Motivations

Many ECMAScript hosts and libraries have various ways of distinguishing types or operations via
some kind of discriminant:

- ECMAScript:
  - `[Symbol.toStringTag]`
  - `typeof`
- DOM:
  - `Node.prototype.nodeType` (`Node.ATTRIBUTE_NODE`, `Node.CDATA_SECTION_NODE`, etc.)
  - `DOMException.prototype.code` (`DOMException.ABORT_ERR`, `DOMException.DATA_CLONE_ERR`, etc.)
  - `XMLHttpRequest.prototype.readyState` (`XMLHttpRequest.DONE`, `XMLHttpRequest.HEADERS_RECEIVED`,
    etc.)
  - `CSSRule.prototype.type` (`CSSRule.CHARSET_RULE`, `CSSRule.FONT_FACE_RULE`, etc.)
  - `Animation.prototype.playState` (`"idle"`, `"running"`, `"paused"`, `"finished"`)
  - `ApplicationCache.prototype.status` (`ApplicationCache.CHECKING`,
    `ApplicationCache.DOWNLOADING`, etc.)
- NodeJS:
  - `Buffer` encodings (`"ascii"`, `"utf8"`, `"base64"`, etc.)
  - `os.platform()` (`"win32"`, `"linux"`, `"darwin"`, etc.)
  - `"constants"` module (`ENOENT`, `EEXIST`, etc.; `S_IFMT`, `S_IFREG`, etc.)

In addition, with the recent adoption of [TypeScript Type Stripping][] in NodeJS, there has been renewed interest in
ECMA-262 adopting a compatible form of TypeScript's `enum` declaration as it is one of the most frequently used
TypeScript features not supported in this mode.

An `enum` declaration provides several advantages over an object literal:
- Closed domain by default &mdash; The declaration is non-extensible and enum members would be non-configurable
  and non-writable.
- Restricted domain of allowed values &mdash; Restricts initializers to a subset of allowed JS values (`Number`, 
  `String`, `Symbol`, `BigInt`).
- Self-references during definition &mdash; Referring to prior enum members of the same enum in the initializer of a
  subsequent enum member.
- Static Typing (tooling) &mdash; Static type systems like TypeScript use enum declarations to discriminate types,
  provide documentation in hovers, etc.
- ADT (future) &mdash; Potential future support for Algebraic Data Types.
- Decorators (future) &mdash; Potential future support for `enum`-specific Decorators.
- Auto-Initializers (future) &mdash; Potential future support for auto-initialized enum members.
- "shared" enums (future) &mdash; Potential future support for a `shared enum` with restrictions on inputs to align with
  [`shared struct`](https://github.com/tc39/proposal-structs)


# Prior Art

- TypeScript: [Enums](https://www.typescriptlang.org/docs/handbook/enums.html)  
- C++: [Enumerations](https://docs.microsoft.com/en-us/cpp/cpp/enumerations-cpp)  
- C#: [Enumeration types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/enum)  
- Java: [Enum types](https://docs.oracle.com/javase/specs/jls/se10/html/jls-8.html#jls-8.9)  


# Syntax

While a Stage 1 proposal is generally encouraged to avoid specific syntax, it is a stated goal of this proposal to
introduce an `enum` declaration whose syntax is compatible with TypeScript's. As such, the syntax of this proposal is
restricted to a subset of TypeScript's `enum`.

```ts
// enum declarations

enum Numbers {
  zero = 0,
  one = 1,
  two = 2,
  three = 3,
  alsoThree = three // self reference
}

enum PlayState {
  idle = "idle",
  running = "running",
  paused = "paused"
}

enum Symbols {
  alpha = Symbol("alpha"),
  beta = Symbol("beta") 
}

enum Named {
  Identifier = 0,
  "string name" = 1,
}

// Accessing enum values:
let x = Numbers.three;
let y = Named["string name"];

// Reverse mapping (formatting, debugging, diagnostics, etc):
for (const [key, value] of Numbers) ...
```

# Semantics

While a Stage 1 proposal is generally encouraged to avoid specific semantics, it is a stated goal of this proposal to
introduce an `enum` declaration whose semantics are compatible with TypeScript's. As such, the semantics of this
proposal are intended to align with that of TypeScript's `enum` where possible.


## Enum Declarations

Enum declarations consist of a finite set of _enum members_ that define the names and values
for each member of the enum. These results are stored as properties of an _enum object_. An 
_enum object_ is an ordinary object whose \[\[Prototype]] is `null`. Each enum member defines
a property on the _enum object_.

In addition, an _enum object_ contains an `@@iterator` method that yields a `[key, value]` entry for each declard enum
member, in document order. To explain the semantics of the `@@iterator` method, an _enum object_ may require an
\[\[EnumMembers]] internal slot.


## Enum Members

Enum members define the set of values that belong to the enum's domain. Each enum member consists of a name and an
initializer that defines the value associated with that name. Enum members are \[\[Writable]]: `false` and
\[\[Configurable]]: `false`.


### Enum Member Names

Enum member names are currently restricted to _IdentifierName_ and _StringLiteral_, as those are the only member names
currently supported by TypeScript's `enum`. We may opt to expand this to allow other member names like
_ComputedPropertyName_ in the future, if there is sufficient motivation.

An enum may not have duplicate member names. We may opt to introduce restrictions on member names like `constructor` if
we deem it necessary to support ADT enums in the future.

 If an enum member name shares the same string value as the name of the enum itself, it shadows the enum declaration
 name within the enum body.

### Enum Member Initializers

An enum member's initializer is restricted to a subset of
ECMAScript values (i.e., `Number`, `String`, `Symbol`, and `BigInt`). This limitation allows us to consider future
support for Algebraic Data Types (ADT) in enums without the possibility of conflicting with something like an enum
member whose value is a function.

An _IdentifierReference_ in an enum member initializer may refer to the name of a prior declaration, and to the enum
itself (much like a `class`).

# API

Aside from the `enum` declaration itself, there is no other proposed API.

An `enum` declaration will have an `@@iterator` method that can be used to iterate over the key/value pairs of the enum's
members.

# Desugaring

An `enum` declaration could potentially be implemented as the following desugaring:

```js
enum E {
  A = 1,
  B = 2,
  C = A | E.B,
}

let E = (() => {
  let E = Object.create(null), A, B, C;
  Object.defineProperty(E, "A", { value: A = 1 });
  Object.defineProperty(E, "B", { value: B = 2 });
  Object.defineProperty(E, "C", { value: C = A | E.A });
  Object.defineProperty(E, Symbol.iterator, {
    value: function* () {
      yield ["A", E.A];
      yield ["B", E.B];
      yield ["C", E.C];
    }
  });
  Object.defineProperty(E, Symbol.toStringTag, { value: "E" });
  Object.preventExtensions(E);
  return E;
})
```

# Other Considerations

## `enum` Expressions

While ECMAScript has both statement and expression forms for `class` and `function` declarations, this proposal does not
currently support `enum` expressions. There is no concept of an `enum` expression in TypeScript, though we may consider
`enum` expressions for ECMAScript if there is sufficient motivation.

## `export`/`export default`

An `enum` declaration would support both `export` and `export default`, much like `class`.

## Interaction with Shared Structs

In general, this proposal hopes to align enum member values to those that can be shared with a `shared struct`, however
it is important to note that a `Symbol`-valued enum that uses `Symbol()` and not `Symbol.for()` would produce values
that cannot be reconciled between two Agents. In addition, ADT enums may need to contain non-shared data, such as in an
`Option` or `Result` enum. As such, this proposal may seek to introduce a `shared enum` declaration that further
restricts allowed inputs.

# Differences from TypeScript

There are several differences in the `enum` declaration for this proposal compared to TypeScript's `enum`:

- [Auto-initializers](#auto-initializers)
- [Declaration merging](#declaration-merging)
- [Reverse mapping](#reverse-mapping)
- [`const enum`](#const-enum)
- [`Symbol` values](#symbol-values)
- [`BigInt` values](#bigint-values)
- [`export default`](#export-default)

## Auto-Initializers

TypeScript's `enum` supports auto-initialization of enum members:

```ts
enum Numbers {
  zero, // 0
  one,  // 1
  two   // 2
}
```

However, this behavior is contentious amongst some TC39 delegates and has been removed from this proposal. TypeScript
will continue to support auto-initializiation due to its frequent use within the developer community, but would emit
explicit initializers to JavaScript. It is possible that another form of auto-initializiation may be introduced in the
future that could be utilized by both TypeScript and ECMAScript.

## Declaration Merging

TypeScript (as of v5.8) allows `enum` declarations to merge with other `enum` (and `namespace`) declarations with the
same name. This is not a desirable feature for ECMAScript enums and will not be supported. TypeScript is considering
[deprecating this feature](https://github.com/microsoft/TypeScript/issues/54500#issuecomment-2770170732) in
TypeScript 6.0.

## Reverse Mapping

TypeScript currently supports reverse-mapping enum values back to enum member names using `E[value]`, but only for
`Number` values. This limitation is intended to avoid collisions for `String`-valued enum members that could potentially
overwrite other members. While this information is invaluable for debugging, diagnostics, formatting, and serialization,
it is far less frequently used compared to `enum` on the whole.

To avoid this inconsistency, we propose instead propose introducing reverse mapping by way of the `@@iterator` built-in
symbol:

```js
enum E {
  A = 0,
  B = "A",
}

for (const [key, value] of E) {
  console.log(`${key}: ${value}`);
}

// prints:
//  A: 0
//  B: A
```

If adopted, TypeScript would add support for `@@iterator` while eventually deprecating existing reverse mapping support.


## `const enum`

TypeScript supports the concept of a `const enum` declaration, which is similar to a normal `enum` declaration except
that enum values are inlined into their use sites. As this capability requires whole program knowledge, it is not a
feature we intend to support in ECMAScript enums.


## `Symbol` values

TypeScript does not currently support `Symbol` values for enums, but would add support if this feature were to be
adopted.


## `BigInt` values

TypeScript does not currently support `BigInt` values for enums, but would add support if this feature were to be
adopted.


## `export default`

TypeScript does not currently support `export default` on an enum, but would add support if this feature were to be
adopted.


# Future Directions

While this proposal is intended to be rather limited in scope, there are several potential areas for future advancement
in the form of follow-on proposals:

- [Algebraic Data Types](#algebraic-data-types-adts)
- [Decorators](#decorators)
- [Auto-Initializers](#auto-initializers-1)

## Algebraic Data Types (ADTs)

Algebraic Data Types (ADT) are structured objects with some form of discriminant property. A future enhancement of an
ECMAScript `enum` declaration might support ADT enums in conjunction with [Extractors][] and [Pattern Matching][]:

```js
enum Option {
  Some(value),
  None()
}

const opt = Option.Some(123);
match (opt) {
  Option.Some(let value): console.log(value);
  Option.None(): console.log("<no value>");
}


enum Result {
  Ok(value),
  Error(reason)
}

function try_(cb) {
  try {
    return Result.Ok(_cb());
  } catch (e) {
    return Result.Error(e);
  }
}

const res = try_(() => obj.doWork());
match (res) {
  Result.Ok(let value): ...;
  Result.Error(let reason): ...;
}
```

## Decorators

In the future we may opt to extend support for [Decorators](https://github.com/tc39/proposal-decorators) to `enum`
declarations to support serialization/deserialization, formatting, and FFI scenarios:

```js
enum Role {
  @Alias(["user", "person"], { ignoreCase: true })
  user = 1,
  @Alias(["admin", "administrator"], { ignoreCase true })
  admin = 2,
}
```

## Auto-Initializers

While this proposal does not support TypeScript's auto-initialization semantics, we may consider introducing an
alternative syntax in a future proposal, such as the `of` clause described in an
[older version of this proposal](https://github.com/rbuckton/proposal-enum/tree/old):

```js
enum Numbers of Number { zero, one, two, three }
Numbers.zero; // 0
```

Or through some form of statically recognizable syntax:

```js
auto enum Numbers { zero, one, two, three }
```


# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.  
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.  
* [x] Illustrative [examples][Examples] of usage.  
* [x] High-level [API][API].  

### Stage 2 Entrance Criteria

* [ ] [Initial specification text][Specification].  
* [ ] [Transpiler support][Transpiler] (_Optional_).  

### Stage 3 Entrance Criteria

* [ ] [Complete specification text][Specification].  
* [ ] Designated reviewers have [signed off][Stage3ReviewerSignOff] on the current spec text.  
* [ ] The ECMAScript editor has [signed off][Stage3EditorSignOff] on the current spec text.  

### Stage 4 Entrance Criteria

* [ ] [Test262](https://github.com/tc39/test262) acceptance tests have been written for mainline usage scenarios and [merged][Test262PullRequest].  
* [ ] Two compatible implementations which pass the acceptance tests: [\[1\]][Implementation1], [\[2\]][Implementation2].  
* [ ] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.  
* [ ] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].  


<!-- # References -->
<!-- Links to other specifications, etc. -->
<!--
* [Title](url)
-->

# Prior Discussion
<!-- Links to prior discussion topics on https://esdiscuss.org -->

* [Enums?](https://esdiscuss.org/topic/enums)
* [Propose simpler string constant](https://esdiscuss.org/topic/propose-simpler-string-constant)
* [Old version of this proposal](https://github.com/rbuckton/proposal-enum/tree/old)
* [Prior `enum` proposal by Jack-Works](https://github.com/jack-works/proposal-enum)

<!-- The following are shared links used throughout the README: -->

[Champion]: #status
[Prose]: #motivations
[Examples]: #examples
[API]: #api
[Specification]: https://rbuckton.github.io/proposal-enum
[Transpiler]: #todo
[Stage3ReviewerSignOff]: #todo
[Stage3EditorSignOff]: #todo
[Test262PullRequest]: #todo
[Implementation1]: #todo
[Implementation2]: #todo
[Ecma262PullRequest]: #todo
[Enumerated Type]: https://en.wikipedia.org/wiki/Enumerated_type
[TypeScript Type Stripping]: https://nodejs.org/docs/latest/api/typescript.html#type-stripping
[Extractors]: https://github.com/tc39/proposal-extractors
[Pattern Matching]: https://github.com/tc39/proposal-pattern-matching