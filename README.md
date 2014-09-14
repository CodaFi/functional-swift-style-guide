#Functional Swift - A Style Guide

##Background

Functional Swift is a proper subset of Swift and its features that encourages immutability, recursion, and higher types instead of mutability, loops, and sub typing respectively.  The end goal is to approach the cleanliness and readability of Haskell while still maintaining the simplicity and semantics of Swift.

##Restrictions

For, do, and while loops are strictly forbidden.  

Assignment is restricted to monadic extraction (`<-`) in `do_` blocks and let statements. 

```
public func foldl1<A>(f: A -> A -> A)(xs0: [A]) -> A {
    let hd = xs0[0]
    let tl = Array<A>(xs0[1..<xs0.count])
    return foldl(f)(z: hd)(lst: tl)
}
```

All classes should derive a kind class (`K0-K10`), then be declared `final` so as to prevent further subtyping.

```
public final class Either<A, B> : K2<A, B> {
```

All higher-kinded classes that can, should provide a destructuring function named `destruct()` that returns an enum so as to facilitate switch-case usage.

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

Swift code shall use tabs.  The only exception is diagrams in comments, which must use spaces to guarantee correct formatting.  Each opening brace introduces another tab-level of indentation.  Each closing brace removes one tab-level of indentation.  Braces appear on the same line as the statements they accompany.  Longer array and dictionary literals may appear on multiple lines.  If statements should have a single space between the if and the case, and the case and the opening brace.  Else statements appear on the same line as the closing brace of the if statement they accompany.

##Functions

Function names should describe exactly what a function does, yet still be concise.  Abbreviations for common words (l for left, m for monad, etc.) are encouraged, as long as they are used consistently.  

A function should always be written in curried form.  Higher order parameters of arity 3 mean there should be a separate overloading for that function to allows operators (which are uncurried by default) to be passed in.

```
public func foldr<A, B>(k: A -> B -> B)(z: B)(lst: [A]) -> B {
	switch lst.destruct() {
		case .Empty:
			return z
		case .Destructure(let x, let xs):
			return k(x)(foldr(k)(z: z)(lst: xs))
	}
}

public func foldr<A, B>(k: (A, B) -> B)(z: B)(lst: [A]) -> B {
	switch lst.destruct() {
		case .Empty:
			return z
		case .Destructure(let x, let xs):
			return k(x, foldr(k)(z: z)(lst: xs))
	}
}
```

Operators should be accompanied by functions that are fully curried and perform the same operations.  Operators are, in general, the infix form of functions.  They are not to be defined on a whim.  Operators should be easily reachable on a Mac's keyboard.

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

Data types should be classes that derive a kind class, then should be final.  Recursive datatypes may require `Box<T>`s, but destructured forms should try to unwrap the box.  A datatype's members are always private, and let-bound whenever possible.  Common operators that act on these private variables should remain in the same file as the data type.

##Protocols

Protocols are weakened typeclasses.  Provide all functions that return higher-kinded types with the appropriate typealiases to properly restrict implementations.  Default implementations are not possible, but whenever it is appropriate, provide a comment showing the default implementation, or provide a method that returns the default definition for that function.  Infix operators should always call their normal form counterparts, not the other way around.

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

##Other

Type annotations include a space between the parameter name and the colon, and the colon and the type.  Function applications do not include a space between the parameter name and the colon, but do include it between the colon and the type.

```
public func until<A>(p : A -> Bool)(f : A -> A)(x : A) -> A {
	if p(x) {
		return x
	}
	return until(p)(f: f)(x: f(x))
}
```

Higher-kinded typealiases are forbidden by the language, so typealiases not required to implement protocols are forbidden.  Functions should never expose typealias'd types, and should instead expose the fully expanded type to the user.  

Non-total functions are allowed, but discouraged and should always be documented as such.  Generally, prefer types to enforce invariants.



