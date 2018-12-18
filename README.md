# Ordered Collection Diffing
* Proposal: SE-NNNN
* Authors: Scott Perry
* Review Manager: TBD
* Status: Prototype

## Introduction

This proposal describes additions to the standard library that provide diffing/patching functionality for ordered collection types.

## Motivation

This proposal is inspired by the convenience of the `diffutils` suite when interacting with text files, and the reluctance to solve similar problems in code with `libgit2`. Representing, manufacturing, and applying transactions between states today requires writing a lot of error-prone code.

A lot of state management patterns would benefit from improvements in this area, including undo/redo stacks, generational stores, and syncing differential content to a service.

## Proposed solution

The concept of an ordered collection is formalized with a new protocol, and new types representing the differences between ordered collections are introduced along with methods that support their production and application among compatible types.

Using this API, a line-by-line three-way merge can be performed in a few lines of code:

``` swift
// Split the contents of the sources into lines
let baseLines = base.components(separatedBy: "\n")
let theirLines = theirs.components(separatedBy: "\n")
let myLines = mine.components(separatedBy: "\n")
    
// Create a difference from base to theirs
let diff = theirLines.difference(from:baseLines)
    
// Apply it to mine, if possible
guard let patchedLines = myLines.applying(diff) else {
    print("Merge conflict applying patch, manual merge required")
    return
}
    
// Reassemble the result
let patched = patchedLines.joined(separator: "\n")
print(patched)
```

## Detailed design

### Formalizing ordered collections

The Swift standard library lacks a protocol for operations that are only valid over `Collection` types with a strong sense of order, such as `elementsEqual(:)`. The correctness of diffing operations also depend on order, so the Swift standard library team has requested that we use this opportunity to formalize this characteristic with a new protocol, `OrderedCollection`:

``` swift
/// An ordered collection treats the structural positions of its elements as
/// part of its interface. Differences in order always affect whether two
/// instances are equal.
///
/// For example, a tree is an ordered collection; a dictionary is not.
@available(swift, introduced: 5.1)
public protocol OrderedCollection : Collection
    where SubSequence : OrderedCollection
{
    /// Returns a Boolean value indicating whether this ordered collection and
    /// another ordered collection contain equivalent elements in the same
    /// order, using the given predicate as the equivalence test.
    ///
    /// The predicate must be a *equivalence relation* over the elements. That
    /// is, for any elements `a`, `b`, and `c`, the following conditions must
    /// hold:
    ///
    /// - `areEquivalent(a, a)` is always `true`. (Reflexivity)
    /// - `areEquivalent(a, b)` implies `areEquivalent(b, a)`. (Symmetry)
    /// - If `areEquivalent(a, b)` and `areEquivalent(b, c)` are both `true`,
    ///   then `areEquivalent(a, c)` is also `true`. (Transitivity)
    ///
    /// - Parameters:
    ///   - other: An ordered collection to compare to this ordered collection.
    ///   - areEquivalent: A predicate that returns `true` if its two arguments
    ///     are equivalent; otherwise, `false`.
    /// - Returns: `true` if this ordered collection and `other` contain
    ///   equivalent items, using `areEquivalent` as the equivalence test;
    ///   otherwise, `false.`
    ///
    /// - Complexity: O(*m*), where *m* is the lesser of the length of the
    ///   ordered collection and the length of `other`.
    func elementsEqual<C>(
       _ other: C, by areEquivalent: (Element, C.Element) throws -> Bool
    ) rethrows -> Bool where C : OrderedCollection
}

extension OrderedCollection {
    /// Returns the difference needed to produce the receiver's state from the
    /// parameter's state, using the provided closure to establish equivalence
    /// between elements.
    ///
    /// This function does not infer element moves, but they can be computed
    /// using `OrderedCollectionDifference.inferringMoves()` if
    /// desired.
    ///
    /// Implementation is an optimized variation of the algorithm described by
    /// E. Myers (1986).
    ///
    /// - Parameters:
    ///   - other: The base state.
    ///   - areEquivalent: A closure that returns whether the two
    ///     parameters are equivalent.
    ///
    /// - Returns: The difference needed to produce the reciever's state from
    ///   the parameter's state.
    ///
    /// - Complexity: O(*n* * *d*), where *n* is `other.count + self.count` and
    ///   *d* is the number of differences between the two ordered collections.
    public func difference<C: OrderedCollection>(
        from other: C, by areEquivalent: (Element, C.Element) -> Bool
    ) -> OrderedCollectionDifference<Element> where C.Element == Self.Element
}

extension OrderedCollection where Element: Equatable {
    /// Returns the difference needed to produce the receiver's state from the
    /// parameter's state, using equality to establish equivalence between
    /// elements.
    ///
    /// This function does not infer element moves, but they can be computed
    /// using `OrderedCollectionDifference.inferringMoves()` if
    /// desired.
    ///
    /// Implementation is an optimized variation of the algorithm described by
    /// E. Myers (1986).
    ///
    /// - Parameters:
    ///   - other: The base state.
    ///
    /// - Returns: The difference needed to produce the reciever's state from
    ///   the parameter's state.
    ///
    /// - Complexity: O(*n* * *d*), where *n* is `other.count + self.count` and
    ///   *d* is the number of differences between the two ordered collections.
    public func difference<C>(from other: C) -> OrderedCollectionDifference<Element>
        where C: OrderedCollection, C.Element == Self.Element
    
    /// Returns a Boolean value indicating whether this ordered collection and
    /// another ordered collection contain the same elements in the same order.
    ///
    /// This example tests whether one countable range shares the same elements
    /// as another countable range and an array.
    ///
    ///     let a = 1...3
    ///     let b = 1...10
    ///
    ///     print(a.elementsEqual(b))
    ///     // Prints "false"
    ///     print(a.elementsEqual([1, 2, 3]))
    ///     // Prints "true"
    ///
    /// - Parameter other: An ordered collection to compare to this ordered
    ///   collection.
    /// - Returns: `true` if this ordered collection and `other` contain the
    ///   same elements in the same order.
    ///
    /// - Complexity: O(*m*), where *m* is the lesser of the `count` of the
    ///   ordered collection and the `count` of `other`.
    public func elementsEqual<C>(_ other: C) -> Bool
        where C : OrderedCollection, C.Element == Element
}
```

