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
enum Colors {
  red,
  green,
  blue
}

enum Numbers {
  zero,
  one,
  two,
  three,
  alsoThree = three
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
  identifierName,
  "string name"
}

// Accessing enum values:
let x = Color.red;
let y = Named["string name"];
```

# Semantics

## Enum Declarations

When an `enum` is declared, its members are evaluated and a new _enum constructor function_ is
created:

  - When called, the constructor function coerces its argument to an enum value (similar to the 
    behavior of `String`, `Number`, and `Boolean`), with precedence given to the earlier 
    declarations: 
    ```js
    enum Numbers { zero, one, two, three, alsoThree = 3 }
    Numbers(1) === Numbers.one
    Numbers(3) === Numbers.three // (as opposed to `Numbers.alsoThree`)
    ```
  - When `new`-ed, the constructor function returns a wrapper `Object` that boxes the enum value
    (Similar to the behavior of `String`, `Number`, and `Boolean`). 

An _enum constructor function_ has a number of common methods, described in the [API](#API) section
below.

Enums can also be created programmatically via the global `Enum.create()` method, also described in
the [API](#API) section.

## Enum Members

Enum members consist of a comma-separated list of enum member names with optional initializers:

- Enum names can be identifiers or string literals.
- The result of the initializer expression must be either `Number`, `String`, `Symbol`, `Boolean`,
  or another `enum` value.
- If the result of the initializer expression is another `enum` value, its _underlying primitive 
  value_ is used instead.
- The initializer can refer to other named members that have come before it in the same enclosing 
  lexical `enum` declaration.
- When no initializer is specified:
  - The default _underlying primitive type_ of an `enum` value is Number.
  - `enum` values auto-increment by `1` starting from `0` or the last initialized Number value.

Enum members are `[[Writable]]`: **false**, `[[Enumerable]]`: **false**, and `[[Configurable]]`: 
**false**.

Enum members can be accessed and modified programmatically via methods on the global `Enum` 
object, as described in the [API](#API) section.

## Enum Values

Enum values are a new primitive type, `enum`, that are composed of two parts: An _underlying 
primitive value_ (which must be either a `Number`, `String`, `Symbol`, or `Boolean`) and a 
reference to its _originating declaration_ (which is used for coercion and equality): 

```js
typeof Numbers.zero === "enum"
Numbers.zero.valueOf() === 0
Numbers.zero.toString() === "zero"

typeof PlayState.idle === "enum"
PlayState.idle.valueOf() === "idle"
PlayState.idle.toString() === "idle"
```

This representation allows multiple use cases:
- [Strict inequality](#Operators): `Numbers.zero !== Colors.red`
- [Bitwise operations](#Operators): `Numbers.one | Numbers.two`
- [Math operations](#Operators): `let state = States.initial; state++;`

### Operators
ECMAScript operators are supported through existing coercion behavior, with a few minor differences:

- Strict equality (`===`) performs no coercion (no change).
- Weak equality (`==`) performs coercion for enum values to their _underlying primitive type_.
- Addition (`+`), Multiplication (`*`), and Binary Bitwise (`&`, `|`) operations perform coercion 
  for enum values per the existing spec text, however if _both_ operands are enum values whose
  _underlying primitive values_ are both `Number` and have the same _originating declaration_, the
  result is coerced back into an enum value with that _originating declaration_.
  ```js
  Numbers.one + 2 === 3 // Coerces `Numbers.one` to Number.
  Numbers.one | Numbers.two === Numbers.three; // Coerces both operands to Number, and the result back to `Numbers`.
  ```
- Update (`++`, `--`) and Bitwise NOT (`~`) operations coerce back to the operand's enum type if its _underlying 
  primitive type_ is `Number`.
- Bitwise Shift (`<<`, `>>`, `>>>`) operations coerce back to the _left_ operand's enum type if its _underlying
  primitive type_ is `Number`.
- Other operations not specified perform coercion as they are specified (no change). 

# API

Enums have the following API:

```ts
type Enum = {
  /** 
   * Gets the string representation of the member name for this `enum` value. 
   */
  toString(): string;

  /** 
   * Gets the _underlying primitive value_ for this `enum` value. 
   */
  valueOf(): string | number | symbol | boolean;

  /** 
   * Gets the _underlying primitive value_ for this `enum` value. 
   */
  [Symbol.toPrimitive](hint: string): string | number | symbol | boolean;
};

