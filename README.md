<!--#region:intro-->
# ECMAScript class static initialization blocks

Class `static` blocks provide a mechanism to perform additional static initialization during class 
definition evaluation.

This is not intended as a replacement for public fields, as they provide useful information for 
static analysis tools and are a valid target for decorators. Rather, this is intended to augment
existing use cases and enable new use cases not currently handled by that proposal.

<!--#endregion:intro-->

<!--#region:status-->
## Status

**Stage:** 4  
**Champion:** Ron Buckton (@rbuckton)  

_For detailed status of this proposal see [TODO](#todo), below._  
<!--#endregion:status-->

<!--#region:authors-->
## Authors

* Ron Buckton (@rbuckton)  
<!--#endregion:authors-->

<!--#region:motivations-->
# Motivations

The current proposals for static fields and static private fields provide a mechanism to perform
per-field initialization of the static-side of a class during ClassDefinitionEvaluation, however 
there are some cases that cannot be covered easily. For example, if you need to evaluate statements
during initialization (such as `try..catch`), or set two fields from a single value, you have to 
perform that logic outside of the class definition. 

```js
// without static blocks:
class C {
  static x = ...;
  static y;
  static z;
}

try {
  const obj = doSomethingWith(C.x);
  C.y = obj.y
  C.z = obj.z;
}
catch {
  C.y = ...;
  C.z = ...;
}

// with static blocks:
class C {
  static x = ...;
  static y;
  static z;
  static {
    try {
      const obj = doSomethingWith(this.x);
      this.y = obj.y;
      this.z = obj.z;
    }
    catch {
      this.y = ...;
      this.z = ...;
    }
  }
}
```

In addition, there are cases where information sharing needs to occur between a class with an 
instance private field and another class or function declared in the same scope.

Static blocks provide an opportunity to evaluate statements in the context of the current class 
declaration, with privileged access to private state (be they instance-private or static-private):

```js
let getX;

export class C {
  #x
  constructor(x) {
    this.#x = { data: x };
  }

  static {
    // getX has privileged access to #x
    getX = (obj) => obj.#x;
  }
}

export function readXData(obj) {
  return getX(obj).data;
}
```

## Relation to "Private Declarations"

Proposal: https://github.com/tc39/proposal-private-declarations

The Private Declarations proposal also intends to address the issue of privileged access between two classes, by lifting 
the private name out of the class declaration and into the enclosing scope. While there is some overlap in that respect,
private declarations do not solve the issue of multi-step static initialization without potentially exposing a private
name to the outer scope purely for initialization purposes:

```js
// with private declarations
private #z; // exposed purely for post-declaration initialization
class C {
  static y;
  static outer #z;
}
const obj = ...;
C.y = obj.y;
C.#z = obj.z;

// with static block
class C {
  static y;
  static #z; // not exposed outside of class
  static {
    const obj = ...;
    this.y = obj.y;
    this.#z = obj.z;
  }
}
```

In addition, Private Declarations expose a private name that potentially allows both read and write access to shared private state
when read-only access might be desireable. To work around this with private declarations requires additional complexity (though there is
a similar cost for `static{}` as well):

```js
// with private declarations
private #zRead;
class C {
  #z = ...; // only writable inside of the class
  get #zRead() { return this.#z; } // wrapper needed to ensure read-only access
}

// with static
let zRead;
class C {
  #z = ...; // only writable inside of the class
  static { zRead = obj => obj.#z; } // callback needed to ensure read-only access
}
```

In the long run, however, there is nothing that prevents these two proposals from working side-by-side:

```js
private #shared;
class C {
  static outer #shared;
  static #local;
  static {
    const obj = ...;
    this.#shared = obj.shared;
    this.#local = obj.local;
  }
}
class D {
  method() {
    C.#shared; // ok
    C.#local; // no access
  }
}
```

<!--#endregion:motivations-->

<!--#region:prior-art-->
# Prior Art 

- C#: [Static Constructors](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/static-constructors)  
- Java: [Static Initializers](https://docs.oracle.com/javase/specs/jls/se10/html/jls-8.html#jls-8.7)
<!--#endregion:prior-art-->

<!--#region:syntax-->
# Syntax

```js
class C {
  static {
    // statements
  }
}
```
<!--#endregion:syntax-->

<!--#region:semantics-->
# Semantics

- A `static {}` initialization block creates a new lexical scope (e.g. `var`, `function`, and block-scoped
  declarations are local to the `static {}` initialization block. This lexical scope is nested within the lexical
  scope of the class body (granting privileged access to instance private state for the class).
- A class may have any number of `static {}` initialization blocks in its class body.
- `static {}` initialization blocks are evaluated in document order interleaved with static field initializers.
- A `static {}` initialization block may not have decorators (instead you would decorate the class itself).
- When evaluated, a `static {}` initialization block's `this` receiver is the constructor object of the class
  (as with static field initializers).
- It is a **Syntax Error** to reference `arguments` from within a `static {}` initialization block.
- It is a **Syntax Error** to include a _SuperCall_ (i.e., `super()`) from within a `static {}` initialization block.
- A `static {}` initialization block may contain _SuperProperty_ references as a means to access or invoke static
  members on a base class that may have been overridden by the derived class containing the `static {}`
  initialization block.
- A `static {}` initialization block should be represented as an independent stack frame in debuggers and exception
  traces.

<!--#endregion:semantics-->

<!--#region:examples-->
# Examples

```js
// "friend" access (same module)
let A, B;
{
  let friendA;

  A = class A {
    #x;

    static {
        friendA = {
          getX(obj) { return obj.#x },
          setX(obj, value) { obj.#x = value }
        };
    }
  };

  B = class B {
    constructor(a) {
      const x = friendA.getX(a); // ok
      friendA.setX(a, x); // ok
    }
  };
}
```
<!--#endregion:examples-->

<!--#region:api-->
<!--
# API

> TODO: Provide description of High-level API.
-->
<!--#endregion:api-->

<!--#region:grammar-->
<!-- 
# Grammar

> TODO: Provide the grammar for the proposal. Please use [grammarkdown][Grammarkdown] syntax in 
> fenced code blocks as grammarkdown is the grammar format used by ecmarkup.

```grammarkdown
``` 
-->
<!--#endregion:grammar-->

<!--#region:references-->
# References

* [Stage 0 Presentation](https://docs.google.com/presentation/d/1TLFrhKMW2UHlHIcjKN02cEJsSq4HL7odS6TE6M-OGYg/edit?usp=sharing)

<!--#endregion:references-->

<!--#region:prior-discussion-->
<!-- 
# Prior Discussion

> TODO: Provide links to prior discussion topics on https://esdiscuss.org.

* [Subject](https://esdiscuss.org)   
-->
<!--#endregion:prior-discussion-->

<!--#region:todo-->
# TODO

The following is a high-level list of tasks to progress through each stage of the [TC39 proposal process](https://tc39.github.io/process-document/):

### Stage 1 Entrance Criteria

* [x] Identified a "[champion][Champion]" who will advance the addition.  
* [x] [Prose][Prose] outlining the problem or need and the general shape of a solution.  
* [x] Illustrative [examples][Examples] of usage.  
* [x] High-level [API][API].  

### Stage 2 Entrance Criteria

* [x] [Initial specification text][Specification].  
* [x] [Transpiler support][Transpiler] (_Optional_). 
  * Babel `v7.12.0`
  * TypeScript `v4.4 beta` ([TypeScript Playground](https://www.typescriptlang.org/play?ts=4.4.0-dev.20210628#code/DYUwLgBA5uAaCEAuCAKAxsgwgSggXgD4IA7AVwFsAjEAJwG4AoBtYAQwGd2JMIBvBiBADEAD3wQADI0HswrMAEs0fAYOhxxywhDQA6UdIgBfBieYB7Yu3Ohdwc1BQwwsFMRAB3bimy+6QA))

### Stage 3 Entrance Criteria

* [x] [Complete specification text][Specification].  
* [x] Designated reviewers have [signed off][Stage3ReviewerSignOff] on the current spec text.  
* [x] The ECMAScript editor has [signed off][Stage3EditorSignOff] on the current spec text.  

### Stage 4 Entrance Criteria
> For up-to-date information on Stage 4 criteria, check: [#48](https://github.com/tc39/proposal-class-static-block/issues/48)
* [x] [Test262](https://github.com/tc39/test262) acceptance tests have been written for mainline usage scenarios and [merged][Test262PullRequest].  
* [x] Two compatible implementations which pass the acceptance tests: 
  * [x] [SpiderMonkey][Implementation1] — Partially shipping Shipping [behind a flag in 92](https://github.com/tc39/proposal-class-static-block/issues/48#issuecomment-867967054), Intent to ship [unflagged in 93](https://bugzilla.mozilla.org/show_bug.cgi?id=1725689):  \
    ![SpiderMonkey Example](https://user-images.githubusercontent.com/3902892/123336344-9912f100-d4fa-11eb-949f-b3692d22cd21.png "Example showing class static blocks working in FireFox Nightly")
  * [x] [V8][Implementation2] — Shipping unflagged (at least as of 9.4.146):  \
    ![V8 Example](https://user-images.githubusercontent.com/3902892/129283096-45218040-2056-452b-8418-3f394b616174.png "Example showing class static blocks working in Chrome 9.4.146")
* [x] A [pull request][Ecma262PullRequest] has been sent to tc39/ecma262 with the integrated spec text.  
* [x] The ECMAScript editor has signed off on the [pull request][Ecma262PullRequest].  
<!--#endregion:todo-->

[Process]: https://tc39.github.io/process-document/
[Proposals]: https://github.com/tc39/proposals/
[Grammarkdown]: http://github.com/rbuckton/grammarkdown#readme
[Champion]: #status
[Prose]: #motivations
[Examples]: #examples
[API]: #api
[Specification]: https://arai-a.github.io/ecma262-compare/?pr=2440
[Transpiler]: https://www.typescriptlang.org/play?ts=4.4.0-dev.20210628#code/DYUwLgBA5uAaCEAuCAKAxsgwgSggXgD4IA7AVwFsAjEAJwG4AoBtYAQwGd2JMIBvBiBADEAD3wQADI0HswrMAEs0fAYOhxxywhDQA6UdIgBfBieYB7Yu3Ohdwc1BQwwsFMRAB3bimy+6QA
[Stage3ReviewerSignOff]: https://github.com/tc39/proposal-class-static-block/issues/23
[Stage3EditorSignOff]: https://github.com/tc39/proposal-class-static-block/pull/31
[Test262PullRequest]: https://github.com/tc39/test262/pull/2968
[Implementation1]: https://bugzilla.mozilla.org/show_bug.cgi?id=1712138
[Implementation2]: https://bugs.chromium.org/p/v8/issues/detail?id=11375
[Ecma262PullRequest]: https://github.com/tc39/ecma262/pull/2440