A number of existing `Collection` types in the standard libraries will also adopt `OrderedCollection`:

``` swift
extension Array : OrderedCollection {}
extension ArraySlice : OrderedCollection {}
extension ClosedRange : OrderedCollection where Bound : Strideable, Bound.Stride : SignedInteger {}
extension CollectionOfOne : OrderedCollection {}
extension ContiguousArray : OrderedCollection {}
extension CountingIndexCollection : OrderedCollection where Base : OrderedCollection {}
extension EmptyCollection : OrderedCollection {}
extension Range : OrderedCollection where Bound : Strideable, Bound.Stride : SignedInteger {}
extension Slice : OrderedCollection where Base : OrderedCollection {}
extension String : OrderedCollection {}
extension Substring : OrderedCollection {}
extension UnsafeBufferPointer : OrderedCollection {}
extension UnsafeMutableBufferPointer : OrderedCollection {}
extension UnsafeMutableRawBufferPointer : OrderedCollection {}
extension UnsafeRawBufferPointer : OrderedCollection {}

// In Foundation:
extension IndexPath : OrderedCollection {}
extension DataProtocol : OrderedCollection {}
```

The `difference(from:)` method on `OrderedCollection` produces a difference type, which is defined as:

``` swift
/// A type that represents the difference between two ordered collection states.
@available(swift, introduced: 5.1)
public struct OrderedCollectionDifference<ChangeElement> {
    /// A type that represents a single change to an ordered collection.
    ///
    /// The `offset` of each `insert` refers to the offset of its `element` in
    /// the final state after the difference is fully applied. The `offset` of
    /// each `remove` refers to the offset of its `element` in the original
    /// state. Non-`nil` values of `associatedWith` refer to the offset of the
    /// complementary change.
    public enum Change {
        case insert(offset: Int, element: ChangeElement, associatedWith: Int?)
        case remove(offset: Int, element: ChangeElement, associatedWith: Int?)
    }

    /// Creates an instance from a collection of changes.
    ///
    /// For clients interested in the difference between two ordered
    /// collections, see `OrderedCollection.difference(from:)`.
    ///
    /// To guarantee that instances are unambiguous and safe for compatible base
    /// states, this initializer will fail unless its parameter meets to the
    /// following requirements:
    ///
    /// 1) All insertion offsets are unique
    /// 2) All removal offsets are unique
    /// 3) All offset associations between insertions and removals are symmetric
    ///
    /// - Parameter changes: A collection of changes that represent a transition
    ///   between two states.
    ///
    /// - Complexity: O(*n* * log(*n*)), where *n* is the length of the
    ///   parameter.
    public init?<C: Collection>(_ c: C) where C.Element == Change

    /// The `.insert` changes contained by this difference, from lowest offset to highest
    public var insertions: [Change] { get }
    
    /// The `.remove` changes contained by this difference, from lowest offset to highest
    public var removals: [Change] { get }
}

/// An OrderedCollectionDifference is itself a RandomAccessCollection.
///
/// The `Change` elements are ordered as:
///
/// 1. `.remove`s, from highest `offset` to lowest
/// 2. `.insert`s, from lowest `offset` to highest
///
/// This guarantees that applicators on compatible base states are safe when
/// written in the form:
///
/// ```
/// for c in diff {
///     switch c {
///     case .remove(offset: let o, element: _, associatedWith: _):
///         arr.remove(at: o)
///     case .insert(offset: let o, element: let e, associatedWith: _):
///         arr.insert(e, at: o)
///     }
/// }
/// ```
extension OrderedCollectionDifference : Collection {
    public typealias Element = OrderedCollectionDifference<ChangeElement>.Change
    public struct Index: Comparable, Hashable {}
}