type EnumConstructor = {
  prototype: Enum;

  /** 
   * Coerce `value` into an `enum` value with this as its _originating declaration_. 
   */
  (value: string | number | symbol | boolean | Enum): Enum;

  /** 
   * Coerce `value` into an `enum` value with this as its _originating declaration_ and 
   * returns that value as a boxed Object. 
   */
  new (value: string | number | symbol | boolean | Enum): Enum;

  /** 
   * Tests whether the provided value is explicitly declared on this enum. 
   */
  has(value: string | number | symbol | boolean | Enum): boolean;

  /** 
   * Tests whether the provided `enum` value has this enum as its _originating declaration_. 
   */ 
  is(value: Enum): boolean;

  /** 
   * Gets the defined `enum` value whose member name is the provided `name`. 
   */
  forName(name: string | symbol): Enum | undefined;

  /** 
   * Gets the defined `enum` value whose _underlying primitive value_ is the provided value. 
   */
  forValue(value: string | number | symbol | boolean | Enum): Enum | undefined;
};

/** 
 * Global object used to programmatically define and manipulate `enum` declarations. 
 */
let Enum: {
  /** 
   * Creates a new `enum` programmatically, using the keys and values of `members` as the member 
   * names and underlying primitive values. 
   */
  create(members: object): EnumConstructor;

  /** 
   * Adds a new member of an `enum`. 
   * 
   * Unlike `Object.defineProperty` this also defines membership for the purposes of 
   * `%Enum%.has()` and `%Enum%.is()` 
   */
  addMember(F: EnumConstructor, key: string | symbol, 
    value: string | number | symbol | boolean | Enum): EnumConstructor;

  /** 
   * Adds new members of an `enum`. 
   *
   * Unlike `Object.defineProperty` this also defines membership for the purposes of 
   * `%Enum%.has()` and `%Enum%.is()`
   */
  addMembers(F: EnumConstructor, members: object): EnumConstructor;

  /** 
   * Iterates over all of the enum member names of the provided enum. 
   */
  memberNames(F: EnumConstructor): IterableIterator<string | symbol>;

  /** 
   * Iterates over all of the enum member values of the provided enum. 
   */
  memberValues(F: EnumConstructor): IterableIterator<Enum>;

  /** 
   * Iterates over all of the enum member names and values of the provided enum. 
   */
  members(F: EnumConstructor): IterableIterator<[string | symbol, Enum]>;
};
```

<!-- # Grammar -->
<!-- Grammar for the proposal. Please use grammarkdown (github.com/rbuckton/grammarkdown#readme) 
     syntax in fenced code blocks as grammarkdown is the grammar format used by ecmarkup. -->
<!--

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
enum Numbers {
    zero,
    one,
    two,
    three,
}

// Enum values are a new primitive type:
typeof Numbers.zero === "enum";

// Enums have a default underlying type of Number...
Numbers.zero.valueOf() === 0;
Numbers.zero == 0;
Numbers.zero !== 0; // ...but they are not numbers!

// Enum values retain information about their declaration:
Numbers.zero.toString() === "zero";

// Subsequent enum values auto-increment if the preceding declaration was number-like:
Numbers.zero.valueOf() === 0;
Numbers.one.valueOf() === 1;
Numbers.two.valueOf() === 2;

// Enum members are [[Writable]: false, [[Enumerable]]: false, and [[Configurable]]: true:
Numbers.zero = 1; // throws in strict mode, noop otherwise.
Numbers.zero.valueOf() === 0;

// operators involving enum values coerce the enum value to its underlying primitive value. If all 
// operands have an underlying type of Number and come from the same enum declaration, the result 
// of the operation is coerced back to the enum declaration:

// enum + number -> number
Numbers.one + 2 === 3;

// enum + enum (same declaration) -> enum
Numbers.one + Numbers.two === Numbers.three;

// enum + enum (different declarations) -> number
enum Colors { red, green, blue }
Numbers.one + Colors.blue === 3;

// You can coerce a number-like to an enum value through the enum's constructor:
Numbers(0) === Numbers.zero;
Numbers(1) === Numbers.one;

// You can coerce a String to an enum value through the enum's constructor:
Numbers("0") === Numbers.zero;
Numbers("zero") === Numbers.zero;

// You can override the underlying value through an initializer as long as the result is a 
// primitive value, allowing enums for...

// ... strings, ...
enum HttpMethods {
    get = "GET",
    put = "PUT",
    post = "POST",
    delete = "DELETE",
}
typeof HttpMethods.get === "enum";
HttpMethods.get.toString() === "get";
HttpMethods.get.valueOf() === "GET";

// ... booleans, ...
enum Switch {
    on = true,
    off = false,
}
typeof Switch.on === "enum";
Switch.on.toString() === "on";
Switch.on.valueOf() === true;

// ... symbols, ...
enum Symbols {
    alpha = Symbol.for("alpha"),
    beta = Symbol()
}
typeof Symbols.alpha === "enum";
Symbols.alpha.toString() === "alpha";
Symbols.alpha.valueOf() === Symbol.for("alpha");

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

// All enum declaration constructors inherit from a built in %Enum% object with common 
// static methods:

// You can test for membership using `%Enum%.has()`:
Numbers.has(0) === true; // for underlying values
Numbers.has("one") === true; // for enum member names
Numbers.has(Numbers.two) === true; // for enum values
Numbers.has(Numbers(9)) === false; // even though coerced
Numbers.has(9) === false;
Numbers.has(Color.blue) == false; // even though the underlying values are the same

// You can test for enum branding using `%Enum%.is()`:
Numbers.is(0) === false;
Numbers.is("one") === false;
Numbers.is(Numbers.two) === true;
Numbers.is(Numbers(9)) === true; // even though not defined
Numbers.is(Color.blue) === false;

// You can lookup an enum value by key or value using `%Enum%.forKey()` and 
// `%Enum%.forValue()`, respectively:
enum AlphaBeta {
    a = "b",
    b = "a",
}

AlphaBeta.forKey("a") === AlphaBeta.a;
AlphaBeta.forKey("b") === AlphaBeta.b;
AlphaBeta.forValue("a") === AlphaBeta.b;
AlphaBeta.forValue("b") === AlphaBeta.a;

// There is a global `Enum` object with additional capabilities:

// `Enum.create()` lets you create a new enum programmatically:
const SyntaxKind = Enum.create({ 
  identifier: 0, 
  number: 1, 
  string: 2 
});

typeof SyntaxKind.identifier === "enum";
SyntaxKind.identifier.toString() === "identifier";
SyntaxKind.identifier.valueOf() === 0;

// `Enum.addMember()` and `Enum.addMembers()` let you extend an existing enum with new 
// members (for monkeypatch support):
Enum.addMember(HttpMethods, "patch", "PATCH");
Enum.addMembers(Numbers, { "four": 4, "five": 5 });

// You can get the names of all defined members using `Enum.memberNames()`:
Enum.memberNames(Numbers); // ["zero", "one", "two", "three"]
Enum.memberNames(HttpMethods); // ["get", "put", "post", "delete"]

// You can get the values of all defined members using `Enum.memberValues()`:
Enum.memberValues(Numbers); // [0, 1, 2, 3]
Enum.memberValues(HttpMethods); // ["GET", "PUT", "POST", "DELETE"]

// You can get the entries of all defined members using `Enum.members()`:
Enum.members(Numbers); // [["zero", 0], ["one", 1], ...]
Enum.members(HttpMethods); // [["get", "GET"], ["put", "PUT"], ...]
```

# Remarks

- Why a new primitive type and the proposed strict equality semantics? 
  - In prior discussions, there are some preferences for the use of 
    [symbol values](https://esdiscuss.org/topic/propose-simpler-string-constant#content-8), 
    while there are other preferences that include the use of 
    [strings and numbers](https://esdiscuss.org/topic/propose-simpler-string-constant#content-14). 
    This approach gives you the ability to support both scenarios through operand coercion 
    and strict-equality semantics. These semantics can only be defined in terms of a new
    primitive type.
- Why default to Number?
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