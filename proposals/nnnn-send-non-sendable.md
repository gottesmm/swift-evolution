# Send Non-Sendable

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm) [Joshua Turcotti](https://github.com/jturcotti)
* Review Manager: TBD
* Status: **Awaiting implementation** or **Awaiting review**
* Implementation: available on public Github: [SendNonSendable.cpp](https://github.com/apple/swift/blob/main/lib/SILOptimizer/Mandatory/SendNonSendable.cpp)
* Upcoming Feature Flag: `SendNonSendable`
* Review: ([pitch](https://forums.swift.org/t/pitch-safely-sending-non-sendable-values-across-isolation-domains/66566))

## Introduction

Swift Concurrency assigns mutable values to specific "isolation domains"
determined by actor and task boundaries. Code running in distinct "isolation
domains" are allowed to execute concurrently. As a result, `SE-SENDABLE` forbids
non-`Sendable` values from being passed over any "isolation boundary" in order
to define away data races. In practice this turns out to be a very significant
semantic restriction. In this document, we propose loosening these rules by
introducing a new SIL analysis that determines whether an arbitrary
non-`Sendable` value can be safely be sent across "isolation boundary".

## Motivation

`SE-SENDABLE` states that non-`Sendable` values cannot be sent across any
"isolation boundary". Thus given the following code that opens a new
`ClientAccount` for a `Client` at a bank:

```swift
/// Not Sendable
class Client { ... }

struct BankAccount {
   /// The amount of funds available in this bank account.
   var amount: Double

   /* ... */
}

actor ClientAccount {
  var client: Client
  var accounts: [BankAccount] = []

  init(_ client: Client, _ initialBalance: Double) { ... }
}

func openNewAccount(initialBalance: Double) -> ClientAccount {
  let client = Client()
  let bankAccount = ClientAccount(c, initialBalance) // Error! 'Client' is non-sendable! This could race!
  return bankAccount
}
```

we get an error in `openNewAccount` when strict concurrency is enabled since
`Client` is not sendable despite us having just constructed the value. This is
overly conservative since there cannot be any races in this code since `client`
does not have any other local uses within `openNewAccount` and `client` was just
constructed implying `client` cannot have any uses outside of
`openNewAccount`. If the language rules allowed the compiler to analyze the uses
of `client`, the compiler could accept this code after proving that this code is
race free.

The simple example above shows the extreme limitations on expressivity caused by
not using a flow-sensitive use based approach as defined currently by Swift's
concurrency model. This will impede the development of new libraries that
pervasively use concurrency or the updating of old libraries to use concurrency
by requiring pervasive auditing of non-`Sendable` types to determine
`Sendability`. To understand the scale of the problem in Apple's public SDK
alone there are ~200k classes each with their own set of APIs implying that
multiples of ~200k APIs would need to be audited. By introducing this flow
sensitive model, we can avoid that problem and provide a light intuitive model
for the use of such non-`Sendable` types.

## Proposed solution

We propose the introduction of a new flow sensitive SIL-analysis that emits
error diagnostics at use sites of non-`Sendable` values that previously were
transferred to a different isolation domain. Of course, our motivating example
does not run afoul of this rule. But if we were to modify `openNewAccount` to
call a logging function on `client`, we would violate our rule since we would be
reusing a value that had already been transferred from a non-isolated context to
an actor-isolated context:

```swift
func openNewAccount(initialBalance: Double) -> ClientAccount {
  let client = Client()
  let bankAccount = ClientAccount(client, initialBalance)
  client.log() // ERROR!
  return bankAccount
}
```

Even though it is unsafe to use `client` in `openBankAccount` after transferring
`client` into `bankAccount`'s isolation domain, we would be safe in using any
other value that could statically be proven as not being accessible by applying
any piece of code to `client`. In order to reason about such values, instead of
reasoning about values directly, we reason about equivalence classes of values
called "isolation regions". Formally, two values `x` and `y` are defined to be
within the same `isolation region` at a program point `p` if:

1. `x` may alias `y` at `p`.
2. `x` or a part of `x` might be referenceable from `y` via iterative access to `y`'s properties at `p`.

This definition ensures that values that are in different "isolation regions"
can be used concurrently since any code that uses `x` could not affect `y`. For
example, if we had two bank accounts one for John and the other for Joanna and
wanted to execute a series of transactions on each account:

```swift
class Transaction { ... }

var johnsTransaction = Transaction()
johnsTransaction.add(withdrawing: 50.0)
johnsTransaction.add(depositing: 100.0)

var joannasTransaction = Transaction()
johnsTransaction.add(depositing: 150.0)
johnsTransaction.add(withdrawing: 50.0)

let johnsAccount = ClientAccount.lookup("John Smith")
let joannasAccount = ClientAccount.lookup("Joanna Schmidt")
johnsAccount.checkingAccount.apply(johnsTransaction)        (1)
joannasAccount.checkingAccount.apply(joannasTransaction)    (2)
```

since we just constructed `johnsTransaction` and `joannasTransaction` we know
that they must be isolated from each other and thus be apart of different
regions. This means that we do not need to worry about any races in between
`joannasTransaction` and `johnsTransaction` since they are isolated from each
other. Thus the use of `joannasTransaction` at `(2)` must be safe and the
analysis must not emit an error.

In contrast, if we wanted to implement an auditing routine on `ClientAccount`
that audited two random bank accounts of a client:

```swift
class Client { ... }
class BankAccount {
  var amount: Double = 0.0
}

actor ClientAccount {
  var client: Client
  var accounts: [BankAccount] = [BankAccount()]
  var randomAccount: BankAccount { ... }
  init(_ client: Client, _ initialBalance: Double) { ... }
}

actor AccountAuditing {
  func prepareAudit(_ x: BankAccount) { ... }
  static var auditor: AccountAuditing
}

extension ClientAccount {
  func transferAccountForAuditing() async {
    let firstRandomAccount = self.accounts.randomAccount
    let secondRandomAccount = self.accounts.randomAccount
    let auditor = AccountAuditing.auditor
    await auditor.prepareAudit(firstRandomAccount)
    await auditor.prepareAudit(secondRandomAccount)
  }
}
```

then at `(1)`, we would consider `firstRandomAccount` and `secondRandomAccount`
to be part of the same region since we do not know if they alias and thus cannot
prove they are isolated from one another. As such, attempting to transfer either
across an isolation boundary would be flagged as a race by the analysis since we
conservatively cannot prove it is safe. This shows an important property of
"isolation regions", if any of the values in the region are transferred across
an isolation boundary then all of the values are considered to be transferred.

By analyzing isolated regions of values, this SIL analysis will ease the
development of libraries that pervasively use swift concurrency without
requiring all types transferred in between isolation domains to be audited for
Sendability.

## Detailed Design

### Isolation Regions in more detail

The formal definition of an isolation region phrased in terms of reachability
and aliasing works well in the abstract but can be hard to apply in
practice. Instead, we suggest that users rely on the following rule of thumb:
given a "generalized" function `y = f(x0, ..., xn)`:

1. All non-Sendable `xi`'s regions are merged into one larger region after `f` executes.
2. If any of `xi` are non-`Sendable` then, `y` is in the same merged region as the
   `xi`. If all `xi` are `Sendable`, then `y` is within a new region that consists only
   of `y`.
3. If `y` is mutable and:
   a. Is not captured by reference then `y`'s previous region is not merged into `y`'s new region. This is called an "assign".
   b. Is captured by reference previously in the current function, then we merge
   the region associated with `y`'s previous value with the resulting region of
   `(2)`.

These rules from the following conservative analysis: without any further
information:

a. Any of the `xi` inside of `f` could become reachable from each other.

b. `y` could be one of the `xi` or alias contents of the `xi`.

c. If `y` previously was captured by reference then the new value stored into
`y` could be referenced via calling the closure.

Of course using type information, we can make this less conservative, but as a
general set of rules, these guide us. Now lets apply these rules to specific
examples to see it in action:

* ``let y = x, var y = x``. Initializing a let or var binding `y` with `x`
  results in `y` being in the same region as `x`. This again follows from `(2)`
  since formally a copy is equivalent to calling a property on `x` that takes
  `x` as self and returns a copy of `x`.

* ``y = x``. Assigning a var binding `y` with `x` results in `y` being in the
  same region as `x`. If `y` is captured by a closure, then `y`'s previous
  assigned region is merged with `x`'s region.

* ``let y = x.f``. Accessing a field `f` on a non-sendable value `x` results in
  a value `y` that must be in the same region as `x`. This follows from `(2)`
  since formally a property access is equivalent to calling a getter passing `x`
  as `self`.

* ``y.f = x``. Assigning `x` into a field `y.f` results in `y.f` being in the
  same region as `x`. This again follows from `(2)`.

* ``closure = { useX(x); useY(y) }``. Capturing non-sendable values `x` and `y`
  results in `x` and `y` being in the same region. This can be viewed as a
  consequence of `(2)` since `x` and `y` are formally arguments to the closure
  formation. This also means that the closure must be part of the same region.

* ``closure = { useXInOut(&x) }``. Capturing a reference to a non-sendable value
  `x` results in closure being placed into `x`'s region and any further
  assignments to `x` being region merges instead of region assigns.

* TODO: Switches

Thus using our simple set of rules above, we can derive the necessary rules for
conservative reachable value sets.

### Function Argument Regions and Self

A callee has very limited information about the arguments passed to it by a
caller a function. For instance, a callee cannot know the passed in value to an
argument or if any of the arguments are classes that alias. Due to this lack of
information, we conservatively must require that all function arguments are
treated as belonging to the same global escaping region.

Since any value within this global escaping region are non-local, we must assume
that they could be used at any time and may even have already been transferred to ano

Any values within this global escaping region cannot be transferred in between
"isolation boundaries" since 

Each function argument from a caller's perspective are treated is treated
as being part of the same region. Additionally, they cannot be transferred since
our the function argument is accessible within our caller and our caller relies
upon us to not transfer the value if we were going to transfer it. This implies
that in this model one can only transfer objects that are defined locally in the
given function. With time, we may be able to reduce this restriction by
introducing the.

Since the `isolation region` that a value belongs to varies as a function
executes, it is possible to write code where two predecessors of a control flow
join point of a value have the value in differing regions. For example if we
wanted to conditionally withdraw money from a client's bank account if the bank
account was not frozen and there were funds available we could write a routine
that chose whether or not 

```swift
class AuditLog { ... }
actor Auditor {
  static var auditor: Auditor { ... }
  func append(_ log: AuditLog) { ... }
}

extension BankAccount {
  var successfulWithdrawalAuditLog: AuditLog { ... }
  var failedWithdrawalAuditLogDueToFrozen: AuditLog { ... }

  mutating func withdrawMoney(_ amountToWithdraw: Double) async -> Bool {
    if amount < amountToWithdraw {
      return false
    }
    amount -= amountToWithdraw
    return true
  }
}

extension ClientAccount {
  func withdrawMoneyIfFundsAvailable(amount: Double) async {
    for i in 0..<self.accounts.count {
      var a = self.accounts[i]
      var auditLog: AuditLog? = nil
      if a.isFrozen {
        if await a.withdrawMoney(amount) {
          auditLog = a.successfulWithdrawalAuditLog
        }
      } else {
        auditLog = a.failedWithdrawalAuditLogDueToFrozen
      }
      if let auditLog = auditLog {
        Auditor.auditor.append(auditLog)
      }
      self.accounts[i] = a
    }
  }
}
```

In this case, the compiler emits the following warnings:

```
bank.swift:86:18: warning: passing argument of non-sendable type 'BankAccount' from actor-isolated context to nonisolated context at this call site could yield a race with accesses later in this function (2 access sites displayed)
        if await a.withdrawMoney(amount) {
                 ^
bank.swift:87:22: note: access here could race
          auditLog = a.successfulWithdrawalAuditLog
                     ^
bank.swift:95:24: note: access here could race
      self.accounts[i] = a
      ~~~~~~~~~~~~~~~~~^~~
bank.swift:93:38: warning: passing argument of non-sendable type 'AuditLog' from actor-isolated context to actor-isolated context at this call site could yield a race with accesses later in this function (1 access site displayed)
        await Auditor.auditor.append(auditLog)
                                     ^~~~~~~~
bank.swift:95:24: note: access here could race
      self.accounts[i] = a
      ~~~~~~~~~~~~~~~~~^~~
```

By our intuitive function isolation rule above, we know that all arguments to a
function within the function body must be initialized to be within the same
region. This applies naturally since without further information, we must assume
that there could be some relation in between them. We could be more aggressive
about this by attempting to use the type system to prove that.

This naturally implies that function arguments in a callee cannot be transferred
across a thread boundary since the value may have another use in its caller.

Every isolation domain in Swift contains a set of "isolation regions" that
correspond

### Global Actors



// DETAILED DESIGN

Since a value's region is defined per program point, the specific region that a value
belongs to can change as a program executes. For instance:

```swift
class ClientMetadata { ... }
class Client {
    var metadata: ClientMetadata
}

var c = Client()
let m = ClientMetadata() (1)
c.metadata = m           (2)
```

at line `(1)`, `m` and `c` are part of separate regions since they cannot alias
and are not reachable from each other. But, once line `(2)` is executed, since
`m` is reachable from `c` via `c.metadata`, `m` and `c` must be in the same
region.

In order for us to be able to reason at compile time using regions, we must
conservatively conclude that two values are part of the same region if they
cannot be proven to not-alias or be reachable from each other. For example, if
we wanted to implement a routine on `ClientAccount` that returned the individual
bank accounts with the largest and smallest amount of money contained within
them:

```swift
class BankAccount {
  var amount: Double
}

extension ClientAccount {
  func getLargestAndSmallest() -> (BankAccount, BankAccount)  {
     let max = self.accounts.max { $0.amount < $1.amount }
     let min = self.accounts.min { $0.amount < $1.amount }       (1)
     return (min, max)
  }
}
```

then at (1), we would need to consider max and min to be part of the same region
since we do not know if they alias.

From a concurrency perspective, region's provide a powerful manner to determine if two values

Given a region `r` at a program point `p`, we know that if any element of `r` is
transferred from one isolation domain to another at `p`:

1. `r` must contain all local values that 
1. We must conservatively assume that /all/ other elements of `r` must also be
transferred to the other isolation domain as well.
2. The region contains all values that could have been transferred 

Since we know that two non-`Sendable` values `v1` and `v2` that are in the same
region may alias or be reachable from each other, we must assume that if we
transfer `v1` from one isolation domain to another, we conservatively may have
also transfered `v2` as well. This yields the general rule that when a single
element of a region is transferred from one is

## Detailed Design

To define such an analysis, we introduce some additional concepts which we
define in the following sections.

### Reachable Value Sets

Given a non-Sendable value `v`, we begin by imprecisely defining `v`'s RVS or
"reachable value set" at a specific program point `p` as the set of program
non-Sendable values that `v` might alias directly at `p` or that can be accessed
via recursive accesses to `v`'s methods and properties at `p`. So in the
following:

```swift
// Not sendable since it is a class
class ClientMetadata { ... }

// Not sendable since it is a class
class Client {
    var name: String
    var metadata: ClientMetadata
}

func createNewClient() -> Client {
  let metadata = ClientMetadata()                               (1)
  let client = Client(name: "Joanna Smith", metadata: metadata) (2)
  ...
}
```

`metadata` at program point `(1)` is in its own reachable value set located in
the global "isolation domain" since it was just constructed. Then at `(2)`, we
construct `client` passing `metadata` to `client`'s initializer resulting in
`metadata` and `client` being in the same reachable value set.

In the above example, `metadata` is a field of `client` and thus is reachable
from `client` implying via our informal definition that they must be in the same
region. But what if we did not have visibility into `client`'s type and just
knew that its initializer took a `ClientMetadata`. In such a situation, to be
conservatively correct, we must still consider `metadata` and `client` to be in
the same RVS since we do not know what `client`'s initializer will actually do
with `metadata`.

This leads us to the formal rule that guides whether two values are within the
same RVS: Given a function `y = f(x0, ..., xn)`:

1. All non-Sendable `xi` are in the same RVS after `f` executes.
2. If any of `xi` are non-Sendable then, `y` is in the same region as the
   `xi`. If all `xi` are Sendable, then `y` is within a RVS that consists only
   of `y`.

Now lets apply this rule to specific examples to see it in action:

* ``let y = x.f``. Accessing a field `f` on a non-sendable value `x` results in
  a value `y` that must be in the same RVS as `x`. This follows from `(2)`
  since formally a property access is equivalent to calling a getter passing `x`
  as self.

* ``let y = x``. Copying `x` into a new value `y` results in `y` being in the
  same RVS as `x`. This again follows from `(2)` since formally a copy is
  equivalent to calling a property on `x` that takes `x` as self and returns a
  copy of `x`.

* ``y.f = x``. Assigning `x` into a field `y.f` results in `y.f` being in the
  same RVS as `x`. This again follows from `(2)`.

* ``closure = { useX(x); useY(y) }``. Capturing non-sendable values `x` and `y`
  results in `x` and `y` being in the same RVS. This can be viewed as a
  consequence of `(2)` since `x` and `y` are formally arguments to the closure
  formation. This also means that the closure must be part of the same RVS.

* TODO: Add vars
* TODO: Switches

Thus using our simple two rules above, we can derive the necessary rules for
conservative reachable value sets.

### Reachable Value Sets and Control Flow

Now that we understand how non-Sendable values become part of the same `RVS`, we
need to consider how they are affected by control flow. Given a program point
`p` at a control flow join point, we say that `x` and `y` must be in the same
RVS if in they are also in the same RVS in any of the predecessor code
blocks. For instance in the following code:

```swift

```

### NonSendable Reachable Value Sets


### Reachable Value 

Given a copyable value `v`, new values are added to `v`'s reachable value set
via the following rules:

* `let x = v`. Creating a new binding `x` for `v`. Since Swift copies are 



1. Conservatively grouping all non-`Sendable` values in the function into
   "reachable sets". A "reachable set" consists of values that may alias and
   as well as any values 

1. Analyzing all uses in the function where the isolation domain crossing
   occurs.
2. Given any such uses that occur after the isolation domain crossing occurs,
   determine if the value or any value reachable from that value.

the totality of uses of
a value in the function where the isolation crossing occurs and determining in a
conservative manner if any values that may alias the non-`Sendable` is used
later in the same function.

It does this by 

To a human programmer, the function `computeAndDisplayData` is clearly safe. Swift concurrency prevents `neighbors` from being sent across isolation domains to the main actor because of the potential for the main actor to race with the remainder of `computeAndDisplayData` over access to `neighbors`. But `computeAndDisplayData` doesn't access `neighbors` again after the call, and doesn't allow it to escape. What is needed is a pass that can tell the difference between safe functions like:

```swift
// SAFE:
func computeAndDisplayData(data : LocationData) async {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors)
  // no subsequent access to `neighbors`
}
```

and unsafe functions like:

```swift
// UNSAFE:
func computeAndDisplayData(data : LocationData) async {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors)
  neighbors.addNewData(receiveNewData()) // racy access to `neighbors`
}
```

or:

```swift
// UNSAFE:
func computeAndDisplayData(data : LocationData) async -> NearestNeigbors {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors)
  return neighbors // allowing `neighbors` to escape
}
```

Current Swift sendable checking is not flow-sensitive, so it cannot determine whether implementations access non-sendable values after they're sent across isolations. This forces it to ban sending non-sendable values across isolations outright.

This motivates the development of the `SendNonSendable` pass, currently available as `-enable-experimental-feature SendNonSendable`. The `SendNonSendable` pass treats non-sendable values linearly: allowing them to be freely used up until they're sent across isolations, at which point they're treated as *transferred* and subsequent usages of them yield diagnostics. This linear checking allows the `SendNonSendable` pass to determine that the safe version of `computeAndDisplay` above is race-free, and suppress any sendable diagnostics that would be thrown about it, and to determine that the unsafe versions are race-prone, and emit informative diagnostics about the potential races.

### Regions

The above unsafe examples are clearly racy because a single value, `neighbors`, is sent across isolations then accessed or escaped. There are also examples in which races could arise where it's slightly less obvious that accessing the sent value could race with accessing the kept value. 

```swift
func computeAndDisplayLargest(datasets : [LocationData]) async {
  var listOfGraphs : [NearestNeighbors]
  for data in datasets {
    listOfGraphs.append(computeNearestNeighbors(data : data))
  }
  
  let largestGraph = listOfGraphs.max(by: { $0.numRootPoints() > $1.numRootPoints() })
  let smallestGraph = listOfGraphs.min(by: { $0.numRootPoints() > $1.numRootPoints() })
  
  await addToDisplay(largestGraph)
  
  smallestGraph.decreaseClusterSize(1)
}
```

In this function, `Largest` attempts to display the largest `NearestNeighbor` graph generated from a list of datasets passed to the function. No single value appears to be both sent across isolations and subsequently accessed. However, there is no guarantee that `largestGraph` and `smallestGraph` are distinct values - they could be the same value! So the access and mutation of `smallestGraph` via `smallestGraph.decreaseClusterSize(1)` could race with the access to `largestGraph` granted to the main actor via `await addToDisplay(largestGraph)`. 

Strict linearity would solve this by preventing aliasing to ever arise in the first place by banning obvious aliasing (e.g. `let x = y`), and as a result preventing implementations of functions like `min` and `max` that provide first class references to objects still present elsewhere in a datastructure. In short - strict linearity enforces "one first class reference at a time" semantics. This is an unnatural programming model, and in addition to outright preventing certain APIs from being implemented it is awkward and cumbersome to work with (try googling "how to write a doubly linked list in Rust"). 

The solution that the `SendNonSendable` pass chose to implement in place of strict, value-level linearity, is *linear regions*. Regions are collections of values that could alias or reference each other, split up so that two values in separate regions *cannot* ever alias or reference each other. Operations such as cross-isolation method calls that would transfer ownership of values instead transfer ownership of *entire regions* at a time - allowing aliasing to arise but preventing it from leading to races. For example, in the above `computeAndDisplayLargest` function, the values `listOfGraphs`, `largestGraph`, and `smallestGraph` would all be considered in the same region - allowing the `SendNonSendable` code to realize that the access  `smallestGraph.decreaseClusterSize(1)` is potentially racy because `smallestGraph`'s region was already transferred to a different isolation. 

To summarize, the proposed `SendNonSendable` pass accomplished the goal of allowing non-sendable values to be sent across isolations while maintaining data-race freedom by:

- Tracking all **non-sendable values** in scope at all program points, partitioning them into *regions* that are permitted to contain arbitrarily aliased values, but are guaranteed not to alias each other
  - For example, track that `listOfGraphs`, `largestGraph` and `smallestGraph` are all in the same region above
- At program points where **non-sendable values** cross isolations, transfer away the region containing that value
  - For example, mark that the region containing `largestGraph` as transferred after the call to `addToDisplay`
- Produce diagnostics upon access to any **non-sendable** values in regions that have already been transfered away
  - For example, prevent the access to `smallestGraph` above because that value is in a transferred region

Note the emphasis passed on **non-sendable values** - values of sendable type are ignored by this pass, and subject to typechecking exactly as currently implemented.

## Detailed design

This proposal does not add new syntax to Swift, nor change any types. The only change it makes to existing behavior is to prevent the emission of the `non_sendable_call_argument` diagnostic that is emitted when a call that is determined to cross isolation domains has non-sendable arguments. Preempting this diagnostic is what allows non-sendable values to now be passed across isolation domains, such as from a nonisolated context to a main actor isolated context as shown above. 

Instead of emitting the `non_sendable_call_argument` diagnostic, the `TypeCheckConcurrency` Sema pass now decorates `ApplyExpr`s with whether they represent an application that crosses isolations. The `SendNonSendable` mandatory SIL pass is then responsible for emitting diagnostics around the subset of isolation-crossing applications that are could yield races, but only that subset. 

The key question then becomes: how does the SIL pass determine which isolation-crossing applications could yield races? 

### <a name="regionrules"></a> At a high level: simple rules for regions

All non-sendable values within a function body are tracked to determine what region they reside in. Some simple rules regarding value initialization:

- If the initializer for a non-sendable value takes any non-sendable arguments, then the regions of all its arguments will be merged together so that they all occupy a single, shared region, after the initializer returns. The initialized value will also reside in that region.
-  If the initializer is passed no arguments, or is passed only sendable arguments, then a fresh region shared with no existing values will be created for the initialized value. 

It is important to keep in mind that these regions are a purely static abstraction, so allocating fresh regions means only choosing a new identifier for that region, not performing a dynamic allocation of any sort. The following code points out region-creation behavior.

```swift
class NonSendable {
  var x : SendableType
  var y : OtherNonSendableType?
  
  init(_ x : SendableType, _ y : OtherNonSendableType? = none) {
    self.x = x; self.y = y
  }
}

// starting region partition: none

let x = SendableType() // x does not have a region because it is sendable
let y = OtherNonSendableType() // y gets a fresh region
let ns1 = NonSendable(x) // ns1 gets a fresh region
let ns2 = NonSendable(x, y) // ns2 gets the same region as y

// ending region partition: {ns1}, {y, ns2}
```

Functions in general actually exhibit the same semantics: non-sendable arguments are merged into a common region, and non-sendable results come from that region or a fresh one in the case of no non-sendable arguments. This is necessary because, without further information, we have to assume the called function creates references between its passed non-sendable arguments.

```swift
func compareBoxContents(_ box0 : Box, _ box1 : Box) {
  if (box0.contents == box1.contents) {
    print("the same")
  } else {
    print("different")
  }
}

func reassignBoxContents(_ box0 : Box, _ box1 : Box) {
  box0.contents = Contents()
  box1.contents = box0.contents
}

// upon initialization, box0 and box1 occupy separate regions
let (box0, box1) = Box(), Box()

compareBoxContents(box0, box1) // this call merges the regions of box0 and box1

// this transfers the region of box0
Task {
  box0.contents.increment()
}

// this is an error, because the region of box1 was transfered
// (same region as box0)
Task {
  box1.contents.increment()
}
```

This function convention is illustrated above. Though the above code is actually perfectly safe, as the call to `compareBoxContents` does NOT introduce aliasing or referencing between `box0` and `box1`, the function convention has to be safe with respect to implementations like `reassignBoxContents`, that have the same static type as `compareBoxContents`, but DO introduce aliasing between the contents of `box0` and `box1`. If  `reassignBoxContents` were emplaced in the above code, then the creation of the two tasks would indeed yield a data race, so the code must be marked unsafe.

### Summary : simple rules for regions

In summary, regions are "created" when non-sendable values such as classes are intialized without any non-sendable arguments. Regions expand in the following ways:

- `let y = x.f`: reading a non-sendable-typed field of `x` yields a value `y` in the same region as `x`
- `let y = x`: creating an alias `y` yields a value in the same region as `x`
- `y.f = x`: creating a reference from a non-sendable value `y` to a non-sendable value `x` merges their regions
- `z = { ... x ... y ...}`: capturing non-sendable values `x` and `y` in a closure merges their regions, and yields a new closure `z` in the same region as `x` and `y`
- `if/for/while/switch` (control flow): if `x` and `y` are in the same region in any predecessor basic block of a function, then they will be merged into the same region in all successors of that basic block

The result of this is that at the point of a send of a non-sendable value between isolation domains, all other values that could potentially be aliased or referenced from/by it, even along just one control flow path to the send, will be known to be in the same region as it, and will be marked "transferred " and be rendered inaccessible after the send.

### Crossing Isolation Domains

The function conventions outlined above apply to function calls that will execute synchonrously in the same isolation domain as the caller. Calls that can cross isolation domains have slightly different semantics: instead of just being merged, the regions of all non-sendable arguments to a cross-isolation call are *transferred away* as well. The reason this is necessary is different for the two types of calls that pass values across isolation domains.

#### Calls into actors

When a non-sendable value is passed to an actor-entering call, i.e. a call to an actor method from any isolation except that actor's own isolation, the value could "escape" into the storage of that actor. So even after the (necessarily async) call returns, the actor could process a new request concurrently with the caller executuing more code. Since the originally passed value is now accesssible to the actor through its storage, the caller must be banned from accessing it to prevent a data race. Thus actor-entering calls must transfer their arguments. The following code illustrates this:

```swift
actor NationalPark {
  var visitors : [Visitor]
  
  func admitVisitor(_ visitor : Visitor) {
    visitors.append(visitor)
  }
  
  func admitAndGreetVisitor(_ visitor : Visitor) {
    // this call does not cross isolations,
    // so it does not transfer `visitor`, it just merges its region
    // with the region of `self`
    admitVisitor(visitor) // callsite 1 - safe
    
    // so access to `visitor` is permitted
    visitor.greet()
  }
}

func visitParks(_ parks : [NationalPark], _ visitorName : String) async {
  let visitor = Visitor(visitorName)
  
  for park in parks {
    // this call enters the `park` actor, so it
    // transfers `visitor`
    // accessing `visitor` in a loop is thus an error
    await park.admitVisitor(visitor) // callsite 2 - ERROR
  }
}
```

Note that the call to `admitVisitor` within the actor method `admitAndGreetVisitor` labelled `callsite 1` does NOT transfer its argument, and continued access after the callsite is permitted. This is because that call is NOT a cross-actor call, so there is no risk of another actor holding a reference to the argument.

On the other hand, the call to `admitVisitor` in the nonisolated function `visitParks` labelled `callsite 2` enters an actor, so it DOES transfer its argument. This makes the written code unsafe, and indeed it yields an error. If the code were allowed, then multiple actors could concurrently reference the same `visitor` object from their storage, racing on it. 

#### Task creation

Special functions that create new tasks, such as `Task.init`, `Task.detached`, or `TaskGroup.addTask`, or functions that otherwise cause passed closures to execute concurrently with the caller such as `MainActor.run`, must also transfer their argument. In particular, the argument will be a closure, so any values captured in the closure will be in the same region as that closure, and transferring it will transfer those values. This transfer prevents races on those values between the caller and the executor of the passed closure. This prevents the following possible racy code from being expressible as well:

```swift
func visitParksParallel(_ parks : [NationalPark], _ visitorName : String) {
  let visitor = Visitor(visitorName)
  
  for park in parks {
    Task {
      await park.admitVisitor(visitor) // will error - closure creation transfers `visitor`
    }
  }
}
```

User-defined functions that takes non-sendable closures should not, in general, exhibit isolation-crossing semantics and transfer their arguments, so it will be necessary to inform the `SendNonSendable` analysis of the special functions such as `Task.init`, `Task.detached`, and `MainActor.run` that execute their passed closure under different isolation than or concurrently with the caller. Three possible approaches for this are:

1. Hard-code a list of such functions - possible but likely considered bad practice
2. Mark such functions with an annotation such as `@IsolationCrossing` at the source level where they're defined - desirable only if [transferring](#transferringargs) is unsupported
3. Use the [transferring](#transferringargs) qualifier on the arguments to such functions to enforce transferrance of their arguments at the calcite

These approaches are listed in increasing order of desirability. Implementing the `transferring` arg qualifier and adding it to the signatures of these functions is the best solution, and indeed the only one in which the implementations of these library functions themselves will typecheck. An attribute such as `@IsolationCrossing` could be used in lieu of the `transferring` feature, but would require some care to make it seem natural and play nicely with other available attributes.

Additionally, to achieve the desired level of expressivity, it will be necessary to *remove* the source-level `@Sendable` annotation on the types of the arguments to these functions. Without the introduction of the `SendNonSendable` pass, race-freedom is attained for functions such as `Task.detached` only by ensuring they only take `@Sendable` args. This prevents entirely the passing of closures that capture non-sendable values to these functions (as such closures are necessarily `@Sendable`). By implementing the `SendNonSendable` pass, sendable closures will still be able to be passed freely to these functions, but non-sendable closures will not be outright banned - rather they will be subject to the same flow-sensitive region-based checking as all other non-sendable values passed to isolation-crossing calls: if their region has not been transferred the call will be allowed, and otherwise it will throw an error. This change (removing `@Sendable` from the argument signatures of these functions) is necessary.

### Diagnostics

The easiest way to generate diagnostics for the `SendNonSendable` pass would be to note each code site at which a value is accessed, but is known by the pass to be in a transferred region. These diagnostics would read something like:

```swift
func giveBoxToActor(a : MyActor) async {
  let b = Box()
  
  await a.accessBox(b)
  
  if (b.contents >= 7) { // warning: value of non-sendable type `Box` accessed here, but could've been sent to concurrently executing code above, yielding a potential raace
		....
}
```

Entirely correct semantics for this pass could be implemented with this diagnostic style, i.e. diagnostics could be thrown iff the pass discovers an error, but this style of diagnostics is potentially confusing to programmers. Although it is not the site at which the error is discovered, the site at which the isolation-crossing send actually took place is likely a much more logical place to put the warning. This is likely because the calls that cross isolations are trivially known to be sites at which concurrency, and concurrent communication, comes into play. The sites at which values in transferred regions are accessed, however, could be any valid AST node in the Swift language. At a high level, it seems reasonable to expect programmers to think the most carefully about the values being sent at the points concurrent communciation is performed. Thus, although a race arises from the combination of a value sent at such a point with a value in the smae region accessed later in the function, highlighting the point at which the value sent allows the programmer to continue to focus their debugging efforts largely on the points at which concurrency communication is performed. In line with this reasoning, the `SendNonSendable` implementation instead emits diagnostics like the following:

```swift
func giveBoxToActor(a : MyActor) async {
  let b = Box()
  
  await a.accessBox(b) // warning: passing argument of non-sendable type 'Box' from nonisolated context to actor-isolated context at this call site could yield a race with accesses later in this function (1 access site displayed)
  
  if (b.contents >= 7) { // note: access here could race
		....
}
```

If there are multiple sites at which values in the region of the sent value are accessed, they will all be displayed:

```swift
func compareListElems(a : MyActor) async {
  let myList : [NonSendableElem] = genList()
  
  let elem0 = myList.min(by : {$0 < $1})
  let elem1 = myList.max(by : {$0 < $1})
  
  await a.displayElem(elem0) // warning: passing argument of non-sendable type 'NonSendableElem' from nonisolated context to actor-isolated context at this call site could yield a race with accesses later in this function (2 access sites displayed)
  
  print("min: \(elem0)") // note: access here could race
  print("max: \(elem1)") // note: access here could race
}
```

There is one other category of diagnostic emitted by this pass. As described in the [section above](#regionrules), the default function convention assumes that functions do not transfer the regions of their arguments (including `self`). Thus making an isolation-crossing call passing any non-sendable values in the same region as a non-sendable `self` value, or any non-sendable args, will be an error.

<a name="passtoactor"></a>

```swift
func passToActor(a : MyActor, v : NonSendableValue) async {
  await a.foo(v) // warning: call site passes `self` or a non-sendable argument of this function to another thread, potentially yielding a race with the caller
}
```

As the diagnostic message indicates, this warning is necessary because function conventions allow code that continues using non-sendable arguments after they are passed to a non-isolation-crossing call.

```swift
func genAndPassTo(a : MyActor) async {
  let v = NonSendableValue()
  
  // call does NOT transfer v
  await passToActor(a, v)
  
  // access here allowed
  print(v)
}
```

To allow the function `passToActor` above to typecheck, a `transferring`-style annotation is needed. See the section [Transferring args](#transferringargs) below.

## Source compatibility

All previously valid, well-typed Swift code will still be valid, well-typed Swift code. 

## ABI compatibility

No currently planned ABI changes, except as indicated above to potentially change the signatures of functions like `MainActor.run`, which should be possible to do without an ABI break. In the future, we could possibly have to add more information to function signatures to support more general operations on the region partition, such as ensuring that result values from functions come from fresh regions or the regions of arguments; or allowing two arguments to a class method to come from a region distinct from `self` but the same as each other.

## Implications on adoption

The `SendNonSendable` pass, if adopted, would encourage the development of libraries that pervasively use Swift concurrency, but do not enforce sendability on all types communicated between isolation domains. This is a large expressivity win, but could fundamentally change the way libraries are written, and this have a large impact if the feature were adopted then rolled back. 

Additioanlly, API designers will have to expect that, by default, values of types they define can be used in a concurrent context; sending them between isolation domains. If they want to be able to maintain the ability to explicitly declare that values of their types can never cross isolations, an [additional language feature would be required](#sendableunavailability).

## Future directions

### <a name="transferringargs"></a>Transferring args

In the current implementation of `SendNonSendable`, functions cannot transfer their arguments. This greatly constricts allowed programming patterns. To send a value to another isolation domain, that value *must* have been initialized locally, not read from an argument or from `self`'s storage. Allowing values from `self`'s storage to be safely sent to other threads is the subject of the `iso` fields extension [discussed below](#iso) and is a bit involved, but allowing arguments to be sent (i.e. transferred), is much simpler. All that is necessary is to add a `transferring` annotation to function signatures that indicates that certain arguments could be transferred by the end of the function body. As a simplest example, the following code shows the behavior of the `transferring` annotation:

```swift
func passToActorTransferring(a : MyActor, transferring v : NonSendableValue) async {
  await a.foo(v)
}

func genAndPassTransferring(a : MyActor) async {
  let v = NonSendableValue()
  
  // call transfers v
  await passToActorTransferring(a, v)
  
  // access here NOT allowed
  print(v)
}
```

Unlike the prior function `passToActor` [defined above](#passtoactor), `passToActorTransferring` *does* typecheck now, but `genAndPassToTransferring` does not - the inverse situation of the non-transferring functions.

`transferring` parameters are a very natural programming pattern. Without them, it is only possible for non-sendable values to ever make a single hop between domains. They are also very easy to implement, and might even be done so before the acceptance of this proposal. The largest difficulty with the addition of this feature is ergonomic interplay with the existing `transferring` annotation that exists in the Swift language. The existing `transferring` keyword focuses on non-copyable types - and specifies ownership conventions for handling refcounts and deallocation. This is related to the idea of `transferring` needed by `SendNonSendable`(let's call it `region-transferring` for now), and in fact, any case in which a parameter is `region-transferring`, it should also be `transferring` (in the existing Swift, non-copyable, sense). Unfortunately, the converse does not hold. For example, parameters to initializers and setters are usually `transferring`, but they should not be `region-transferring`. The reason for this is that the region-based typechecking of `SendNonSendable` is able to track the fact that by passing values to a setter or initializer, ownership has been transferred, but to a known target: the region of `self` of the called method. Thus by not marking such parameters as `region-transferring`, they can still be used after being passed to a setter or initializer as long as the set or initialized value is not transferred. If all such methods' parameters were `region-transferring`, then even if `self` were not transferred, the arguments to the methods would not be accessible after the call.

It is worth noting that for any methods meant to be called only in isolation-crossing contexts, it is a strict gain of expressivity to mark their non-sendable arguments as `transferring`; at callsites, transferance is enforced anyways, and within the function body, the freedom to transfer the arguments again by passing to a third isolation domain would be attained. This provides further evidence that exposing it as an annotation would be useful.

To concretely illustrate the semantics of this extension, `transferring` parameters would:

- transfer the regions of their non-sendable arguments at callsites *whether or not* the callsite is isolation-crossing (in contrast to non-`transferring` parameters that only transfer their arguments' regions if the callsite is isolation-crossing)
- be allocated separate regions from `self` and all other arguments in the region partition used at the entry to function bodies.

It is of note that there is technically an even more general approach that expands on the second point of semantics above by allowing two `transferring` parameters to be assumed to come from the same region as each other. Without this feature, `transferring` parameters would always have to be passed values known to be in a separate region from all other arguments at the callsite. With this feature, two values that are possibly aliases of each other, or possibly reference each other, could be passed to two `transferring` arguments. The benefits of this further extension are less concrete that `transferring` itself, so it is not likely this will be introduced soon.

### Returning fresh

Another way that function signatures could be made more expressive is through their results. In the current `SendNonSendable`, there is no way to return non-sendable values from an isolation-crossing function. In fact, the diagnostics produced for obtaining non-sendable results across isolation boundaries that arise from `TypecheckConcurrency` are not even suppressed. This makes code such as the following impossible:

```swift
func generateFreshPerson(_ name : String, _ age : Int, _ ancestryMap : AncestryMap) -> Person {
  let person = Person(name, age)
  if (ancestryMap.containsChild(person)) { ... /* do some logic */ }
  
  return person
}
```

This code basically wraps an initializer with extra logic, and aims to return a non-sendable result to the caller in a fresh region. Unfortunately, the current `SendNonSendable` pass does not allow its result to be used. A useful extension would allow functions with non-sendable results to allow those results to be accessible to cross-isolation callers in the following cases:

- For methods that are not actor methods, the result of isolation-crossing calls to those methods are always available to the caller. 
  - By default, results are provided in the same region as `self` and any other arguments (except `transferring` arguments, [see above](#transferringargs))
  - If the `fresh` keyword is placed on the result type in the function signature (e.g. `func generateFreshPerson(...) -> fresh Person`), then the result is provided in a fresh region
- For actor methods:
  -  If the result is not annotated with the `fresh` keyword, then it is never accessible to cross-isolation callers, as the result is in the same region as actor-isolated storage, which is not accessible to the caller.'
  - If the result is annotated with `fresh`, then it is provided to the caller in a fresh region just as for non-actor methods.

Running the `SendNonSendable` pass now involved also ensuring that any values used as `fresh` non-sendable results do indeed come from a region distinct from the region of self and any args.

### <a name="iso"></a>`iso` fields

In the system outlined up to this point, a notable weak point lies in that regions tend to grow very large. In fact, any data structures, including actors and all their stroage, will comprise a single region. As shown in the rest of this proposal, this still allows a large variety of concurrent programming patterns currently impossible in Swift, but it also bans many more. For example, it prevents actors from sending parts of their state to other actors and replacing it with fresh values, such as the following:

```swift
actor IndecisiveBox {
  var contents : NonSendable?

  // set `contents` to the passed `val`, and return the old value
  // of `contents`
  func replaceContents(_ val : NonSendable?) -> NonSendable? {
    let oldContents = contents
    contents = val
    return oldContents
  }
  
  func swapWithOtherBox(_ otherBox : IndecisiveBox) async {
    let otherContents = await otherBox.replaceContents(contents) // warning: call site passes `self` or a non-sendable argument of this function to another thread, potentially yielding a race with the caller
    contents = otherContents
  }
}
```

This could be a perfectly safe pattern, but in the current `SendNonSendable` pass will produce diagnostics as shown above. The issue is that it is not statically known that it is safe to send `contents` to another thread while continuing to access the rest of the actor's storage. To this end, the system could introduce the `iso` keyword. 

The `iso` keyword, when placed on fields, for example as `iso var contents : NonSendable` above, would indicate that instead of tracking the source and target of the reference (here, `self` and `self.contents`) as necessarily belonging to the same region, they could belong to *different* regions. This allows the above code to typecheck, but comes at the cost of greater complexity to the analysis. For more details see (the PLDI paper)[https://www.cs.cornell.edu/andru/papers/gallifrey-types/].

### Global actors

Global actors can have state in the form of actor-isolated global variables. Unlike non-isolated global variables, global actor isolated variables will be able to safely be non-sendable, but they need to be modelled as residing in a `self`-like region dedicate to that global actor in functions isolated to the global actor. This will ensure that non-sendable values read from global actor isolated state cannot be sent to another isolation domain, and, via the region mechanim, that any values that could reference or alias global actor isolated state cannot be sent to another isolation domain

### Async let, and task completion

When structured tasks or statements executed with `async let` complete, not just their possibly non-sendable result but any non-sendable values captured by them should become accessible to the caller. This is not yet implemented.

### <a name="sendableunavailability"></a>`Sendable` unavailability

Current `Sendable` checking allows for the following declaration:

```swift
@available(*, unavailable)
extension T: Sendable { }
```

This ensures that `T` will never be `Sendable`, and allows API designers to express a constraint that values of type `T` should never become available in any isolation domain except the one they were created in. With the introduction of the `SendNonSendable` pass, declaring the `Sendable` protocol `unavailable` for a type will still have the effect of preventing it from being made to confom to `Sendable`, and thus preventing values of that type from being sent without using the `SendNonSendable` pass to ensure no aliases of the value linger, but it will not have the effect of ouright preventing values from being sent. 

It could potentially be useful to still expose a way for developers to ensure values of their defined types are never able to be sent, which would involve a new annotation or protocol such as `~Transferrable` or `~Sendable`. Values of such types would still need to be tracked, as their region labels are necessary to ensure closure under aliasing and references still holds for the regions of non-annotated values, but at the point of a send between isolations, attempting to send any `~Transferable` type would always yield a diagnostic.

It is worth evaluating whether this language feature is actually something developers would find useful/should use; what are the concrete use cases?

## Alternatives considered

### Leaving sendability requirements as is

### Relying on move-only (i.e. purely linear) types

### Non-inferred regions

### A finer capabilities lattice

## Acknowledgments

This proposal is based on joint work with Mae Milano and Andrew Myers, published in the PLDI 2022 paper (A Flexible Type System for Fearless Concurrency)[https://www.cs.cornell.edu/andru/papers/gallifrey-types/]. Doug Gregor and Kavon Farvardin assisted with the development of the `SendNonSendable` implementation and this proposal as well.
