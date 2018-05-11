# Proposal for ECMAScript enums

A common and oft-used feature of many languages is the concept of an [Enumerated Type](https://en.wikipedia.org/wiki/Enumerated_type), or `enum`. Enums provide a finite domain of constant values that are regularly used to indicate choices, discriminants, and bitwise flags.

## Status

**Stage:** 0  
**Champion:** Ron Buckton (@rbuckton)

_For more information see the [TC39 proposal process](https://tc39.github.io/process-document/)._

## Authors

* Ron Buckton (@rbuckton)

# Motivations

Many ECMAScript hosts and libraries have various ways of distinguishing types or operations via some kind of discriminant:

- ECMAScript:
  - `[Symbol.toStringTag]`
  - `typeof`
- DOM:
  - `Node.prototype.nodeType` (`Node.ATTRIBUTE_NODE`, `Node.CDATA_SECTION_NODE`, etc.)
  - `DOMException.prototype.code` (`DOMException.ABORT_ERR`, `DOMException.DATA_CLONE_ERR`, etc.)
  - `XMLHttpRequest.prototype.readyState` (`XMLHttpRequest.DONE`, `XMLHttpRequest.HEADERS_RECEIVED`, etc.)
  - `CSSRule.prototype.type` (`CSSRule.CHARSET_RULE`, `CSSRule.FONT_FACE_RULE`, etc.)
  - `Animation.prototype.playState` (`"idle"`, `"running"`, `"paused"`, `"finished"`)
  - `ApplicationCache.prototype.status` (`ApplicationCache.CHECKING`, `ApplicationCache.DOWNLOADING`, etc.)
- NodeJS:
  - `Buffer` encodings (`"ascii"`, `"utf8"`, `"base64"`, etc.)
  - `os.platform()` (`"win32"`, `"linux"`, `"darwin"`, etc.)
  - `"constants"` module (`ENOENT`, `EEXIST`, etc.; `S_IFMT`, `S_IFREG`, etc.)


# Prior Art

* Language: [Concept](url)  


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

enum Mixed {
  number = 123,
  string = "abc",
  boolean = false,
  symbol = Symbol()
}

enum Named {
  identifier,
  "string name",
  [Symbol.for("symbol name")]
}

// enum expressions
let Kind = enum { String, Number, Identifier };

// Accessing enum values:
let red = Color.red;
let x = Named["string name"];
```

# Semantics

- An `enum` values are composed of two parts: 
  - An _underlying primitive value_: `Numbers.zero.valueOf() === 0`, `PlayState.idle.valueOf() === "idle"`.
  - An internal reference to its _originating declaration_: `Numbers.zero.toString() === "zero"`, `PlayState.idle.toString() === "red"`.
- `enum` members can have an initializer:
  - The result of the initializer expression must be either Number, String, Symbol, Boolean, or another `enum` value.
  - If the result of the initializer expression is another `enum` value, its _underlying primitive value_ is used instead.
  - The initializer can refer to other named members of the `enum` that have come before it.
- When no initializer is specified:
  - The default _underlying primitive type_ of an `enum` value is Number.
  - `enum` values auto-increment by `1` starting from `0` or the last initialized Number value.
- `enum` values are a new primitive type: `typeof Numbers.zero === "enum"`.
- `enum` values are strictly equal if they have the same _underlying primitive value_ and _originating declaration_: `Numbers.three === Numbers.alsoThree`, `Numbers.two !== 2`.
- `enum` values are weakly equal to their underlying primitive values: `Numbers.one == 1`.
- `enum` members are `{ [[Writable]: false, [[Enumerable]]: false, [[Configurable]]: true }`.
- Operations involving `enum` values coerce to their underlying primitive value: `Numbers.one + 1 === 2`.
- If all operands have an _underlying primitive type_ of Number and have the same _originating declaration_, then the result of the operation is coerced back to an `enum` value: `Numbers.one + Numbers.two === Numbers.three`.
- An `enum` declaration or expression results in an _enum constructor function_:
  - When called, the constructor function coerces its argument to an enum value, with precedence given to the earliest member declarations: `Numbers(1) === Numbers.one`, `Numbers(3).toString() === "three"` (as opposed to `"alsoThree"`). This is similar to `String`, `Number`, etc.
  - When `new`-ed, the constructor function returns an `Object` that boxes the enum value. This is similar to `String`, `Number`, etc.
- An _enum constructor function_ has a number of common methods, described in the [API](#API) section.
- All `enum` values in the same _originating declaration_ share a common prototype. That common prototype itself has a prototype shared by _all_ `enum` values, regardless of their _originating declaration_.
- In addition to defining an `enum` declaratively, you can also define an enum imperatively using the `Enum` built-in object, described in the [API](#API) section.

# API

Enums have the following API:

```ts
type Enum = {
  /** Gets the string representation of the member name for this `enum` value. */
  toString(): string;

  /** Gets the _underlying primitive value_ for this `enum` value. */
  valueOf(): string | number | symbol | boolean;

  /** Gets the _underlying primitive value_ for this `enum` value. */
  [Symbol.toPrimitive](hint: string): string | number | symbol | boolean;
  [Symbol.toStringTag]: string;
};

type EnumConstructor = {
  prototype: Enum;

  /** Coerce `value` into an `enum` value with this as its _originating declaration_. */
  (value: string | number | symbol | boolean | Enum): Enum;

  /** Coerce `value` into an `enum` value with this as its _originating declaration_ and returns that value as a boxed Object. */
  new (value: string | number | symbol | boolean | Enum): Enum;

  /** Tests whether the provided value is explicitly declared on this enum. */
  has(value: string | number | symbol | boolean | Enum): boolean;

  /** Tests whether the provided `enum` value has this enum as its _originating declaration_. */ 
  is(value: Enum): boolean;

  /** Iterates over all of the enum member names of this enum. */
  keys(): IterableIterator<string | symbol>;

  /** Iterates over all of the enum member values of this enum. */
  values(): IterableIterator<Enum>;

  /** Iterates over all of the enum member names and values of this enum. */
  entries(): IterableIterator<[string | symbol, Enum]>;

  /** Gets the defined `enum` value whose member name is the provided `key`. */
  forKey(key: string | symbol): Enum | undefined;

  /** Gets the defined `enum` value whose _underlying primitive value_ is the provided value. */
  forValue(value: string | number | symbol | boolean | Enum): Enum | undefined;
};

/** Global object used to programmatically define and manipulate `enum` declarations. */
let Enum: {
  /** Creates a new `enum` programmatically, using the keys and values of `members` as the member names and underlying primitive values. */
  create(members: object): EnumConstructor;

  /** Defines a new member of an `enum`. Unlike `Object.defineProperty` this also defines membership for the purposes of `%Enum%.has()` and `%Enum%.is()`*/
  defineMember(F: EnumConstructor, key: string | symbol, value: string | number | symbol | boolean | Enum): EnumConstructor;

  /** Defines new members of an `enum`. Unlike `Object.defineProperty` this also defines membership for the purposes of `%Enum%.has()` and `%Enum%.is()`*/
  defineMembers(F: EnumCosntructor, members: object): EnumConstructor;

  /** Lets you create an enum from key/value pairs */
  fromEntries(iterable: Iterable<[string | symbol, string | number | symbol | boolean | Enum]>): EnumConstructor;
};
```

In concert with the [Decorators](https://github.com/tc39/proposal-decorators) proposal, we would also add the following API:

```ts
type EnumDescriptor = {
  kind: "enum";
  members: ElementDescriptor[];
};

let Enum: {
  create(members: object): EnumConstructor;
  defineMember(F: EnumConstructor, key: string | symbol, value: string | number | symbol | boolean | Enum): EnumConstructor;
  defineMembers(F: EnumCosntructor, members: object): EnumConstructor;
  fromEntries(iterable: Iterable<[string | symbol, string | number | symbol | boolean | Enum]>): EnumConstructor;

  /** Decorator that allows you to control string parsing and formatting for numeric enums that allow for bitwise combinations of flags */  
  flags(): (descriptor: EnumDescriptor) => EnumDescriptor;
}
```

# Grammar

<!-- Grammar for the proposal. Please use grammarkdown (github.com/rbuckton/grammarkdown#readme) 
     syntax in fenced code blocks as grammarkdown is the grammar format used by ecmarkup. -->

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
  EnumElementList[?Yield, ?Await] EnumElement[?Yield, ?Await]

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

// Enum member names can be defined using identifiers, strings, or computed properties:
enum Named {
    id,
    "string name",
    [Symbol.for("computed")]
}

// Enums can also be declared using an expression:
let Kind = enum { identifier, number, string, binary };

// Enums can be exported:
export enum Zoo { lion, tiger, bear };
export default enum { up, down, left, right };

// All enum declaration constructors inherit from a built in %EnumConstructor% object with common 
// static methods:

// You can test for membership using `%EnumConstructor%.has()`:
Numbers.has(0) === true; // for underlying values
Numbers.has("one") === true; // for enum member names
Numbers.has(Numbers.two) === true; // for enum values
Numbers.has(Numbers(9)) === false; // even though coerced
Numbers.has(9) === false;
Numbers.has(Color.blue) == false; // even though the underlying values are the same

// You can test for enum branding using `%EnumConstructor%.is()`:
Numbers.is(0) === false;
Numbers.is("one") === false;
Numbers.is(Numbers.two) === true;
Numbers.is(Numbers(9)) === true; // even though not defined
Numbers.is(Color.blue) === false;

// You can get the names of all defined members using `%EnumConstructor%.keys()`:
Numbers.keys(); // ["zero", "one", "two", "three"]
HttpMethods.keys(); // ["get", "put", "post", "delete"]

// You can get the values of all defined members using `%EnumConstructor%.values()`:
Numbers.values(); // [0, 1, 2, 3]
HttpMethods.values(); // ["GET", "PUT", "POST", "DELETE"]

// You can get the entries of all defined members using `%EnumConstructor%.entries()`:
Numbers.entries(); // [["zero", 0], ["one", 1], ["two", 2], ["three", 3]]
HttpMethods.entries(); // [["get", "GET"], ["put", "PUT"], ["post", "POST"], ["delete", "DELETE"]]

// You can lookup an enum value by key or value using `%EnumConstructor%.forKey()` and 
// `%EnumConstructor%.forValue()`, respectively:
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
const SyntaxKind = Enum.create({ identifier: 0, number: 1, string: 2 });
typeof SyntaxKind.identifier === "enum";
SyntaxKind.identifier.toString() === "identifier";
SyntaxKind.identifier.valueOf() === 0;

// `Enum.defineMember()` and `Enum.defineMembers()` let you extend an existing enum with new 
// members (for monkeypatch support). Unlike `Object.defineProperty` this also defines membership 
// for the purposes of `%EnumConstructor%.has()` and `%EnumConstructor%.is()`.
Enum.defineMember(HttpMethods, "patch", "PATCH");
Enum.defineMembers(Numbers, { "four": 4, "five": 5 });

// `Enum.fromEntries()` lets you create an enum from key/value pairs:
const MyNumbers = Enum.fromEntries(Numbers.entries());
MyNumbers.zero.toString() === "zero";
MyNumbers.zero.valueOf() === 0;

// With the Decorators proposal:

// The @Enum.flags decorator allows you to control string parsing and formatting for numeric enums
// that allow for bitwise combinations of flags:
@Enum.flags
enum Axis {
    ancestors = 1,
    descendents = 2,
    self = 4,

    // within an enum body you can reference preceding members
    ancestorsOrSelf = ancestors | self,
    descendentsOrSelf = descendents | self,
}

(Axis.ancestors | Axis.self) === Axis.ancestorsOrSelf;
(Axis.ancestors | Axis.self).toString() === "ancestorsOrSelf";
(Axis.ancestors | Axis.descendents).toString() === "ancestors, descendents";
Axis("ancestors, self") === Axis.ancestorsOrSelf;
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

<!-- # Prior Discussion -->
<!-- Links to prior discussion topics on https://esdiscuss.org -->
<!-- 
* [Subject](https://esdiscuss.org)
-->

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