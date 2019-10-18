# Sound and Explicit Variance

Author: eernst@google.com

Status: Draft


## CHANGELOG

2019.10.15:
- Created this as a variant of the feature specification that only has use-site invariance (`exactly`): This proposal has all of `out`/`inout`/`in`.


## Summary

This document proposes a specification for sound and explicit management of [variance](https://github.com/dart-lang/language/issues/213) in Dart.

Issues on topics related to this proposal can be found [here](https://github.com/dart-lang/language/issues?utf8=%E2%9C%93&q=is%3Aissue+label%3Avariance+).

Currently, a parameterized class type is covariant in every type parameter. For example, `List<int>` is a subtype of `List<num>` because `int` is a subtype of `num` (so the list type and its type argument "co-vary").

This is sound for all covariant occurrences of such type parameters in the class body (for instance, the getter `first` of a list has return type `E`, which is sound). It is also sound for contravariant occurrences when a sufficiently exact receiver type is known (e.g., for a literal like `<num>[].add(4.2)`, or for a generative constructor `SomeClass<num>.foo(4.2)`).

However, in general, every member access where a covariant type parameter occurs in a non-covariant position may cause a dynamic type error, because the actual type annotation at run time&mdash;say, the type of a parameter of a method&mdash;is a subtype of the one which occurs in the static type.

This proposal introduces explicit variance modifiers for type parameters (aka declaration-site variance), as well as variance modifiers for actual type arguments (aka use-site variance). It includes compile-time restrictions on type declarations and on the use of objects whose static type includes these modifiers, ensuring that the above-mentioned dynamic type errors cannot occur.

In order to ease the transition where types with explicit variance are created and used, this proposal allows for certain subtype relationships where dynamic type checks are still needed when using legacy types (where type parameters are _implicitly_ covariant) to access an object, even in the case where the object has a type with explicit variance. For example, it is allowed to declare `class MyList<out E> implements List<E> {...}`, even though this means that `MyList` has members such as `add` that require dynamic checks and may incur a dynamic type error.


## Syntax

The grammar is adjusted as follows:

```
<typeParameter> ::= // Modified.
    <metadata> <varianceModifier>? <typeIdentifier>
    ('extends' <typeNotVoid>)?

<varianceModifier> ::= // New.
    'out' | 'inout' '?'? | 'in'

<typeArguments> ::= // Modified.
    '<' <typeArgumentList> '>'

<typeArgumentList> ::= // New.
    <typeArgument> (',' <typeArgument>)*

<typeArgument> ::= // New.
    <varianceModifier>? <type> |
    '*'
```


## Static Analysis

This feature allows type parameters to be declared with a _variance modifier_ which is one of `out`, `inout`, or `in`. This indicates that the type parameter is classified as having a specific variance, which in turn implies that the use of such a type parameter is restricted, in return for improved static type safety. This mechanism is also known as declaration-site variance.

The feature also allows type arguments to have a variance modifier, optionally followed by a question mark. This indicates that the enclosing parameterized type is transformed into a supertype or subtype, allowing the instances of this type to have many different values for the given type argument (using `out` or `in`) respectively restricting them to have just one value (using `inout`). This mechanism is also known as use-site variance.

Finally, the rules for other topics like subtyping and for determining the variance of a subterm in a type are adjusted.


### Subtype Rules

The [subtype rule](https://github.com/dart-lang/language/blob/e3010343a8e6f608a831078b0a04d4f1eeca46d4/specification/dartLangSpec.tex#L14845) for interface types that is concerned with the relationship among type arguments ('class covariance') is modified as follows:

In order to conclude that _C&lt;S<sub>1</sub>,... S<sub>s</sub>&gt; <: C&lt;T<sub>1</sub>,... T<sub>s</sub>&gt;_ the current rule requires that _S<sub>j</sub> <: T<sub>j</sub>_ for each _j_ in 1 .. _s_. *This means that, to be a subtype, all actual type arguments must be subtypes.*

The rule is updated as follows in order to take variance modifiers into account:

Some occurrences of variance modifiers are redundant, and they are ignored for the purpose of determining subtype relationships: If a type parameter _X<sub>j</sub>_ has no variance modifier, or it has the variance modifier `out` then if _S<sub>j</sub>_ and/or _T<sub>j</sub>_ is of the form `out U` is treated as `U` (*which means that the rule for the case with no use-site variance modifier applies*). Similarly, if the type parameter _X<sub>j</sub>_ has the variance modifier `inout` then if _S<sub>j</sub>_ and/or _T<sub>j</sub>_ is of the form `inout U` is treated as `U`, and if the corresponding type parameter _X<sub>j</sub>_ has the variance modifier `in`, then if _S<sub>j</sub>_ and/or _T<sub>j</sub>_ is of the form `in U` is treated as `U`.

Let _j_ in 1 .. _s_. If none of _S<sub>j</sub>_ or _T<sub>j</sub>_ have a variance modifier then the following rules apply:

If the corresponding type parameter _X<sub>j</sub>_ has no variance modifier, or it has the variance modifier `out`, we require _S<sub>j</sub> <: T<sub>j</sub>_.

If the corresponding type parameter _X<sub>j</sub>_ has the variance modifier `inout`, we require _S<sub>j</sub> <: T<sub>j</sub>_ as well as _T<sub>j</sub> <: S<sub>j</sub>_.

If the corresponding type parameter _X<sub>j</sub>_ has the variance modifier `in`, we require _T<sub>j</sub> <: S<sub>j</sub>_.

If _T<sub>j</sub>_ is `*` then there are no further requirements. (*So `C<S> <: C<*>` no matter what `S` is, and no matter which variance the type parameter of `C` has.*)

If both _S<sub>j</sub>_ and _T<sub>j</sub>_ have the modifier `inout` then let _S1<sub>j</sub>_ be _S<sub>j</sub>_ except that `inout` has been eliminated, and similarly for _T1<sub>j</sub>_; we then require _S1<sub>j</sub> <: T1<sub>j</sub>_ as well as _T1<sub>j</sub> <: S1<sub>j</sub>_ (*whether or not the corresponding type parameter has a variance modifier, and no matter which one*).

If the type parameter _X<sub>j</sub>_ has the variance modifier `inout`, and we have the relationship that either _T<sub>j</sub>_ is _out S<sub>j</sub>_ or _T<sub>j</sub>_ is _in T<sub>j</sub>_, there are no further requirements. (*So `C<T> <: C<out T>` and `C<T> <: C<in T>` whenever the type parameter is `inout`.*)

If both _S<sub>j</sub>_ and _T<sub>j</sub>_ have the modifier `out` then let _S1<sub>j</sub>_ be _S<sub>j</sub>_ except that `out` has been eliminated, and similarly for _T1<sub>j</sub>_; we then require _S1<sub>j</sub> <: T1<sub>j</sub>_ (*whether or not the corresponding type parameter has a variance modifier, and no matter which one, except that it cannot be `in` because that makes use-site `out` an error.*)

If both _S<sub>j</sub>_ and _T<sub>j</sub>_ have the modifier `in` then let _S1<sub>j</sub>_ be _S<sub>j</sub>_ except that `in` has been eliminated, and similarly for _T1<sub>j</sub>_; we then require _T1<sub>j</sub> <: S1<sub>j</sub>_ (*no matter which variance modifier the corresponding type parameter has, except that it cannot be `out` because that makes use-site `in` an error.*)

If we have the relationship that _S<sub>j</sub>_ is _inout T<sub>j</sub>_, there are no further requirements. (*So `C<inout T> <: C<T>` no matter which variance modifier the type parameter has, including none.*)

*For instance:*

```dart
class C<inout X> {}

// Not runnable code, just examples of subtype relationships:
List<inout num> <: List<num> <: List<Object> <: List<*> <: Object
List<inout List<num>> <: List<List<num>> <: List<Object>
List<inout List<inout num>> <: List<List<inout num>> <: List<List<num>>
List<inout num> <: Iterable<inout num> <: Iterable<num>
List<inout num> <: List<out num> <: List<*>
List<inout num> <: List<in num> <: List<*>
List<out int> <: List<out num> <: List<*>
List<in num> <: List<in int> <: List<*>

// `C<inout T>` is the same as `C<T>`.
C<num> <: C<out num> <: C<out Object> <: C<*>
C<num> <: C<in num> <: C<in int> <: C<*>
C<out int> <: C<out num>
C<in num> <: C<in int>

// But the subtype relation does _not_ include the following:
List<num> <\: List<inout num>
List<inout List<inout num>> <\: List<inout List<num>>
```

*We cannot assign an expression of type `List<num>` to a variable of type `List<inout num>` without a dynamic type check. Similarly, we cannot assign an expression of type `List<inout List<inout num>>` to a variable of type `List<inout List<num>>`. Here is an example illustrating why this is so.*

```dart
main() {
  List<num> xs = <int>[];
  List<inout num> ys = xs; // This must be an error because:
  ys.add(3.41); // Safe, based on the type of `ys`, but it should throw.

  List<inout List<inout num>> zs = [];
  List<inout List<num>> ws = zs; // This must be an error because:
  ws.add(<int>[]); // Safe, based on the type of `ws`, but it should throw.
  zs[0][0] = 4.31; // Safe, based on the type of `zs`, but it should throw.
}
```

*In the above example, an execution that maintains heap soundness would throw already at `ws.add(<int>[])`, because `List<int>` is not a subtype of `List<inout num>`. Otherwise, we can proceed to `zs[0][0] = 4.31` and cause a base level type error, which illustrates that we cannot just ignore the distinction between `List<inout num>` and `List<num>` as a type argument, neither statically nor dynamically. So there is no way we can admit that subtype relationship.*

Finally, the subtype rule that connects a class to its superinterfaces is updated to take use-site variance into account:

Given that _C_ is a class that declares type parameters _X<sub>1</sub> .. X<sub>s</sub>_ such that
_D&lt;T<sub>1</sub> .. T<sub>m</sub>&gt;_ is a direct superinterface of _C_, we conclude that
_C&lt;v<sub>1</sub> S<sub>1</sub> .. v<sub>s</sub> S<sub>s</sub>&gt; <: T_
if
_[v<sub>1</sub> S<sub>1</sub>/X<sub>1</sub> .. v<sub>s</sub> S<sub>s</sub>/X<sub>s</sub>]D&lt;T<sub>1</sub> .. T<sub>m</sub>&gt; <: T_
where _S<sub>j</sub>_ is a type and _v<sub>j</sub>_ is either empty or a variance modifier, for all _j_ in 1 .. _s_.


### Variance Rules

The rules for determining the variance of a position are updated as follows:

We say that a type _S_ occurs in a _covariant position_ in a type _T_ iff one of the following conditions is true:

- _T_ is _S_.

- _T_ is of the form _G&lt;S<sub>1</sub>,... S<sub>n</sub>&gt;_ where _G_ denotes a generic class or type alias and _S_ occurs in a covariant position in _S<sub>j</sub>_ which has no variance modifier, for some _j_ in 1 .. _n_ where the corresponding type parameter of _G_ is covariant; or in a contravariant position where the corresponding type parameter is contravariant.

- _T_ is of the form _G&lt;S<sub>1</sub>,... out S<sub>j</sub> ... S<sub>n</sub>&gt;_ where _G_ denotes a generic class or type alias, _j_ is in 1 .. _n_, and _S_ occurs in a covariant position in _S<sub>j</sub>_.

- _T_ is of the form _G&lt;S<sub>1</sub>,... in S<sub>j</sub> ... S<sub>n</sub>&gt;_ where _G_ denotes a generic class or type alias, _j_ is in 1 .. _n_, and _S_ occurs in a contravariant position in _S<sub>j</sub>_.

- _T_ is of the form _S<sub>0</sub> Function&lt;X<sub>1</sub> extends B<sub>1</sub>, ...>(...)_ where the type parameter list may be omitted, and _S_ occurs in a covariant position in _S<sub>0</sub>_.

- _T_ is of the form _S<sub>0</sub> Function&lt;X<sub>1</sub> extends B<sub>1</sub>, ...&gt;(S<sub>1</sub> x<sub>1</sub>, ... S<sub>k</sub> x<sub>k</sub>, [S<sub>k+1</sub> x<sub>k+1</sub> = d<sub>k+1</sub>, ..., S<sub>n</sub> x<sub>n</sub> = d<sub>n</sub>])_ or of the form _S<sub>0</sub> Function&lt;X<sub>1</sub> extends B<sub>1</sub>, ...&gt; (S<sub>1</sub> x<sub>1</sub>, ... S<sub>k</sub> x<sub>k</sub>, {S<sub>k+1</sub> x<sub>k+1</sub> = d<sub>k+1</sub>, ..., S<sub>n</sub> x<sub>n</sub> = d<sub>n</sub>})_ where the type parameter list and each default value may be omitted, and _S_ occurs in a contravariant position in _S<sub>j</sub>_ for some _j_ in 1 .. _n_.

We say that a type _S_ occurs in a _contravariant position_ in a type _T_ iff one of the following conditions is true:

- _T_ is of the form _G&lt;S<sub>1</sub>, ... S<sub>n</sub>&gt;_ where _G_ denotes a generic class or type alias and _S_ occurs in a contravariant position in _S<sub>j</sub>_ which has no variance modifier, for some _j_ in 1 .. _n_ where the corresponding type parameter of _G_ is covariant; or in a covariant position where the corresponding type parameter is contravariant.

- _T_ is of the form _G&lt;S<sub>1</sub>,... out S<sub>j</sub> ... S<sub>n</sub>&gt;_ where _G_ denotes a generic class or type alias, _j_ is in 1 .. _n_, and _S_ occurs in a contravariant position in _S<sub>j</sub>_.

- _T_ is of the form _G&lt;S<sub>1</sub>,... in S<sub>j</sub> ... S<sub>n</sub>&gt;_ where _G_ denotes a generic class or type alias, _j_ is in 1 .. _n_, and _S_ occurs in a covariant position in _S<sub>j</sub>_.

- _T_ is of the form _S<sub>0</sub> Function&lt;X<sub>1</sub> extends B<sub>1</sub>, ...>(...)_ where the type parameter list may be omitted, and _S_ occurs in a contravariant position in _S<sub>0</sub>_.

- _T_ is of the form _S<sub>0</sub> Function&lt;X<sub>1</sub> extends B<sub>1</sub>, ...&gt;(S<sub>1</sub> x<sub>1</sub>, ... S<sub>k</sub> x<sub>k</sub>, [S<sub>k+1</sub> x<sub>k+1</sub> = d<sub>k+1</sub>, ... S<sub>n</sub> x<sub>n</sub> = d<sub>n</sub>])_ or of the form _S<sub>0</sub> Function&lt;X<sub>1</sub> extends B<sub>1</sub>, ...&gt;(S<sub>1</sub> x<sub>1</sub>, ... S<sub>k</sub> x<sub>k</sub>, {S<sub>k+1</sub> x<sub>k+1</sub> = d<sub>k+1</sub>, ... S<sub>n</sub> x<sub>n</sub> = d<sub>n</sub>})_ where the type parameter list and each default value may be omitted, and _S_ occurs in a covariant position in _S<sub>j</sub>_ for some _j_ in 1 .. _n_.

We say that a type _S_ occurs in an _invariant position_ in a type _T_ iff one of the following conditions is true:

- _T_ is of the form _G&lt;S<sub>1</sub>, ... S<sub>n</sub>&gt;_ where _G_ denotes a generic class or type alias, and _S_ occurs in an invariant position in _S<sub>j</sub>_ for some _j_ in 1 .. _n_; or _S_ occurs (in any position) in _S<sub>j</sub>_, and the corresponding type parameter of _G_ is invariant.

- _T_ is of the form _G&lt;S<sub>1</sub>,... inout S<sub>j</sub> ... S<sub>n</sub>&gt;_ where _G_ denotes a generic class or type alias, _j_ is in 1 .. _n_, and _S_ occurs in _S<sub>j</sub>_ (in any position).

- _T_ is of the form _S<sub>0</sub> Function&lt;X<sub>1</sub> extends B<sub>1</sub>, ... X<sub>m</sub> extends B<sub>m</sub>&gt;(S<sub>1</sub> x<sub>1</sub>, ... S<sub>k</sub> x<sub>k</sub>, [S<sub<k+1</sub> x<sub>k+1</sub> = d<sub>k+1</sub>, ... S<sub>n</sub> x<sub>n</sub> = d<sub>n</sub>])_ or of the form _S<sub>0</sub> Function&lt;X<sub>1</sub> extends B<sub>1</sub>, ... X<sub>m</sub> extends B<sub>m</sub>&gt;(S<sub>1</sub> x<sub>1</sub>, ... S<sub>k</sub> x<sub>k</sub>, {S<sub>k+1</sub> x<sub>k+1</sub> = d<sub>k+1</sub>, ... S<sub>n</sub> x<sub>n</sub> = d<sub>n</sub>})_ where the type parameter list and each default value may be omitted, and _S_ occurs in an invariant position in _S<sub>j</sub>_ for some _j_ in 0 .. _n_, or _S_ occurs (in any position) in _B<sub>i</sub>_ for some _i_ in 1 .. _m_.

It is a compile-time error if a type parameter declared by a static extension has a variance modifier.

*Variance is not relevant to static extensions, because there is no notion of subsumption. Each usage will be a single call site, and the value of every type argument associated with an extension method invocation is statically known at the call site.*

It is a compile-time error if a type parameter _X_ declared by a type alias _F_ has a variance modifier, unless it is `inout` and _X_ occurs in an invariant position on the right hand side of _F_ (*for an old-style type alias, rewrite it to the form using `=` and then check*), or it occurs both in a covariant and a contravariant position; or unless it is `out` and the right hand side of the type alias has covariant occurrences of _X_ and no other occurrences _X_; or unless it is `in` and the right hand side of the type alias has contravariant occurrences of _X_ and no other occurrences of _X_.

*The variance for each type parameter of a type alias is restricted based on the body of the type alias. Explicit variance modifiers can only be used to document how the type parameter is used on the right hand side.*

We say that a type parameter _X_ of a type alias _F_ _is covariant/invariant/contravariant_ if it has the variance modifier `out`/`inout`/`in`, respectively. We say that it is _covariant/contravariant_ if it has no variance modifier, and it occurs only covariantly/contravariantly, respectively, on the right hand side of `=` in the type alias. Otherwise (*when _X_ has no modifier, but occurs invariantly or both covariantly and contravariantly*), we say that _X_ _is invariant_.

Let _D_ be the declaration of a class or mixin, and let _X_ be a type parameter declared by _D_.

We say that _X_ _is covariant_ if it has no variance modifier or it has the variance modifier `out`; that it _is invariant_ if it has the variance modifier `inout`; and that it _is contravariant_ if it has the variance modifier `in`.

If _X_ has the variance modifier `out` then it is a compile-time error for _X_ to occur in a non-covariant position in a member signature in the body of _D_, except that it is not an error if it occurs in a covariant position in the type annotation of a formal parameter that is covariant by declaration (*note that this is a contravariant position in the member signature as a whole*). *For instance, _X_ can not be the type of a method parameter (unless marked `covariant`), and it can not be the bound of a type parameter of a generic method.*

If _X_ has the variance modifier `in` then it is a compile-time error for _X_ to occur in a non-contravariant position in a member signature in the body of _D_, except that it is not an error if it occurs in a contravariant position in the type of a formal parameter that is covariant by declaration. *For instance, _X_ can not be the return type of a method or getter, and it can not be the bound of a type parameter of a generic method.*

*If _X_ has the variance modifier `inout` then there are no variance related restrictions on the positions where it can occur.*

*For superinterfaces we need slightly stronger rules than the ones that apply for types in the body of a type declaration.*

Let _D_ be a class or mixin declaration, let _S_ be a direct superinterface of _D_, and let _X_ be a type parameter declared by _D_.

It is a compile-time error if _X_ has no variance modifier and _X_ occurs in an actual type argument in _S_ such that the corresponding type parameter has a variance modifier. It is a compile-time error if _X_ has the modifier `out`, and _X_ occurs in a non-covariant position in _S_. It is a compile-time error if _X_ has the variance modifier `in`, and _X_ occurs in a non-contravariant position in _S_.

*A type parameter with variance modifier `inout` can occur in any position in a superinterface, and other variance modifiers have constraints such that if we consider type arguments _Args1_ and _Args2_ passed to _D_ such that the former produces a subtype, then we also have _S1 <: S2_ where _S1_ and _S2_ are the corresponding instantiations of _S_.*

```dart
class A<out X, inout Y, in Z> {}
class B<out U, inout V, in W> implements
    A<U Function(W), V Function(V), W Function(V)> {}

// B<int, String, num> <: B<num, String, int>, and hence
// A<int Function(num), String Function(String), num Function(String)> <:
// A<num Function(int), String Function(String), int Function(String)>.
```

*But a type parameter without a variance modifier can not be used in an actual type argument for a parameter with a variance modifier, not even when that modifier is `out`. The reason for this is that it would allow a subtype to introduce the potential for dynamic errors with a member which is in the interface of the supertype and considered safe.*

```dart
// `A` is subject to strict checks concerning the use of `X`.
abstract class A<out X> {
  Object foo();
}

// `B` would relax the checks on the use of `X`.
class B<X> extends A<X> {
  // The following declaration would be an error with `class B<out X>`,
  // so we do not allow it in a subtype of `class A<out X>`.
  void Function(X) foo() => (X x) {};
}
```

*Note that this is a pragmatic design choice. It would be possible to allow classes like `B` to exist, if it turns out to ease migration of existing code.*

*It _is_ allowed to create the opposite relationship, which is surely helpful during migration:*

```dart
class A<X> {
  void foo(X x) {}
}

class B<out X> extends A<X> {}

main() {
  B<num> myB = B<int>();
  ...
  myB.foo(42.1);
}
```

*In this situation, the invocation `myB.foo(42.1)` is subject to a dynamic type check (and it will fail if `myB` is still a `B<int>` when that invocation takes place). But it is statically known at the call site that `foo` has this property for any subtype of `A`, so we can deal with the situation statically, e.g., via a lint.*

*An upcast (like `(myB as A<num>).foo()`) could be used to silence any diagnostic messages, so a strict rule whereby a member access like `myB.foo(42.1)` is a compile-time error may not be very helpful in practice.*

*Note that the class `B` _can_ be written in such a way that the potential dynamic type error is eliminated:*

```dart
class A<X> {
  void foo(X x) {}
}

class B<out X> extends A<X> {
  void foo(Object? o) {...}
}
```

*In this case an invocation of `foo` on a `B` will never incur a dynamic error due to the run-time type of its argument, which might be a useful migration strategy in the case where a lot of call sites are flagged by a lint, especially when `B.foo` is genuinely able to perform its task with objects whose type is not `X`.*

*However, in the more likely case where `foo` does require an argument of type `X`, we do not wish to insist that developers declare an apparently safe member signature like `void foo(Object?)`, and then throw an exception in the body of `foo`. That would just eliminate some compile-time errors at call sites which are actually justified.*

*If such a method needs to be implemented, the modifier `covariant` must be used, in order to avoid the compile-time error for member signatures involving an `out` type parameter:*

```dart
class A<X> {
  void foo(X x) {}
}

class B<out X> extends A<X> { // or `implements`.
  void foo(covariant X x) {...}
}
```


### Member signatures with Widening Use-site Variance

!!!HERE!!!

*For example:*

```dart
class B<inout X> {
  B<B<X>> m() => B<B<X>>();
}

main() {
  B<int> bi = B<int>();
  B<out num> bn = bi;
  var bi2 = bi.m(); // `bi2` has type `B<B<int>>`.
  var bn2 = bn.m(); // `bn2` has type `B<out B<out num>>`.
}
```


### Member Signatures with Conditional Variance

In addition to the plain variance modifiers, it is possible to use the conditional form `inout?`. The conditional variance modifier has no effect on the variance of a position in a type, but in return it will be erased from types where the corresponding typing is not guaranteed to be sound.

*A major point of having `inout?` in a member signature is that it enables us to preserve a more precise return type from an instance member invocation, in cases where use-site variance (that is, `inout`) has been used to make the receiver type more precise.*

It is a compile-time error if a conditional variance modifier occurs anywhere else than in the signature of an instance member of a class. If `inout?` occurs on a type argument _inout? A_ such that the corresponding type parameter has the variance modifier `inout`, that type argument is treated as _A_ (*that is, a redundant `inout?` is ignored, just like a redundant `inout`*).

During static analysis of the body of a member `m`, the signature of `m` is taken to have the variance modifier `inout` wherever it is specified to be `inout?`.

*This ensures soundness for the special case where `inout?` in the signature of `m` is transformed into `inout` in an expression where `m` is invoked.*

For soundness, each occurrence of the modifier `inout?` in a member signature is transformed into `inout` or eliminated when computing the static type and type checking each member access (*such as an invocation of a method, getter, or setter*), based on the type of the receiver.

Let _D_ be the declaration of a generic class or mixin named _N_ and let _X<sub>1</sub> .. X<sub>k</sub>_ be the type parameters declared by _D_. Let _T_ be a parameterized type which applies _N_ to a list of actual type arguments _S<sub>1</sub> .. S<sub>k</sub>_ in a context where the name _N_ denotes the declaration _D_, and consider the situation where a member access to a member `m` in the interface of _T_ is performed (*for instance, we consider `e.m()`, where `e` has type _T_*). Let _S1<sub>1</sub> .. S1<sub>k</sub>_ be types such that for each _j_ in 1 .. _k_, _S<sub>j</sub>_ is of the form _<varianceModifier> S1<sub>j</sub>_. (*In other words, _S1_ is just _S_ where variance modifiers have been stripped off.*)

Let _s_ be the member signature of `m` from the interface of _D_, and _s1_ be _[S1<sub>1</sub>/X<sub>1</sub> .. S1<sub>k</sub>/X<sub>k</sub>]s_.

*This is the "raw" version of the statically known type of `m`; to obtain a sound typing we need one more step where occurrences of `inout?` are transformed into `inout` or eliminated.*

For each _j_ in 1 .. _k_, if _X<sub>j</sub>_ does not have the variance modifier `inout`, and _S<sub>j</sub>_ does not have the modifier `inout`, then for each occurrence of `inout?` on a type that contains _X<sub>j</sub>_ in _s_, the corresponding occurrence of `inout?` in _s1_ is eliminated. All remaining occurrences of `inout?` in _s1_ are replaced by `inout`, yielding the final member signature _s2_. The static analysis of the member access, including its static type, is then computed based on _s2_.

*For example:*

```dart
class A<X> {
  List<inout? List<inout? X>> get g => [];
}

main() {
  A<inout int> ai = A<int>();
  A<num> an = ai;
  var xsi = ai.g; // `xsi` has type `List<inout List<inout int>>`.
  xsi.add(<int>[]); // Statically safe.
  var xsn = an.g; // `xsn` has type `List<List<num>>`.
  xsn.add(<double>[]); // No compile-time error, but dynamic check, will throw.
}
```

*This example illustrates why the ability to have `inout?` in a member signature helps improving the static typing: The declaration in class `A` ensures that `g` actually returns a `List<inout List<inout X>>`. In the situation where the value of `X` is known at the call site to be a specific type `T`, this allows the returned result to be typed `List<inout List<inout T>>`, which in turn makes the usage of `add` and similar members statically safe.*


### Expressions

It is a compile-time error for an instance creation expression or a collection literal to pass a type argument with a variance modifier. It is a compile-time error to pass an actual type argument to a generic function invocation which has a variance modifier.

```dart
class A<inout X> {}

main() {
  var xs = <inout num>[]; // Error.
  var ys = <List<inout num>>[]; // OK.
  var a = A<out String>(); // Error.
  A<in String> a2 = A(); // OK.

  void f<X>(X x) => print(x);
  f<inout int>(42); // Error.
}
```

*We could say that the list of "type arguments" passed to a constructor invocation or a literal collection contains types, not type arguments; and only type arguments can have a variance modifier. However, those types may themselves receive type arguments, and they can have variance modifiers as needed.*

The static type of an instance creation expression that invokes a generative constructor of a generic class `C` with type arguments `T1, ... Tk` is `C<inout T1, ..., inout Tk>`.

The static type of a list literal receiving type argument `T` is `List<inout T>`; the static type of a set literal receiving type argument `T` is `Set<inout T>`; and the static type of a map literal receiving type arguments `K` and `V` is `Map<inout K, inout V>`.

*It cannot be assumed that a similar relationship exists for regular invocations, say, of a generic function or a factory constructor, so it is an error for an actual type argument to have a variance modifier.*

```dart
class C<X> {
  C();
  factory C.named() => C<Never>();
}

main() {
  // OK, but the static and dynamic type of `c` is `C<int>`, not `C<inout int>`:
  var c = C<int>.named();

  // Error, because it is misleading:
  var c2 = C<inout int>.named();
}
```

*Note that the use of `inout` even for a type argument where the corresponding type parameter _X_ is marked `out` or `in` can be useful: The members declared in the type that receives this type argument must in general be sound with respect to _X_, but there may be some member signatures inherited from a supertype where some type parameters have no variance modifier, and the use of `inout` will then provide a guarantee against dynamic type errors which does not otherwise exist:*

```dart
class A<X> {
  void foo(X x) {}
}

class B<out X> extends A<X> {}

class C<out Y> {
  List<inout? Y> get bar => [];
}

main() {
  B<inout num> b = B();
  b.foo(17.9); // Statically safe.

  C<inout num> c = C();
  c.bar.add(179); // Statically safe, even though `c.bar` has a legacy type.
}
```

*Note that there is no way to make it statically safe to pass an actual argument to a covariant formal parameter of a given member `m`. Any receiver may have a dynamic type which is a proper subtype of the statically known type, and it may have an overriding declaration of `m` that makes the parameter covariant. So, by design, a modular static analysis cannot guarantee that any given invocation will not cause a dynamic error due to a dynamic type check for a covariant parameter.*


## Dynamic Semantics

Every instance of a generic class has a dynamic type where every type argument has the modifier `inout`.

*Note that this only applies at the top level in the dynamic type of the object. It may or may not have variance modifiers on type arguments of type arguments.*

```dart
main() {
  var xs = <List<num>>[]; // Dynamic type is `List<inout List<num>>`.
  var ys = <List<inout num>>[]; // `List<inout List<inout num>>`.
}
```

The dynamic representation of generic class types include information about whether a given actual type argument has a certain variance modifier or not.

*This is required for soundness.*

```dart
main() {
  dynamic xs = <List<inout num>>[];
  xs.add(<int>[]); // Must throw, hence `inout` must be known at run time.
}
```


## Migration

This proposal supports migration of code using dynamically checked covariance to code where some explicit variance modifiers are used, thus eliminating the potential for some dynamic type errors. There are two main scenarios.

Let _legacy class_ denote a generic class that has one or more type parameters with no variance modifiers.

If a new class _A_ has no superinterface relationship with any legacy class (directly or indirectly) then all non-dynamic member accesses to instances of _A_ and its subtypes will be statically safe.

*In other words, if the plan is to use explicit variance only with type declarations that are not "connected to" unsoundly covariant type parameters then there is no migration.*

However, there is a need for migration support in the case where an existing legacy class _B_ is modified such that an explicit variance modifier is added to one or more of its type parameters.

In particular, an existing subtype _C_ of _B_ must now add variance modifiers in order to remain error free, and this may conflict with the existing member signatures of _C_:

```dart
// Before the update.
class B<X> {}
class C<X> implements B<X> {
  void f(X x) {}
}

// After the update of `B`.
class B<out X> {}
class C<X> implements B<X> { // Error.
  void f(X x) {} // If we just make it `C<out X>` then this is an error.
}

// Adjusting `C` to eliminate the errors.
class B<out X> {}
class C<out X> implements B<X> {
  void f(covariant X x) {}
}
```

This approach can be used in a scenario where all parts of the program are migrated to the new language level where explicit variance is supported.

In the other scenario, some libraries will opt in using a suitable language level, and others will not.

If a library _L1_ is at a language level where explicit variance is not supported (so it is 'opted out') then code in an 'opted in' library _L2_ is seen from _L1_ as erased, in the sense that (1) the variance modifiers `out` and `inout` are ignored, and `inout` in types is ignored, and (2) it is a compile-time error to pass a type argument `T` to a type parameter with variance modifier `in`, unless `T` is a top type; (3) any type argument `T` passed to an `in` type parameter in opted-in code is seen in opted-out code as `Object?`.

Conversely, declarations in _L1_ (opted out) is seen from _L2_ (opted in) without changes. So class type parameters declared in _L1_ are considered to be unsoundly covariant by both opted in and opted out code, and similarly for type aliases used to declare function types. Types of entities exported from _L1_ to _L2_ are seen as erased (which matters when _L1_ imports entities from some other opted-in library).

Reification of `inout` on type parameters is required for a sound semantics, but during a transitional period it could be considered as a static-only attribute, thus allowing for soundness violations of this property at run time, and only enforcing it for programs with no opted out code.