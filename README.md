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

// Implicitly extends `Number`, auto-increments values by 1 starting at 0
enum Numbers {
  zero,
  one,
  two,
  three,
  alsoThree = three
}

// Explicitly extends `Number`, auto-increments values by 1 starting at 0
enum Colors extends Number {
  red,
  green,
  blue
}

// Explicitly extends `String`, each value is the SV of its member name.
enum PlayState extends String {
  idle,
  running,
  paused
}

// Explicitly extends `Symbol`, each value is a `Symbol` whose description is the SV of its 
// member name.
enum Symbols extends Symbol {
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

The Number constructor would have an additional @@toEnum method with parameters `key` and `value`
that returns the value of `value`.

## Properties of the String Constructor

The String constructor would have an additional @@toEnum method with parameters `key` and `value` 
that returns a string derived from `key`.

## Properties of the Symbol Constructor

The Symbol constructor would have an additional @@toEnum method that parameters `key` and `value`
that returns a symbol whose description is derived from `key`.

## Properties of the BigInt Constructor

The BigInt constructor would have an additional @@toEnum method with parameters `key` and `value`
that returns the value of `value` as a BigInt.

## Enum Declarations

When an `enum` is declared, its members are evaluated and a new _enum object_ is
created:

  - An _EnumDeclaration_ may have an `extends` clause:
    - The result of evaluating the `extends` clause (<var>memberType</var>) must be an Object with 
      a @@toEnum method.
    - The value of this method (<var>enumMap</var>) will be used to determine the initialized 
      value for each enum member.
    - If <var>enumMap</var> is not callable, a **TypeError** is thrown.
  - If an _EnumDeclaration_ does not have an `extends` clause:
    - <var>memberType</var> would be `undefined`
    - <var>enumMap</var> would be a built-in function that returns the value provided to its second argument. 
  - The _enum members_ of the declaration are evaluated with the provided <var>memberType</var> and <var>enumMap</var>.
  - An _EnumDeclaration_ may be decorated, and the decorators may add, remove, or modify _enum members_.
  - An _enum object_ is an Object with a \[\[Prototype]] of `null`.
  - An _enum object_ has an \[\[EnumMembers]] internal slot, which is a List of the names of its
    [Enum Members](#enum-members).
  - An _enum object_ has an @@parseEnum property whose value is a Function that returns the value
    of the _enum member_ whose name corresponds to the provided argument. 
    - This member is \[\[Writable]]: `false`, \[\[Configurable]]: `true`, and 
      \[\[Enumerable]]: `false`.
  - An _enum object_ has an @@formatEnum property whose value is a Function that returns the name
    of the first _enum member_ whose value corresponds to the provided argument.
    - This member is \[\[Writable]]: `false`, \[\[Configurable]]: `true`, and 
      \[\[Enumerable]]: `false`.
  - An _enum object_ has an @@toStringTag property whose value is `"Enum"`.
    - This member is \[\[Writable]]: `false`, \[\[Configurable]]: `true`, and 
      \[\[Enumerable]]: `false`.
  - An _enum object_ has an @@iterator property whose value is a Function that returns an iterator
    for this enum's \[\[EnumMembers]] internal slot where each yielded value is a two-element 
    array containing the enum member name at index 0 and the enum member value at index 1.
    - This member is \[\[Writable]]: `false`, \[\[Configurable]]: `true`, and 
      \[\[Enumerable]]: `false`.
  - An _enum object_ has a property for each _enum member_ with that _enum member_'s name,
    whose value is that _enum member_'s value. 
    - These properties are \[\[Writable]]: `false`, \[\[Configurable]]: `false`, and 
      \[\[Enumerable]]: `true`.
  - An _enum object_'s \[\[Extensible]] internal slot is `false`.

## Enum Members

Enum members consist of a comma-separated list of enum member names with optional initializers:

- _enum members_ are evaluated with a supplied <var>memberType</hint> object and <var>enumMap</var>
  function.
- Enum names can be identifiers, string literals, or computed property names. When evaluated
  each name is coerced via ToPropertyKey.
- It is a runtime error for if there are two enum members with the same name.
  - A runtime error is necessary as computed property names must be evaluated.
- Enum members may be decorated, and the decorator may modify or add new _enum members_, or 
  replace the default initializer.
- When evaluating _enum members_, <var>autoValue</var> is initialized to 0. This variable is used
  to manage auto-increment behavior.  
- If an _enum member_ has an initializer:
  - The result of the initializer expression will be coerced via ToPrimitive (<var>memberName</var>).
  - The initializer can refer to other named members that have come before it in the same enclosing 
    lexical `enum` declaration.
  - If the result of evaluating the initializer is an integer, store the result in <var>autoValue</var>.
- When no initializer is specified:
  - The <var>enumMap</var> function is called with <var>memberType</var> as its receiver and the 
    arguments <var>memberName</var> and <var>autoValue</var>.
  - <var>autoValue</var> is incremented by 1.

Enum members are `[[Writable]]`: **false**, `[[Enumerable]]`: **false**, and `[[Configurable]]`: 
**false**.

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
  member whose value on `E` corresponds to`value`.
- `Enum.format(E, value)` - Calls the @@formatEnum method of `E` with argument `value`.
- `Enum.parse(E, value)` - Calls the @@parseEnum method of `E` with argument `value`.
- `Enum.create(members)` - Creates an _enum object_ using the property keys and values of `members`
  as the _enum members_ for the new enum.
- `Enum.flags(descriptor)` - A built-in decorator that modifies the _enum object_ in the following ways:
  - The auto-increment behavior is changed to shift the current auto-increment value left by 1. 
  - The @@parseEnum method is modified to parse a comma-separated string and OR the resulting values
    together. If no corresponding name can be found and the name can be successfully coerced to a number,
    that number is OR'ed with the result.
  - The @@formatEnum method is modified to convert a bitwise combination of flag values into a comma
    separated string of corresponding names. If no corresponding name can be found, the SV of the 
    bits is appended to the string.

```ts
let Enum: {
  keys(E: object): IterableIterator<string | symbol>;
  values(E: object): IterableIterator<any>;
  entries(E: object): IterableIterator<[string | symbol, any]>;
  has(E: object, key: string | symbol): boolean;
  hasValue(E: object, value: any): boolean;
  format(E: object, value: any): string | symbol | undefined;
  parse(E: object, value: string): any;
  create(members: object): object;
  flags(descriptor: EnumDescriptor): EnumDescriptor;
};
```

<!--
# Grammar

```grammarkdown
EnumDeclaration[Yield, Await, Default] :
  `enum` BindingIdentifier[?Yield, ?Await] EnumTail[?Yield, ?Await]
  [+Default] `enum` EnumTail[?Yield, ?Await]

EnumExpression[Yield, Await] :
  `enum` BindingIdentifier[?Yield, ?Await]? EnumTail[?Yield, ?Await]

EnumTail[Yield, Await] :
  `{` EnumBody[?Yield, ?Await] `}`

EnumBody[Yield, Await] :
  EnumElementList[?Yield, ?Await]

EnumElementList[Yield, Await] :
  EnumElement[?Yield, ?Await]
  EnumElementList[?Yield, ?Await] `,` EnumElement[?Yield, ?Await]

EnumElement[Yield, Await] :
  PropertyName[?Yield, ?Await] Initializer[+In, ?Yield, ?Await]?

PrimaryExpression[Yield, Await] :
  ...
  EnumExpression[?Yield, ?Await]

Declaration[Yield, Await] :
  ...
  EnumDeclaration[?Yield, ?Await]

ExportDeclaration :
  `export` `default` EnumDeclaration[~Yield, ~Await, +Default]
  `export` `default` [lookahead ∉ { `function`, `async` [no LineTerminator here] `function`, `class`, `enum` }] AssignmentExpression[+In, ~Yield, ~Await];
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
enum HttpMethods extends String { GET, PUT, POST, DELETE }

typeof HttpMethods.GET === "string"
HttpMethods.GET === "GET"

// ... booleans, ...
enum Switch { on = true, off = false }

typeof Switch.on === "boolean";
Switch.on === true

// ... symbols, ...
enum AlphaBeta extends Symbol { alpha, beta }

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

Enum.parse(AToB, "a") === AToB.a // "b"
Enum.parse(AToB, "b") === AToB.b // "a"

Enum.format(AToB, AToB.a) === "b"
Enum.format(AToB, AToB, b) === "a"

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
    This approach gives you the ability to support both scenarios through the optional `extends` clause.
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