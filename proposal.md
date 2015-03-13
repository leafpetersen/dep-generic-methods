# Generic Methods and Functions for Dart


## Contact information

1. **Leaf Petersen**

2. **leafp@google.com**

3. **[https://github.com/leafpetersen/dep-generic-methods](https://github.com/leafpetersen/dep-generic-methods)**

Other stakeholders:

  * Vijay Menon, vsm@google.com

## Summary

Generics in Dart are currently limited to classes only. This imposes significant
limitations on the ability of programmers to write typed Dart code (see the
Motivation section for concrete examples). This document proposes to extend Dart
with a fairly simple system of generic methods and functions. This proposal
would allow programmers to write generic methods, static methods, and top level
functions using the same familiar syntax as is currently used for generic
classes. For example, an Option class providing a typed map method might be
written as follows:

```dart
abstract class Option<T> {
  Option<S> map<S>(S f(T x));
  T get val;
}

class Some<T> extends Option<T> {
  final T _x;
  Some(this._x);
  Some<S> map<S>(S f (T x)) => new Some<S>(f(_x));
  T get val => _x;
}

class None<T> extends Option<T> {
  None();
  Option<S> map<S>(S f(T x)) => new None<S>();
  T get val => throw new Error("No value");
}
```

A generic partial map over lists (written against the current List/Iterable
interface) might then be written and used as follows:

```dart
List<S> mapPartial<T, S>(List<T> l, Option<S> f(T x)) {
     var m = l.map(f).where((x) => x is Some).map((x) => x.val);
     return new List<S>.from(m);
  }
}

List<int> parseInts(List<String> l, Option<int> parseInt(String s)) {
  return mapPartial<int, String>(l, parseInt);
}
```


## Motivation

Requests for generic methods come up frequently.  There's a longstanding feature
request [here](https://code.google.com/p/dart/issues/detail?id=254) with 163
stars.  The topic comes up frequently in discussions.  Some examples follow:

  * **[https://groups.google.com/a/google.com/d/msg/dart-discuss/Vcz4cdEmJrU/wbAOGwz0YGAJ](https://groups.google.com/a/google.com/d/msg/dart-discuss/Vcz4cdEmJrU/wbAOGwz0YGAJ)**
  * **[https://groups.google.com/a/dartlang.org/d/msg/misc/0CWCCE3Iffc/oJlwOxamsMsJ](https://groups.google.com/a/dartlang.org/d/msg/misc/0CWCCE3Iffc/oJlwOxamsMsJ)**
  * **[https://groups.google.com/a/dartlang.org/d/msg/misc/uHPi5fM9sLQ/v2Y4LJcar5YJ](https://groups.google.com/a/dartlang.org/d/msg/misc/uHPi5fM9sLQ/v2Y4LJcar5YJ)**
  * **[https://groups.google.com/a/dartlang.org/d/msg/misc/KqegdaJzm6A/cK2JfwewMMEJ](https://groups.google.com/a/dartlang.org/d/msg/misc/KqegdaJzm6A/cK2JfwewMMEJ)**
  * **[https://groups.google.com/a/dartlang.org/d/msg/misc/hQSxFVhJUx0/xqNCdl7eSREJ](https://groups.google.com/a/dartlang.org/d/msg/misc/hQSxFVhJUx0/xqNCdl7eSREJ)**
  * **[https://groups.google.com/a/dartlang.org/d/msg/misc/-mzUuqxeieU/Gb5dXfGklScJ](https://groups.google.com/a/dartlang.org/d/msg/misc/-mzUuqxeieU/Gb5dXfGklScJ)**
  * **[https://groups.google.com/a/dartlang.org/d/msg/misc/32WdLuwH5nc/Vr0LjcxflD8J](https://groups.google.com/a/dartlang.org/d/msg/misc/32WdLuwH5nc/Vr0LjcxflD8J)**

The lack of generic methods in Dart greatly limits the ability of programmers to
take advantage of both runtime and static type checking, as well as limiting the
effectiveness of tooling.  For example, the map method on the List class is
currently implemented as:

```dart
  Iterable map(f(E element)) => new MappedListIterable(this, f);
```

The inability to provide a generic method parameter to describe the return type
of the mapping function ("f") has the following implications.

1.  The reified runtime type of the resulting Iterable is
    ```Iterable<dynamic>```.  Consequently, checked mode checks cannot be relied on to
    check that the result of a call to map is used appropriately, since Iterable
    is a subtype of any concrete instantation of Iterable.
2.  The static type of the resulting Iterable is
    Iterable<dynamic>. Consequently, the analyzer cannot provide warnings about
    incorrect uses of the result.
3.  The static type of the resulting Iterable is ```Iterable<dynamic>```.
    Consequently tooling cannot in general provide useful completions based on
    the type of the elements of the iterable

Concretely, these three points can be seen in the following code fragment:

```dart
  List<String> l = <String>["hello", "world"];
  Iterable<int> i = l.map((x) => x);
  var m = l.map((x) => x.length);
```

The assignment to "i" disguises a list of strings as a list of integers, but
produces no static warnings (point 2), and succeeds at runtime in checked mode
(point 1).  The expression "i.first" will be treated by tools as having type
"int".  Warnings will be emitted if "i" is used (correctly) as an iterable of
strings, but not if it is used (incorrectly) as an iterable of integers. The
assignment to "m" is valid, but results in an iterable for which tooling
provides no useful completions (Point 3): for example, typing "m.first."
results in no completions for the integer type in the current Dart Editor.

These limitations have proved sufficiently painful in practice that special case
support for the Future.then API call has been added to the Dart Analyzer to
allow it to infer types as if it had the obvious (but unwritable in Dart)
generic type, so that useful warnings and completions can be provided within the
scope of the callback parameter.  This special treatment is not available even
for other core Dart libraries (e.g. List, Iterable, Map etc), much less for user
written code.

The goal of this proposal then is to provide programmers with the linguistic
mechanisms needed to communicate the static type properties of generically
defined methods and functions to the runtime system, the compiler, tooling, and
not least of all other programmers.

## Examples


As a concrete example, consider an immutable binary tree, implemented as a
single node class which holds a key, a value, and optional left and right trees;
and a main tree class which holds a root node (or null if empty).

The node class might be implemented as follows:

```dart
class BinaryTreeNode<K extends Comparable<K>, V> {
  final K _key;
  final V _value;
  final BinaryTreeNode<K, V> _left;
  final BinaryTreeNode<K, V> _right;

  BinaryTreeNode(this._key, this._value, {left = null, right = null}) :
    _left = left, _right = right;

  static BinaryTreeNode<K, V> insertOpt<K, V>(
      BinaryTreeNode<K, V> t, K key, V value) {
    return (t == null) ? new BinaryTreeNode(key, value) : t.insert(key, value);
  }

  BinaryTreeNode<K, V> insert(K key, V value) {
    var c = key.compareTo(_key);
    if (c == 0) return this;
    var _insert = insertOpt<K, V>;
    var left = _left;
    var right = _right;
    if (x < 0) left = _insert(_left, key, value);
    else       right = _insert(_right, key, value);
    return new BinaryTreeNode(_key, _value, left:left, right:right);
  }

  static BinaryTreeNode<K, U> mapOpt<K extends Comparable<K>, V, U>(
      BinaryTreeNode<K, V> t, U f(V x)) {
    return (t == null) ? null : t.map<U>(f);
  }

  BinaryTreeNode<K, U> map<U>(U f(V x)) {
    var _map = mapOpt<K, V, U>;
    return new BinaryTreeNode<K, U>(_key, f(_value),
                                    left:_map(_left, f),
                                    right:_map(_right, f));
  }

  static S foldPreOpt<K extends Comparable<K>, V, S>(
      BinaryTreeNode<K, V> t, S init, S f(V t, S s)) {
    return (t == null) ? init : t.foldPre<S>(init, f);
  }

  S foldPre<S>(S init, S f(V t, S s)) {
    var _fold = foldPreOpt<K, V, S>;
    var s = init;
    s = f(_value, s);
    s = _fold(_left, s, f);
    s = _fold(_right, s, f);
    return s;
  }
}
```

As examples of uses of generic methods, the implementation includes typed map
and pre-order fold operations with the following generic signatures:

```dart
  BinaryTreeNode<K, U> map<U>(U f(V x))
  S foldPre<S>(S init, S f(V t, S s)) {
```

Note that here the "K" and "V" type parameters are bound by the enclosing class,
and describe the types of the keys and the values of the receiver tree.  The "U"
type parameter is bound by the map function, and is in scope in the return type
of map, in its parameter list, and in the scope of its body.  The "U" type
descibes the type of the values of the output tree.  Similarly, the "S" type
parameter to foldPre describes the type of the value computed by the fold, and
is similarly scoped.

The implementation of tree nodes also contains two static helper functions
  factoring out the logic for dealing with possibly null tree nodes.

```dart
  BinaryTreeNode<K, U>
    mapOpt<K extends Comparable<K>, V, U>(BinaryTreeNode<K, V> t, U f(V x));
  static S foldPreOpt<K extends Comparable<K>, V, S>(
    BinaryTreeNode<K, V> t, S init, S f(V t, S s))
```

Note that here that the "K" and "V" type parameters are bound by the functions
themselves, rather than by the enclosing class, reflecting the fact that these
are static functions applicable to any kind of tree node, rather than specific
to one concrete instance.  As with type parameters on a class, type parameters
on generic methods and functions may have type bounds, as shown here with the
type bound on "K" which enforces the property that we may only create trees
using key types which support the "compareTo" method (used in the definition of
"insert").

The mapOpt function contains an example of calling a generic method on an
object:

```dart
  return (t == null) ? null : t.map<U>(f);
```

The method call to "t.map" passes in a type argument "U", and a regular argument
"f".  If the type argument were to be elided, the parameter would be implicitly
instantiated with the type dynamic.

Generic functions and methods can also be specialized to concrete functions and
methods by providing type arguments but no term arguments.  For example, in the
definition of the "map" method, the following code defines "_map" to be the
generic function "mapOpt" with its type arguments specialized to the type
arguments in scope in the "map" function.

```dart
    var _map = mapOpt<K, V, U>;
```

The type arguments may also be elided as follows, in which case they would be
implicitly instantiated to dynamic:

```dart
    var _map = mapOpt;
```

The main binary tree implementation might be written as follows:

```dart
class BinaryTree<K extends Comparable<K>, V> {
  final BinaryTreeNode<K,V> _root;

  BinaryTree._internal(this._root);
  BinaryTree.empty() : this._internal(null);

  BinaryTreeM<K, V> insert(K key, V value) {
    var root = BinaryTreenode.insertOpt<K, V>(_root, key, value);
    return new BinaryTree._internal(root);
  }

  BinaryTree<K, U> map<U>(U f(V x)) {
    var root = BinaryTreeNode.mapOpt<K, V, U>(_root, f);
    return new BinaryTree._internal(root);
  }

  S foldPre<S>(S init, S f(V t, S s)) {
    return BinaryTreeNode.foldPreOpt<K, V, S>(_root, init, f);
  }
}
```

A concrete use of the binary tree implementation might look something like this:

```dart
main() {
  // Create a new Binary Tree mapping ints to Strings
  // sT has type BinaryTree<int, String>
  var sT = new BinaryTree<int, String>();
  // Add some strings
  sT = sT.insert(0, "");
  sT = sT.insert(1, " ");
  sT = sT.insert(2, "  ");
  sT = sT.insert(3, "   ");
  // Map the string length function over the values in sT
  // iT has type BinaryTree<int, int>
  var iT = sT.map<int>((s) => s.length);
  // Compute the sum of the lengths
  var total = sT.foldPre<int>(0, (i, s) => i + s);
}

```

## Proposal

The proposal is to add a simple form of generic methods and functions to Dart.
Specifically, it allows top level functions, local function definitions, static
methods, and instance methods to be parameterized over type variables.  The
function call and method invocation syntax is extended to allow formal type
parameters to be instantiated with their corresponding actual type arguments.
Generic function types are second class in the type system, and do not need to
be reified at runtime.  Consequently: there is no syntax for a generic function
type; type parameters may not be instantiated with generic function types;
functions and methods may not be abstracted over generic functions.
Technically, the proposal is to add prenex, predicative F-bounded quantification
to Dart methods and functions.

Getters and setters may not be generics.  Similarly, constructors, named
constructors and factory constructors may not be generics.

### Syntax

#### Functions

The function declaration syntax is extended with the addition of an optional
list of type parameters, written using the same syntax as is used for class type
parameters ("typeParameters?" in the grammar of the language).  Type parameters
may be written with upper bounds using the "extends type" syntax.  The list of
type parameters immediately follows the function name, enclosed in angle braces:
```methodName<T_0 extends B_0, ..., T_n extends B_n>```.  The scope of the type parameters includes the return
type of the function, the formal parameter list of the function, and the
function body.

Generic function declarations are permitted at the top level, and at local
variable scope.

#### Instance methods

Instance method syntax inherits the function declaration syntax above.  Note
that type parameters bound at the method declaration level may shadow type
parameters bound at the class declaration level.

#### Static methods

Static method syntax inherits the function declaration syntax above.

#### Expressions

The expression syntax is extended with generic instantiation expressions of the
form ```p<S_0, ..., S_n>```, denoting the instantiation of a generic function or
method with the types ```S_0, ..., S_n```, where ```p``` is an identifier, or a
path of the form ```e.identifier``` for some expression ```e```.

This syntax introduces a grammatical ambiguity when it appears in a sequence
context, since (for example) ```[f<int, String>(3)]``` could be parsed either as
a list literal with two elements (```[(f < int), (String > (3))]```) or as a
list literal with one element (```[(f<int, String>)(3))]```).  This ambiguity
can be resolved by preferring to parse an instantiation expression, but allowing
a parse as an instantiation expression only when the next token after the
closing ```>``` is an open parenthesis (```(```) or is otherwise not a valid
first token for an expression production.  This ensures that
```[f<int, String>(3)]``` parses as a single element list, and ```g(f<int,
String>(3))``` parses as an application of ```g``` to a single argument.  The
alternative parses, while in principle valid, are in practice unlikely as
comparison with a type literal is not a commong pattern.

An alternative resolution to the ambiguity is to introduce new tokens for
generic instantiations, such as ```<:``` and ```:>```.  This would result in an
instantiation syntax of ```p<:S_0, ..., S_n:>``` (and possibly a corresponding
change in the generic method declaration syntax).  The lack of symmetry to the
generic class syntax makes this a less preferred choice.

### Static typing

#### Function types

Generic function types do not appear in the external syntax, but are necessary
in the static semantics to account for subtyping of instance methods.

Function types are extended with additional type parameters as
follows:

*```<S_0 extends B_0, ..., S_m extends B_m>(T_0, ..., T_n, [T_{n+1}, ..., T_{n+k}]) -> T```*

and

*```<S_0 extends B_0, ..., S_m extends B_m>(T_0, ..., T_n, {T_{n+1} p_{n+1}, ..., T_{n+k} p_{n+k}}) -> T```*

where the *```S_i```* are type parameters in scope in the rest of the type. 

A function type with a non-empty type parameter list is mal-formed if it appears
as a sub-component of any other type: that is, as an argument to a generic class
or function, as a bound on a type parameter, or as a parameter or return type in
another function type (in the current proposal, there is any case no syntax to
introduce such a type, and so this restriction may be moot).

#### Function subtyping

Two generic function types GT_1 and GT_2 are subtypes if:

1. ```GT_1 = <S_0 extends B_0, ..., S_m extends B_m>FT_1``` where FT_1 is a standard (non-generic) function type
2. ```GT_2 = <T_0 extends C_0, ..., T_m extends C_m>FT_2``` where FT_2 is a standard (non-generic) function type
3. ```C_i <: B_i``` for i in 0...m
4. ```[T_i/S_i]FT_1 <: FT_2```  (that is, FT_1 is a subtype of FT_2, where all occurrences of the S_i in FT_1 are replaced with T_i).


A generic function type GT_1 is a subtype of a standard function type FT_2 if:

1. ```GT_1 = <S_0 extends B_0, ..., S_m extends B_m>FT_1``` where FT_1 is a standard (non-generic) function type
2. ```FT_1 <: FT_2```

> Commentary: This subtyping rule (combined with implicit instantiation) allows
> generic methods to override non-generic methods in compatible ways.  For
> example, a subclass of List which provides a generically typed map function
> would be a valid implementation of the List interface.

A class C declaring a ```call``` method of generic type ```GT``` is considered a
subtype of ```GT```.

#### Declaration typing

Within the scope of a generic function or method, recursive (or mutually
recursive) references to the function or method are typed using the full generic
type: that is, generic functions and methods provide polymorphic recursion.

The definition of method overriding remains unchanged, except indirectly via the
change in the subtype relation.

#### Expression typing

Let ```f``` be a path to a generic function or method with static type ```<T_0
extends B_0, ..., T_m extends B_m>FT```.  Let ```f<S_0, ..., S_n>``` be an
instantiation expression.

It is a static warning if ```m != n```.  It is a static warning if ```S_i <:
B_i``` does not hold for all i.  It is a static warning if any of the ```S_i```
are malformed.

If ```m != n```, then the ```S_i``` are ignored, and the instantiation is
treated as it were replaced by an instantiation with the appropriate number of
arguments, all set to ```dynamic```.

The static type of the instantiation expression ```f<S_0, ..., S_n>``` is
```[S_i/T_i]FT```: that is, ```FT``` with all occurrences of the formal
parameters ```T_i``` replaced by the actual types ```S_i``` (taking into account
the replacement of the actual parameters with ```dynamic``` in the case of an
arity mismatch).

Any use of a generic function or method outside of an instantiation expression
is treated as an implicit instantation with the appropriate number of arguments,
each set to ```dynamic```.

### Dynamic semantics

#### Generic instantiation expressions (functions)

Let ```p<S_0, ..., S_n>``` be a generic instantiation expression where ```p```
is a path to a generic function or static generic method ```f``` (using the same
resolution rules for unqualified identifiers as is done with non-generic
functions) .  It is a checked mode error if the number of generic type
parameters expected by ```f``` is different from ```n```.  In unchecked mode,
any additional type parameters are ignored, and any missing type parameters are
filled in with dynamic.  It is a checked mode error if any of the type
parameters are not subtypes of the bounds on the formal type parameters to
```f``` .

Otherwise, let ```T_0, ..., T_n``` be the formal type parameters for ```f```.
The instantiantation expression evaluates to a function with the same definition
as ```f```, except with each ```S_i``` substituted for the corresponding
```T_i```, using the usual capture avoiding substitution.  It is not guaranteed
that two instantiations of the same generic function with the same type
parameters will return the same object (nor is it forbidden).

#### Generic instantation expressions (method application)

Let ```p<S_0, ..., S_n>(e0, ..., en)``` be an invocation of a generic
instantiation expression where ```p``` is a path which evaluates to a reference
to an instance method of the form ```o.m``` where ```o.m``` resolves to generic
method (using the same method resolution rules as for ordinary methods).  It is
a checked mode error if the number of generic type parameters expected by
```m``` is different from ```n```.  In unchecked mode, any additional type
parameters are ignored, and any missing type parameters are filled in with
dynamic.  It is a checked mode error if any of the type parameters are not
subtypes of the bounds on the formal type parameters to ```m``` .

Otherwise, let ```T_0, ..., T_n``` be the formal type parameters for ```m```.
Evaluation proceeds exactly as if ```m``` were defined as a non-generic method
applied to ```(e0, ..., en)```, with each ```S_i``` substituted for the
corresponding ```T_i``` in the definition of ```m``` using the usual capture
avoiding substitution.

#### Generic instantation expressions (method closurization)

Let ```p<S_0, ..., S_n>``` be a generic instantiation expression in a non-method
invocation context where ```p``` is a path which evaluates to a reference to an
instance method of the form ```o.m``` where ```o.m``` resolves to generic method
(using the same method resolution rules as for ordinary methods).  It is a
checked mode error if the number of generic type parameters expected by ```m```
is different from ```n```.  In unchecked mode, any additional type parameters
are ignored, and any missing type parameters are filled in with dynamic.  It is
a checked mode error if any of the type parameters are not subtypes of the
bounds on the formal type parameters to ```m```.

Otherwise, the closurization of the generic method follows the definition of
closurization of a non-generic method, except that in the bodies of the
extracted closures, the method ```m``` is invoked with ```<S_0, ..., S_n>``` as
type parameters.  As with normal closurization, closurization of the same
generic method on identical objects with equal types must produce results
that compare equal:

Iff ```identical(o1, o2) && S_0 == T_0 && ... && S_n == T_n``` then ```o1.m<S_0, ..., S_n> == o2.m<T_0, ... T_n>```

> Commentary: A more restrictive version of this would require identity on the
> types as well as the receiver.  This seems overly strong, since it is not
> clear that it is possible for the programmer to ensure that this is the case.

#### Implicit generic instantiations (escaping uses)

Let ```p``` be an expression which is not in the context of a generic
instantiation (that is, ```p``` is not immediately syntactically applied to a
list of type arguments).  If ```p``` evaluates to a generic method or function,
then it is treated as an implicit instantiation of the generic method or
function with the appropriate number of type parameters, all set to
```dynamic``` .

> Commentary: Note that combined with the lack of a requirement for
> canonicalization on instantations, this implies that implementations are free
> to implement instantiation in such a way as to make ```identical(f, f)```
> evaluate to false if ```f``` is a generic method.  While not fundamentally
> different from the fact that ```identical(o.m, o.m)``` may also not hold, this
> is admittedly a somewhat surprising property.  Alternatives would be to either
> require canonicalization of instantations (which seems potentially too
> expensive), or to simply forbid escaping uses of generics (restricting
> implicit instantiations to application sites).  Requiring canonicalization
> only for implicit instantiations might also be acceptable.

## Alternatives

### Reification

The choice to make the generic parameters reified in the body of the generic
function/method (as opposed to using them simply for static checking) is
essential for the usefulness of generic methods in Dart, since one of the
principal goals is to support typed programming in Dart as it exists.  The Dart
development model is based around using the runtime type of objects to enable
checked mode checks.  Without reified generic parameters, the map function on
List (for example) cannot produce a result list with the appropriate runtime
type.

### Type argument inference

This proposal does not address the issue of inferring type arguments for generic
methods.  Many other languages (e.g. Swift, C#, Java, Scala) support this, and
it provides great benefits to programmers in reducing the verbosity of code.  I
suspect that for the proposal as written, inferring type arguments for generic
methods should be feasible via an algorithm using an adaptation of the local
type inference algorithm (or the colored local type inference algorithm) that
form the basis of inference in C# and Scala.  However, Dart subtyping introduces
wrinkles, since (for example) the definition of function subtyping would seem to
require assignability constraints rather than just subtyping constraints.  Since
this algorithm would need to be implemented in the JIT, it is important that the
algorithm for computing the candidates and choosing from the candidate sets be
quite efficient.  In the interest of moving forward on the basic functionality
for generic methods, this proposal leaves the question of inference to be
addressed later.

The implicit instantiation semantics whereby elided type arguments are treated
as ```dynamic``` means that adding type argument inference in the future would
either need to be a breaking change, or else would require explicit syntax for
the programmer to request inference.  In the latter case an empty type argument
list could be re-purposed to this as ```f<>(e0, ..., en)``` (a small breaking
change in production mode only), or an additional token could be added to avoid
a breaking change entirely (e.g. ```f<?>(e0, ..., en)```).

### Non-prenex polymorphism (or other fragments of System FSub)

The general subtyping problem for full system FSub is undecideable, but there
are numerous sub-fragments which are decideable.  It is plausible (though would
require verification) that these fragments remain decideable given Dart's
non-standard typing rules.

One potentially interesting point in the space would be to allow predicative but
non-prenex type abstraction: that is, allow generic functions to be passed
around as first class values (as opposed to only allowing instantations of
generic functions to be passed around), while forbidding the instantation of
generic type parameters with universal types.  This would allow functions to be
parameterized over (and to return) generic functions, and data structures to
contain generic functions.

This proposal already allows for non-prenex polymorphism via objects.  An object
instance can contain generic methods, functions can take and return such
objects, and data structures can contain such objects.  On the one hand, it
seems appealing to allow this same functionality to be expressed without
requiring a class to be declared and an instance created.  On the other hand,
since the expressive power is already there, the argument for adding in full
non-prenex polymorphism is weaker.

An advantage of this proposal over allowing non-prenex polymorphism is that as
written, universal types do not need to be reified at runtime, and runtime type
tests do not need to be performed on universal types.  Reifying universal types
does not seem likely to be too onerous (a De Bruijn representation of binders
allows for a clean canonical representation), but type testing is noticeably
more heavyweight once binding and type bounds come into play.  Overall, this
seems like a fairly substantial increase in complexity for a fairly small gain
in expressive power.

This proposal for the most part does not rule out the possibility of adding
non-prenex polymorphism to the language at a later date.  The main point of
future incompatibilty lies in the implicit instantation of generic functions on
escaping uses: that is, treating things like ```var f = g``` where ```T g<T>(T
x) => x``` is a generic function with a single type parameter as implicitly
meaning ```var f = g<dynamic>```.  With non-prenex polymorphism, it would be
valid to assign ```g``` to a variable without implicitly instantiating it, and
then to instantiate the variable ```f``` with a type parameter at a later point
in the program.  Re-interpreting this syntax as such would be a breaking change.
Avoiding the breaking change could be done using syntax to explicitly mark uses
of generic functions as non-instantiating.  Alternatively, it does not seem
unreasonable to modify this proposal to forbid implicit instantiations except in
the context of an immediate invocation, thereby avoiding the potential forwards
incompatibility.

Another possible resolution to this tension would be to choose to reify
universal types at runtime, but not to expose them in the external syntax.  This
would allow the declaration ```var f = g``` from above to be interpreted as
binding ```f``` to a function with runtime type ```<T>(T) -> T``` rather than a
function with runtime type ```dynamic -> dynamic```.  The subtyping rules as
written allow the former type to be used in many (but not all) of the same
contexts as the second: in particular, the former can be used at any type which
is an instantiation of it.  This approach is appealing from a forward
compatibility standpoint, but has the unfortunate consequence of introducing a
fair bit of the complexity of reifying universal types for a fairly small
increase in forwards compatibility.  It is also somewhat harder to explain to
programmers, since it introduces a notion of object type which is otherwise not
explicitly present in the semantics of the language.

Other restrictions of non-prenex polymorphic languages (e.g. rank-2
polymorphism) could also be considered, though it is not clear that there is any
advantage to making these restrictions since full type reconstruction is not a
goal.  These restrictions would require reification of universal types and the
associated runtime type tests, as with the full non-prenex case.


## Implications and limitations

### Integration with the existing language

The design of this proposal was driven by the desire to integrate naturally into
the existing Dart language.  The additional syntax required is small.  Prenex
polymorphism matches the style of top level class genericity already present in
Dart. Implicit instantiation with dynamic is a natural analog of the current
Dart semantics for implicit instantation of generic class types.

The choice of subtyping rules along with implicit instantion allows generic code
to integrate fairly cleanly with non-generic code, and provides a migration path
for APIs to add genericity.  Making a method generic by replacing uses of
dynamic with generic parameters is a non-breaking change from the standpoint of
clients that invoke the method (though it remains a breaking change from the
standpoint of clients that override the method).


###  Limitations and forwards incompatibilities

Some of the implications and limitations of this proposal are discussed in line
with the text.

Issues with grammatical ambiguity are discussed in the section on generic
instantiation syntax above.  Adding the preferred instantation syntax would be a
small breaking change in the language, and would require in some rare cases that
valid programs be parenthesized differently to produce the desired parse.
Tooling should be aware of the potential ambiguity and produce useful error
messages where possible.

Issues with forwards compatibility with respect to more powerful generic systems
and with respect to inference of type arguments are discussed in the previous
section.  For the most part, this proposal is forward compatible with rank-2 and
higher polymorphism, except for the implicit instantation rule for escaping uses
of generic methods.  Implicit instantation for both escaping uses and for
invocations is a point of potential forwards incompatibility for inference of
type arguments, as discussed above.



## Deliverables

### Language specification changes

To be filled out once we're agreed on the details.

### A working implementation

TODO

### Tests

TODO

## Patents rights

TC52, the Ecma technical committee working on evolving the open [Dart standard][], operates under a royalty-free patent policy, [RFPP][] (PDF). This means if the proposal graduates to being sent to TC52, you will have to sign the Ecma TC52 [external contributer form][] and submit it to Ecma.

[tex]: http://www.latex-project.org/
[language spec]: https://www.dartlang.org/docs/spec/
[dart standard]: http://www.ecma-international.org/publications/standards/Ecma-408.htm
[rfpp]: http://www.ecma-international.org/memento/TC52%20policy/Ecma%20Experimental%20TC52%20Royalty-Free%20Patent%20Policy.pdf
[external contributer form]: http://www.ecma-international.org/memento/TC52%20policy/Contribution%20form%20to%20TC52%20Royalty%20Free%20Task%20Group%20as%20a%20non-member.pdf
