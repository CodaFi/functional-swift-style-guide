#Functional Swift - A Style Guide

##Background

Functional Swift is a proper subset of Swift and its features that encourages [immutability](http://en.wikipedia.org/wiki/Immutable_object), [recursion](http://en.wikipedia.org/wiki/Recursion#Recursion_in_computer_science), and [higher types](http://en.wikipedia.org/wiki/Kind_(type_theory)) instead of mutability, loops, and subtyping respectively.  The end goal is to approach the cleanliness and readability of Declarative Programming proper while still maintaining the simplicity and semantics of Swift.

*Note* This document is not a sample of best practices, nor will adhering to it necessarily the most efficient or practical code.  It merely serves as an exploration of pure declarative programming in an imperative language.

##Restrictions

For, do, and while loops are strictly forbidden.  What you cannot express recursively, you should not express at all.

**For Example**

```
public func foldl<A, B>(f: (B, A) -> B) -> B -> [A] -> B {
	return { z in { l in
		switch destruct(l) {
			case .Empty:
				return z
			case .Destructure(let x, let xs):
				return foldl(f)(f(z, x))(xs)
		}
	} }
}
```

Assignment is restricted to `let` bindings and [monadic extraction]() (prefix `!`) in `do_` blocks. 

**For Example**

```
public func finally<A, B>(a : IO<A>)(then : IO<B>) -> IO<A> {
	return mask({ (let restore : IO<A> -> IO<A>) -> IO<A> in
		return do_ { () -> A in
			let r = !onException(restore(a))(what: then)
			let b = !then
			return r
		}
	})
}
```

All classes should derive a [kind class](https://github.com/typelift/Basis/blob/master/Basis/Kinds.swift) (`K0-K10`), then be declared `final` so as to prevent further subtyping.

**For Example**

```
public final class Either<A, B> : K2<A, B> {
```


All higher-kinded classes that can, should provide a destructuring function named `destruct()` that returns an enum so as to facilitate switch-case usage.

**For Example**

```
public enum EitherD<A, B> {
	case Left(Box<A>)
	case Right(Box<B>)
}

//...

public func destruct() -> EitherD<A, B> {
	if lVal != nil {
		return .Left(Box(lVal!))
	}
	return .Right(Box(rVal!))
}
```

Leave the interface inside classes as minimal as possible.  Prefer functions that take that class as their first parameter (CoreFoundation style) to encourage composition.

**For Example**

```
public class Chan<A> : K1<A> {
	init(read : MVar<MVar<ChItem<A>>>, write: MVar<MVar<ChItem<A>>>) {}
}

public func newChan<A>() -> IO<Chan<A>> {}
public func writeChan<A>(c : Chan<A>)(x : A) -> IO<()> {}
public func readChan<A>(c : Chan<A>) -> IO<A> {}
public func dupChan<A>(c : Chan<A>) -> IO<Chan<A>> {}

```

Never return the same object from a function without performing some kind of evaluation (notable exception: `id`).

##Whitespace

Swift code shall use tabs.  The only exception is diagrams in comments, which must use spaces to guarantee correct formatting.  Each opening brace introduces another tab-level of indentation.  Each closing brace removes one tab-level of indentation.  Braces appear on the same line as the statements they accompany.  

**For Example**

```
/// Functors map the functions and objects in one set to a different set of functions and objects.
/// This is represented by any structure parametrized by some set A, and a function `fmap` that 
/// takes objects in A to objects in B and wraps the result back in a functor.  `fmap` respects the
/// following commutative diagram:
///
///               F(A)
///                 •
///                / \
///               /   \
///              /     \
///     fmap f  /       \  fmap (f • g)
///            /         \
///           •-----------•
///       F(B)              F(C)
///               fmap g
///
/// Formally, a Functor is a mapping between Categories, but we have to restrict ourselves to the
/// Category of Swift Types (S), so in practice a Functor is just an Endofunctor.  Functors are
/// often described as "Things that can be mapped over".
public protocol Functor {
	/// Type of Source Objects
	typealias A
	/// Type of Target Objects
	typealias B
	
	/// Type of our Functor
	typealias FA = K1<A>
	
	/// Type of a target Functor
	typealias FB = K1<B>
	
	/// Map "inside" a Functor.
	///
	/// F on our diagram.
	class func fmap(A -> B) -> FA -> FB
	func <%>(A -> B, FA) -> FB

	/// Constant Replace | Replaces all values in the target Functor with a singular constant value
	/// from the source Functor.
	///
	/// Default definition: 
	///		`curry(<%>) • const`
	func <%(A, FB) -> FA
}
```

Longer array and dictionary literals may appear on multiple lines.  If statements should have a single space between the if and the case, and the case and the opening brace.  Else statements appear on the same line as the closing brace of the if statement they accompany.

**For Example**

```
public func destruct<T>(l : [T]) -> ArrayD<T> {
	if l.count == 0 {
		return .Empty
	} else if l.count == 1 {
		return .Destructure(l[0], [])
	}
	let hd = l[0]
	let tl = Array<T>(l[1..<l.count])
	return .Destructure(hd, tl)
}
```

##Functions

Function names should describe exactly what a function does, yet still be concise.  Abbreviations for common words (l for left, m for monad, etc.) are encouraged, as long as they are used consistently.  

A function should always be written in curried form and without argument labels if possible.  Higher order parameters of arity 3 mean there should be a separate overloading for that function to allows operators (which are uncurried by default) to be passed in.  Strive for point-free code.

**For Example**

```
public func foldr<A, B>(k: A -> B -> B) -> B -> [A] -> B {
	return { z in { l in
		switch destruct(l) {
			case .Empty:
				return z
			case .Destructure(let x, let xs):
				return k(x)(foldr(k)(z)(xs))
		}
	} }
}
```

Operators should be accompanied by functions that are fully curried and perform the same operations.  Operators are, in general, the infix form of functions.  They are not to be defined on a whim.  Operators should be easily reachable on a Mac's keyboard.

**For Example**

```
/// A type-restricted version of const.  In cases of typing ambiguity, using this function forces
/// its first argument to resolve to the type of the second argument.
public func asTypeOf<A>(x : A) -> A -> A {
	return const(x)
}

/// "As Type Of" | A type-restricted version of const.  In cases of typing ambiguity, using this 
/// function forces its first argument to resolve to the type of the second argument.  
///
/// Composed because it is the face one makes when having to tell the typechecker how to do its job.
public func >=<<A>(x : A, y : A) -> A {
	return asTypeOf(x)(y)
}
```

##Data Types

Data types should be classes that derive a kind class, then should be final.  Recursive datatypes may require `Box<T>`s or `@autoclosure`s.  A datatype's members are always private, and let-bound whenever possible.  Common operators that act on these private variables should remain in the same file as the data type.

**For Example**

```
public final class Version : K0 {
	public let versionBranch : [Int]
	public let versionTags : [String]

	public init(_ versionBranch : [Int], _ versionTags : [String]) {
		self.versionBranch = versionBranch
		self.versionTags = versionTags
	}
}
```

##Protocols

Protocols are weakened typeclasses.  Provide all functions that return higher-kinded types with the appropriate typealiases to properly restrict implementations.  Default implementations are not possible, but whenever it is appropriate, provide a comment showing the default implementation, or provide a method that returns the default definition for that function.  Infix operators should always call their normal form counterparts, not the other way around.

**For Example**

```
public protocol Functor {
	/// Type of Source Objects
	typealias A
	/// Type of Target Objects
	typealias B

	/// Type of our Functor
	typealias FA = K1<A>

	/// Type of a target Functor
	typealias FB = K1<B>

	/// Map "inside" a Functor.
	///
	/// F on our diagram.
	class func fmap(A -> B) -> FA -> FB
	func <%>(A -> B, FA) -> FB

	/// Constant Replace | Replaces all values in the target Functor with a singular constant value
	/// from the source Functor.
	func <^(A, FB) -> FA
}

/// Eases writing a definition for Constant Replace.  Hand it an fmap, and x in B, and a source.
public func defaultReplace<A, B, FA : Functor, FB : Functor>(fmap : (A -> B) -> FA -> FB)(x : B)(f : FA) -> FB {
	return (fmap • const)(x)(f)
}
```

##Bodies

Function bodies begin one line after and one tab-level from their surrounding function or closure brace.  Function bodies may only contain let bindings, returns, switches, or simple if-else statements.  Functions must return a value.  For functions that perform side effects, that value shall be wrapped in an IO monad.  For functions that perform only side effects, a result of type `IO<()>` shall always be returned.

**For Example**

```
/// Exits with error code 0 (success).
public func exitSuccess<A>() -> IO<A> {
	return exitWith(.ExitSuccess)
}
```

##Other

Type annotations include a space between the parameter name and the colon, and the colon and the type.  Function applications do not include a space between the parameter name and the colon, but do include it between the colon and the type.

**For Example**

```
public func until<A>(p : A -> Bool)(f : A -> A)(x : A) -> A {
	if p(x) {
		return x
	}
	return until(p)(f: f)(x: f(x))
}
```

Generic typealiases are forbidden by the language, so typealiases not required to implement protocols are forbidden.  Functions should never expose typealias'd types, and should instead expose the fully expanded type to the user.  

Non-total functions are allowed, but discouraged and should always be documented as such.  Generally, prefer types to enforce invariants.



