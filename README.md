# Proposal for ECMAScript enums

A common and oft-used feature of many languages is the concept of an 
[Enumerated Type](https://en.wikipedia.org/wiki/Enumerated_type), or `enum`. Enums provide a finite
domain of constant values that are regularly used to indicate choices, discriminants, and bitwise
flags.

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


# Prior Art

- C++: [Enumerations](https://docs.microsoft.com/en-us/cpp/cpp/enumerations-cpp)  
- C#: [Enumeration types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/enum)  
- Java: [Enum types](https://docs.oracle.com/javase/specs/jls/se10/html/jls-8.html#jls-8.9)  
- TypeScript: [Enums](www.typescriptlang.org/docs/handbook/enums.html)  


# Syntax

```ts
// enum declarations

// Each auto-initialized member value is a `Number`, auto-increments values by 1 starting at 0
enum Numbers {
  zero,
  one,
  two,
  three,
  alsoThree = three
}

// Each auto-initialized member value is a `Number`, auto-increments values by 1 starting at 0
enum Colors of Number {
  red,
  green,
  blue
}

// Each auto-initialized member value is a `String` whose value is the SV of its member name.
enum PlayState of String {
  idle,
  running,
  paused
}

// Each auto-initialized member value is a `Symbol` whose description is the SV of its member name.
enum Symbols of Symbol {
  alpha,
  beta
}

enum Named {
  identifierName,
  "string name",
  [expr]
}

// Accessing enum values:
let x = Color.red;
let y = Named["string name"];
```

# Semantics

## Well-Known Symbols

This proposal introduces three new well-known symbols that are used with enums:

| Specification Name | \[\[Description]] | Value and Purpose |
|:-|:-|:-|
| @@toEnum | `"Symbol.toEnum"` | A method that is used to derive the value for an enum member during _EnumMember_ evaluation. |
| @@formatEnum | `"Symbol.formatEnum"` | A method of an _enum object_ that is used to convert a value into a string representation based on the member names of the enum. Called by `Enum.format`. |
| @@parseEnum | `"Symbol.parseEnum"` | A method of an _enum object_ that is used to convert a member name String into the value represented by that member of the enum. Called by `Enum.parse`. |

## Properties of the Number Constructor

The Number constructor would have an additional @@toEnum method with parameters `key`, `value`, 
and `autoValue` that performs the following steps:

1. If Type(`value`) is not Number, set `value` to `autoValue`.
1. If `value` is `undefined`, return `0`.
1. Return `value + 1`.

## Properties of the String Constructor

The String constructor would have an additional @@toEnum method with parameters `key`, `value`,
and `autoValue` that returns a string derived from `key`.

## Properties of the Symbol Constructor

The Symbol constructor would have an additional @@toEnum method that parameters `key`, `value`,
and `autoValue` that returns a symbol derived from `key`.

## Properties of the BigInt Constructor

The BigInt constructor would have an additional @@toEnum method with parameters `key`, `value`,
and `autoValue` that performs the following steps:

1. If Type(`value`) is not BigInt, set `value` to `autoValue`.
1. If `value` is `undefined`, return `0n`.
1. Return `value + 1n`.

## Enum Declarations

Enum declarations consist of a finite set of _enum members_ that define the names and values
for each member of the enum. These results are stored as properties of an _enum object_. An 
_enum object_ is an ordinary object with an \[\[EnumMembers]] internal slot, and whose 
\[\[Prototype]] is `null`.

### Automatic Initialization

If an _enum member_ does not supply an _Initializer_, the value of that _enum member_ will be 
automatically initialized:

```js
enum DaysOfTheWeek {
  Sunday, // 0
  Monday, // 1
  Tuesday, // 2
  // etc.
}
```

Auto-initialization can be controlled through the use of an `of` clause:

```js
enum DaysOfTheWeek of Symbol {
  Sunday, // Symbol("Sunday")
  Monday, // Symbol("Monday")
  Tuesday, // Symbol("Tuesday")
  // etc.
}
```

Constructors for built-in primitive values like `String`, `Number`, `Symbol`, and `BigInt` are 
defined to have a `@@toEnum` method that is used during evaluation to select an 
auto-initialization value. If the expression in the `of` clause does not have a `@@toEnum` method,
it will instead be called directly. This allows constructors for built-ins to be used in the `of`
clause without adding a niche constructor overload. This also allows developers to control the 
behavior of `of` if its expression is an ECMAScript `class` which cannot be called directly.

### Evaluation

Before we evaluate the _enum members_ of the declaration, we first choose a `mapper` Object. 
If the enum declaration has an `of` clause, the `mapper` is the result of evaluating that clause.
Otherwise, `mapper` uses the default value of %Number%.

From the `mapper` we then get an `enumMap` function from `mapper[@@toEnum]`. If `enumMap` is
`undefined`, then we set `enumMap` to `mapper` and `mapper` to `undefined`.

To support auto-initialization we also define to variables (both initialized to `undefined`): 
- `value`: Stores the result of the last explicit or automatic initialization.
- `autoValue`: Stores the result of the last automatic initialization only.

As we evaluate each _enum member_, we perform the following steps:

1. Derive `key` from the _enum member_'s name.
1. If the _enum member_ has an _Initializer_, 
    1. Set `value` to be the result of evaluating _Initializer_.
1. Else,
    1. Set `autoValue` to be ? Call(`enumMap`, `mapper`, &laquos; `key`, `value`, `autoValue`)
    1. Set `value` to be `autoValue`
1. Add `key` to the List of member names in the \[\[EnumMembers]] internal slot of the 
  _enum object_.
1. Define a new property on the _enum object_ with the name `key` and the value `value`,
  and the attributes `[[Writable]]`: **false**, `[[Configurable]]`: **false**, and
  `[[Enumerable]]`: **true**.

In addition, the following additional properties are added to _enum objects_:

  - A `@@parseEnum` property whose value is a Function that returns the value of the _enum member_ 
    whose name corresponds to the provided argument. 
    - This member is \[\[Writable]]: `false`, \[\[Configurable]]: `true`, and 
      \[\[Enumerable]]: `false`.
  - A `@@formatEnum` property whose value is a Function that returns the name of the first 
    _enum member_ whose value corresponds to the provided argument.
    - This member is \[\[Writable]]: `false`, \[\[Configurable]]: `true`, and 
      \[\[Enumerable]]: `false`.
  - A `@@toStringTag` property whose value is `"Enum"`.
    - This member is \[\[Writable]]: `false`, \[\[Configurable]]: `true`, and 
      \[\[Enumerable]]: `false`.
  - An `@@iterator` property whose value is a Function that returns an iterator for this enum's 
    \[\[EnumMembers]] internal slot where each yielded value is a two-element array containing the 
    enum member name at index 0 and the enum member value at index 1.
    - This member is \[\[Writable]]: `false`, \[\[Configurable]]: `true`, and 
      \[\[Enumerable]]: `false`.
  
Finally, the _enum object_ is made non-extensible.

# API

To make it easier to work with enums, an `Enum` object is added to the global scope, with the following
methods:

- `Enum.keys(E)` - Returns an `Iterator` for the member names in the \[\[EnumMembers]] internal 
  slot of `E`.
- `Enum.values(E)` - Returns an `Iterator` for the value on `E` of each member in the 
  \[\[EnumMembers]] internal slot of `E`.
- `Enum.entries(E)` - Returns an `Iterator` for each member in the \[\[EnumMembers]] internal
  slot of `E`, where each result is two-element array containing the enum member name at index 0 
  and the enum member value at index 1.
- `Enum.has(E, key)` - Returns `true` if the the \[\[EnumMembers]] internal slot of `E` contains `key`.
- `Enum.hasValue(E, value)` - Returns `true` if the \[\[EnumMembers]] internal slot of `E` contains a
  member whose value on `E` corresponds to `value`.
- `Enum.getName(E, value)` - Gets the first name in the \[\[EnumMembers]] internal slot of `E` whose
  value on `E` corresponds to `value`.
- `Enum.format(E, value)` - Calls the `@@formatEnum` method of `E` with argument `value`.
- `Enum.parse(E, value)` - Calls the `@@parseEnum` method of `E` with argument `value`.
- `Enum.create(members)` - Creates an _enum object_ using the property keys and values of `members`
  as the _enum members_ for the new enum.
- `Enum.flags(descriptor)` - A built-in decorator that modifies the _enum object_ in the following ways:
  - The auto-increment behavior is changed to shift the current auto-increment value left by 1. 
  - The `@@parseEnum` method is modified to parse a comma-separated string and OR the resulting values
    together. If no corresponding name can be found and the name can be successfully coerced to a number,
    that number is OR'ed with the result.
  - The `@@formatEnum` method is modified to convert a bitwise combination of flag values into a comma
    separated string of corresponding names. If no corresponding name can be found, the SV of the 
    bits is appended to the string.

```ts
let Enum: {
  keys(E: object): IterableIterator<string | symbol>;
  values(E: object): IterableIterator<any>;
  entries(E: object): IterableIterator<[string | symbol, any]>;
  has(E: object, key: string | symbol): boolean;
  hasValue(E: object, value: any): boolean;
  getName(E: object, value: any): string | undefined;
  format(E: object, value: any): string | symbol | undefined;
  parse(E: object, value: string): any;
  create(members: object): object;
  flags(descriptor: EnumDescriptor): EnumDescriptor;
};
```

<!--
# Grammar

```grammarkdown
```
-->

# Examples

<!-- Examples of the proposal -->

```js
enum Numbers { zero, one, two, three, }

typeof Numbers.zero === "number"
Numbers.zero === 0
Enum.getName(Numbers, 0) === "zero"
Enum.parse(Numbers, "zero") === 0

// ... strings, ...
enum HttpMethods of String { GET, PUT, POST, DELETE }

typeof HttpMethods.GET === "string"
HttpMethods.GET === "GET"

// ... booleans, ...
enum Switch { on = true, off = false }

typeof Switch.on === "boolean";
Switch.on === true

// ... symbols, ...
enum AlphaBeta of Symbol { alpha, beta }

typeof AlphaBeta.alpha === "symbol";
AlphaBeta.alpha.toString() === "Symbol(AlphaBeta.alpha)";

// ... or a mix.
enum Mixed {
    number = 0,
    string = "",
    boolean = false,
    symbol = Symbol()
}

// Enums can be exported:
export enum Zoo { lion, tiger, bear };
export default enum { up, down, left, right };

// You can test for name membership using `Enum.has()`
Enum.has(Numbers, "one") === true
Enum.has(Numbers, "five") === false

// You can test for value membership using `Enum.hasValue()`:
Enum.hasValue(Numbers, 0) === true
Enum.hasValue(Numbers, 9) === false

// You can convert enums between names and values using 
// `Enum.parse` and `Enum.format`, respectively.
enum AToB {
    a = "b",
    b = "a",
}

Enum.parse(AToB, "a") === AToB.a
Enum.parse(AToB, "b") === AToB.b

Enum.getName(AToB, AToB.a) === "b"
Enum.getName(AToB, AToB, b) === "a"

// `Enum.create()` lets you create a new enum programmatically:
const SyntaxKind = Enum.create({ 
  identifier: 0, 
  number: 1, 
  string: 2 
});

typeof SyntaxKind.identifier === "number";
SyntaxKind.identifier === 0;


// The `Enum.flags` decorator lets you declare a enum containing 
// bitwise flag values:
@Enum.flags
enum FileMode {
  none
  read,
  write,
  exclusive,
  readWrite = read | write,
}

FileMode.none === 0x0
FileMode.readOnly === 0x1
FileMode.readWrite === 0x3

// `Enum.flags` modifies @@formatEnum:
Enum.format(FileMode, FileMode.readWrite | FileMode.exclusive) === "readWrite, exclusive"

// `EnumFlags` modifies @@parseEnum:
Enum.parse(FileMode, "read, 4") === 5 // FileMode.read | FileMode.exclusive
```

# Remarks

- Why default to Number?
  - In prior discussions, there are some preferences for the use of 
    [symbol values](https://esdiscuss.org/topic/propose-simpler-string-constant#content-8), 
    while there are other preferences that include the use of 
    [strings and numbers](https://esdiscuss.org/topic/propose-simpler-string-constant#content-14). 
    This approach gives you the ability to support both scenarios through the optional `of` clause.
  - The auto-increment behavior of enums in other languages is used fairly regularly. Auto-
    increment is not viable if String or Symbol were the default type. 
  - We could consider switching on auto-increment if the prior declaration was initialized with a
    Number, but then you would have confusion over declarations like this:

    ```ts
    enum Mixed {
      first, // If this is a Symbol by default...
      second = 1,
      third // ...is this a Symbol or the Number `2`?
    }
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