extension OrderedCollectionDifference.Change: Equatable where ChangeElement: Equatable {}
extension OrderedCollectionDifference: Equatable where ChangeElement: Equatable {}

extension OrderedCollectionDifference.Change: Hashable where ChangeElement: Hashable {}
extension OrderedCollectionDifference: Hashable where ChangeElement: Hashable {
    /// Infers which `ChangeElement`s have been both inserted and removed only
    /// once and returns a new difference with those associations.
    ///
    /// - Returns: an instance with all possible moves inferred.
    ///
    /// - Complexity: O(*n*) where *n* is `self.count`
	public func inferringMoves() -> OrderedCollectionDifference<ChangeElement>
}

extension OrderedCollectionDifference: Codable where ChangeElement: Codable {}
```

A `Change` is a single mutating operation, an `OrderedCollectionDifference` is a plurality of such operations that represents a complete transition between two states. Given the interdependence of the changes, `OrderedCollectionDifference` has no mutating members, but it does allow index- and `Slice`-based access to its changes via `Collection` conformance as well as a validating initializer taking a `Collection`.

Fundamentally, there are only two operations that mutate ordered collections, `insert(:,at:)` and `remove(at:)`, but there are benefits from being able to represent other operations such as moves and replacements, especially for UIs that may want to animate a move differently from an `insert`/`remove` pair. These operations are represented using `associatedWith:`. When non-`nil`, they refer to the offset of the counterpart as described in the headerdoc.

### Application of instances of `OrderedCollectionDifference`

``` swift
extension RangeReplaceableCollection {
    /// Applies a difference to a collection.
    ///
    /// - Parameter difference: The difference to be applied.
    ///
    /// - Returns: An instance representing the state of the receiver with the
    ///   difference applied, or `nil` if the difference is incompatible with
    ///   the receiver's state.
    ///
    /// - Complexity: O(*n* + *c*), where *n* is `self.count` and *c* is the
    ///   number of changes contained by the parameter.
    @available(swift, introduced: 5.1)
    public func applying(_ difference: OrderedCollectionDifference<Element>) -> Self?
}
```

Applying a diff to an incompatible base state results in an invalid collection state. `applying(:)` expresses this by returning nil.

## Source compatibility

This proposal is additive and the names of the types it proposes are not likely to already be in wide use, so it does not represent a significant risk to source compatibility.

## Effect on ABI stability

This proposal does not affect ABI stability.

## Effect on API resilience

This feature additive and symbols marked with `@available(swift, introduced: 5.1)` as appropriate.

## Alternatives considered

### `difference(from:,by:)` defined in protocol instead of extension

I'd love to hear feedback on this decision; it was informed by an intention to minimize defects without compromising performance.

The diffing function is defined in an extension because thus far all performance opportunities have been possible to satisfy by adding different required protocol methods that provide functionality such as fast membership testing or identifying the first mismatch between two slices.

Furthermore, making the algorithm that produces the diff immutable creates a guarantee that two diffs constructed from different collections but representing the same state transition will always equate to each other. This would certainly be violated if `difference(from:,by:)` were overridden because of the possibility for the same state transition to be represented in multiple ways by implementations that don't produce [LCS](https://en.wikipedia.org/wiki/Longest_common_subsequence_problem) results.

### Communicating changes via a series of callbacks

Breaking up a transaction into a sequence of imperative events is not very Swifty, and the pattern has proven to be fertile ground for defects.

### More nuanced cases in `OrderedCollectionDifference.Change`

While the tools in this proposal are used to represent a transaction between two states, it's tempting to extend the solution to include the "how" of the transition as well as the "what". Unfortunately implementing such a thing depends on a notion of identity that is different from equality, which is defined differently for every kind of Element that may be used with this API.

Put another way, diffs already lack temporal information (the change order that produced the state transition is almost always irrecoverable), the same is true for the kinds of operations that produced the state transition.

Representing different kinds of operations also introduces a problem where diffs can't be compared; should they be equal if they transition between the same two states, or only if they make that transition in the same way?

Clients are encouraged to infer moves vs independant insert/removes by applying their own notion of identity to their interpretation of the difference.

### `applying(:) throws -> Self` instead of `applying(:) -> Self?`

Assuming the code producing and applying the diff are both under your control, it is always possible to be certain that a difference is safe to apply. The only reason application can fail is when the diff is being applied to an invalid base state.

The existence of a function that returns nil when application fails is a concession to the reality that `OrderedCollectionDifference` may be used as a boundary data type by APIs that wish to respond to bad inputs in a non-fatal manner. Since there's only one reason diff application may fail, the granularity provided by an error type isn't especially useful.

### `Change` using `Index` instead of offset

Because indexes cannot be navigated in the absence of the collection instance that generated them, a diff would be much more limited in usefulness as a boundary type. If indexes are required, they can be rehydrated from the offsets in the presence of the collection(s) to which they belong.

### `OrderedCollection` conformance for `OrderedCollectionDifference`

Because the change offsets refer directly to the resting positions of elements in the base and modified states, the changes represent the same state transition regardless of their order. The purpose of ordering is to optimize for understanding, safety, and/or performance. In fact, this prototype already contains examples of two different equally valid sort orders: 

* The order provided by `for in` is optimized for safe diff application when modifying a compatible base state one element at a time.
* `applying(_:)` uses a different order where `insert` and `remove` instances are interleaved based on their adjusted offsets in the base state.

Both sort orders are "correct" in representing the same state transition.

### `Change` generic on `BaseElement` and `OtherElement` instead of just `Element`

Application of differences would only be possible when both `Element` types were equal, and there would be additional cognitive overhead with comparators with the type `(Element, Other.Element) -> Bool`.

Since the comparator forces both types to be effectively isomorphic, a diff generic over only one type can satisfy the need by mapping one (or both) ordered collections to force their `Element` types to match.

## Intentional omissions:

### Further adoption

This API allows for more interesting functionality that is not included in this proposal.

For example, this propsal could have included a reversed() function on the difference types that would return a new difference that would undo the application of the original.

The lack of additional conveniences and functionality is intentional; the goal of this proposal is to lay the groundwork that such extensions would be built upon.

In the case of `reversed()`, clients of the API in this proposal can use `Collection.map()` to invert the case of each `Change` and feed the result into `OrderedCollectionDifference`:

``` swift
let reversed = diff.map({ (change) -> OrderedCollectionDifference</* your ChangeElement here */>.Change in
    switch change {
    case .insert(offset: let o, element: let e, associatedWith: let a):
        return .remove(offset: o, element: e, associatedWith: a)
    case .remove(offset: let o, element: let e, associatedWith: let a):
        return .insert(offset: o, element: e, associatedWith: a)
    }
})
```

### Initialization conveniences

While the initializer performs as many safety checks as possible given the available context, the creation of `OrderedCollectionDifference` instances via `init()` is a vector for bugs where a difference may not represent a transition between any known states.

For example, removing all remove cases from a difference makes it incompatible with the state that produced it.

This API design encourages safety via the inclusion of map() (which is easy to get right) and the omission of ExpressibleByArrayLiteral from the difference types (which would be easy to get wrong). init() is included for authors of collection-like types, such as reference types that provide a live view into a model that changes over time.

### `mutating apply(:)`

There is no mutating applicator because there is no algorithmic advantage to in-place application.

### `mutating inferringMoves()`

While there is no algorithmic improvement to be had from in-place move inferencing, there may be other savings to be had; we're considering this function for a future proposal, if there is sufficient demand.