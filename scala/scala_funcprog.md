# Scala functional programming 
## Evaluation Rules

- Call by value: evaluates the function arguments before calling the function
- Call by name: evaluates the function first, and then evaluates the arguments if need be

<!-- code -->
```scala
    def example = 2      // evaluated when called
    val example = 2      // evaluated immediately
    lazy val example = 2 // evaluated once when needed
    
    def square(x: Double)    // call by value
    def square(x: => Double) // call by name
    def myFct(bindings: Int*) = { ... } // bindings is a sequence of int, containing a varying # of arguments
```

## Higher order functions

These are functions that take a function as a parameter or return functions.
```scala
    // sum() returns a function that takes two integers and returns an integer  
    def sum(f: Int => Int): (Int, Int) => Int = {  
      def sumf(a: Int, b: Int): Int = {...}  
      sumf  
    } 
    
    // same as above. Its type is (Int => Int) => (Int, Int) => Int  
    def sum(f: Int => Int)(a: Int, b: Int): Int = { ... } 

    // Called like this
    sum((x: Int) => x * x * x)          // Anonymous function, i.e. does not have a name  
    sum(x => x * x * x)                 // Same anonymous function with type inferred

    def cube(x: Int) = x * x * x  
    sum(x => x * x * x)(1, 10) // sum of cubes from 1 to 10
    sum(cube)(1, 10)           // same as above      
```

## Currying

Converting a function with multiple arguments into a function with a
single argument that returns another function.
```scala
    def f(a: Int, b: Int): Int // uncurried version (type is (Int, Int) => Int)
    def f(a: Int)(b: Int): Int // curried version (type is Int => Int => Int)
```
    
## Classes
```scala
    class MyClass(x: Int, y: Int) {           // Defines a new type MyClass with a constructor  
      require(y > 0, "y must be positive")    // precondition, triggering an IllegalArgumentException if not met  
      def this (x: Int) = { ... }             // auxiliary constructor   
      def nb1 = x                             // public method computed every time it is called  
      def nb2 = y  
      private def test(a: Int): Int = { ... } // private method  
      val nb3 = x + y                         // computed only once  
      override def toString =                 // overridden method  
          member1 + ", " + member2 
    }

    new MyClass(1, 2) // creates a new object of type
```

`this` references the current object, `assert(<condition>)` issues `AssertionError` if condition
is not met. See `scala.Predef` for `require`, `assume` and `assert`.

## Operators

`myObject myMethod 1` is the same as calling `myObject.myMethod(1)`

Operator (i.e. function) names can be alphanumeric, symbolic (e.g. `x1`, `*`, `+?%&`, `vector_++`, `counter_=`)
    
The precedence of an operator is determined by its first character, with the following increasing order of priority:

    (all letters)
    |
    ^
    &
    < >
    = !
    :
    + -
    * / %
    (all other special characters)
   
The associativity of an operator is determined by its last character: Right-associative if ending with `:`, Left-associative otherwise.
   
Note that assignment operators have lowest precedence. (Read Scala Language Specification 2.9 sections 6.12.3, 6.12.4 for more info)

## Class hierarchies
```scala
    abstract class TopLevel {    // abstract class  
      def method1(x: Int): Int   // abstract method  
      def method2(x: Int): Int = { ... }  
    }

    class Level1 extends TopLevel {  
      def method1(x: Int): Int = { ... }  
      override def method2(x: Int): Int = { ...} // TopLevel's method2 needs to be explicitly overridden  
    }

    object MyObject extends TopLevel { ... } // defines a singleton object. No other instance can be created
```

To create a runnable application in Scala:

```scala
    object Hello {  
      def main(args: Array[String]) = println("Hello world")  
    }
```
    
or
```scala
    object Hello extends App {
      println("Hello World")
    }
```

## Class Organization

- Classes and objects are organized in packages (`package myPackage`).

- They can be referenced through import statements (`import myPackage.MyClass`, `import myPackage._`,
`import myPackage.{MyClass1, MyClass2}`, `import myPackage.{MyClass1 => A}`)

- They can also be directly referenced in the code with the fully qualified name (`new myPackage.MyClass1`)

- All members of packages `scala` and `java.lang` as well as all members of the object `scala.Predef` are automatically imported.

- Traits are similar to Java interfaces, except they can have non-abstract members:
```scala
        trait Planar { ... }
        class Square extends Shape with Planar
```

- General object hierarchy:

  - `scala.Any` base type of all types. Has methods `hashCode` and `toString` that can be overridden
  - `scala.AnyVal` base type of all primitive types. (`scala.Double`, `scala.Float`, etc.)
  - `scala.AnyRef` base type of all reference types. (alias of `java.lang.Object`, supertype of `java.lang.String`, `scala.List`, any user-defined class)
  - `scala.Null` is a subtype of any `scala.AnyRef` (`null` is the only instance of type `Null`), and `scala.Nothing` is a subtype of any other type without any instance.

## Type Parameters

Conceptually similar to C++ templates or Java generics. These can apply to classes, traits or functions.
```scala
    class MyClass[T](arg1: T) { ... }  
    new MyClass[Int](1)  
    new MyClass(1)   // the type is being inferred, i.e. determined based on the value arguments  
```

It is possible to restrict the type being used, e.g.
```scala
    def myFct[T <: TopLevel](arg: T): T = { ... } // T must derive from TopLevel or be TopLevel
    def myFct[T >: Level1](arg: T): T = { ... }   // T must be a supertype of Level1
    def myFct[T >: Level1 <: Top Level](arg: T): T = { ... }
```

## Variance

Given `A <: B`

If `C[A] <: C[B]`, `C` is covariant

If `C[A] >: C[B]`, `C` is contravariant

Otherwise C is nonvariant
```scala
    class C[+A] { ... } // C is covariant
    class C[-A] { ... } // C is contravariant
    class C[A]  { ... } // C is nonvariant
```

For a function, if `A2 <: A1` and `B1 <: B2`, then `A1 => B1 <: A2 => B2`.

Functions must be contravariant in their argument types and covariant in their result types, e.g.
```scala
    trait Function1[-T, +U] {
      def apply(x: T): U
    } // Variance check is OK because T is contravariant and U is covariant

    class Array[+T] {
      def update(x: T)
    } // variance checks fails
```

Find out more about variance in
[lecture 4.4](https://class.coursera.org/progfun-2012-001/lecture/81)
and [lecture 4.5](https://class.coursera.org/progfun-2012-001/lecture/83)

## Pattern Matching

Pattern matching is used for decomposing data structures:
```scala
    unknownObject match {
      case MyClass(n) => ...
      case MyClass2(a, b) => ...
    }
```

Here are a few example patterns
```scala
    (someList: List[T]) match {
      case Nil => ...          // empty list
      case x :: Nil => ...     // list with only one element
      case List(x) => ...      // same as above
      case x :: xs => ...      // a list with at least one element. x is bound to the head,
                               // xs to the tail. xs could be Nil or some other list.
      case 1 :: 2 :: cs => ... // lists that starts with 1 and then 2
      case (x, y) :: ps => ... // a list where the head element is a pair
      case _ => ...            // default case if none of the above matches
    }
```

The last example shows that every pattern consists of sub-patterns: it
only matches lists with at least one element, where that element is a
pair. `x` and `y` are again patterns that could match only specific
types.

### Options

Pattern matching can also be used for `Option` values. Some
functions (like `Map.get`) return a value of type `Option[T]` which
is either a value of type `Some[T]` or the value `None`:
```scala
    val myMap = Map("a" -> 42, "b" -> 43)
    def getMapValue(s: String): String = {
      myMap get s match {
        case Some(nb) => "Value found: " + nb
        case None => "No value found"
      }
    }
    getMapValue("a")  // "Value found: 42"
    getMapValue("c")  // "No value found"
```

Most of the times when you write a pattern match on an option value,
the same expression can be written more concisely using combinator
methods of the `Option` class. For example, the function `getMapValue`
can be written as follows: 
```scala
    def getMapValue(s: String): String =
      myMap.get(s).map("Value found: " + _).getOrElse("No value found")
```

### Pattern Matching in Anonymous Functions

Pattern matches are also used quite often in anonymous functions:
```scala
    val pairs: List[(Char, Int)] = ('a', 2) :: ('b', 3) :: Nil
    val chars: List[Char] = pairs.map(p => p match {
      case (ch, num) => ch
    })
```

Instead of `p => p match { case ... }`, you can simply write `{case ...}`, so the above example becomes more concise:
```scala
    val chars: List[Char] = pairs map {
      case (ch, num) => ch
    }
```

## Collections

Scala defines several collection classes:

### Base Classes
- [`Iterable`](http://www.scala-lang.org/api/current/index.html#scala.collection.Iterable) (collections you can iterate on)
- [`Seq`](http://www.scala-lang.org/api/current/index.html#scala.collection.Seq) (ordered sequences)
- [`Set`](http://www.scala-lang.org/api/current/index.html#scala.collection.Set)
- [`Map`](http://www.scala-lang.org/api/current/index.html#scala.collection.Map) (lookup data structure)

### Immutable Collections
- [`List`](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.List) (linked list, provides fast sequential access)
- [`Stream`](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Stream) (same as List, except that the tail is evaluated only on demand)
- [`Vector`](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Vector) (array-like type, implemented as tree of blocks, provides fast random access)
- [`Range`](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Range) (ordered sequence of integers with equal spacing)
- [`String`](http://docs.oracle.com/javase/1.5.0/docs/api/java/lang/String.html) (Java type, implicitly converted to a character sequence, so you can treat every string like a `Seq[Char]`)
- [`Map`](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Map) (collection that maps keys to values)
- [`Set`](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Set) (collection without duplicate elements)

### Mutable Collections
- [`Array`](http://www.scala-lang.org/api/current/index.html#scala.Array) (Scala arrays are native JVM arrays at runtime, therefore they are very performant)
- Scala also has mutable maps and sets; these should only be used if there are performance issues with immutable types

### Examples
```scala
    val fruitList = List("apples", "oranges", "pears")
    // Alternative syntax for lists
    val fruit = "apples" :: ("oranges" :: ("pears" :: Nil)) // parens optional, :: is right-associative
    fruit.head   // "apples"
    fruit.tail   // List("oranges", "pears")
    val empty = List()
    val empty = Nil

    val nums = Vector("louis", "frank", "hiromi")
    nums(1)                     // element at index 1, returns "frank", complexity O(log(n))
    nums.updated(2, "helena")   // new vector with a different string at index 2, complexity O(log(n))
    
    val fruitSet = Set("apple", "banana", "pear", "banana")
    fruitSet.size    // returns 3: there are no duplicates, only one banana

    val r: Range = 1 until 5 // 1, 2, 3, 4
    val s: Range = 1 to 5    // 1, 2, 3, 4, 5
    1 to 10 by 3  // 1, 4, 7, 10
    6 to 1 by -2  // 6, 4, 2

    val s = (1 to 6).toSet
    s map (_ + 2) // adds 2 to each element of the set

    val s = "Hello World"
    s filter (c => c.isUpper) // returns "HW"; strings can be treated as Seq[Char]

    // Operations on sequences
    val xs = List(...)
    xs.length   // number of elements, complexity O(n)
    xs.last     // last element (exception if xs is empty), complexity O(n)
    xs.init     // all elements of xs but the last (exception if xs is empty), complexity O(n)
    xs take n   // first n elements of xs
    xs drop n   // the rest of the collection after taking n elements
    xs(n)       // the nth element of xs, complexity O(n)
    xs ++ ys    // concatenation, complexity O(n)
    xs.reverse  // reverse the order, complexity O(n)
    xs updated(n, x)  // same list than xs, except at index n where it contains x, complexity O(n)
    xs indexOf x      // the index of the first element equal to x (-1 otherwise)
    xs contains x     // same as xs indexOf x >= 0
    xs filter p       // returns a list of the elements that satisfy the predicate p
    xs filterNot p    // filter with negated p 
    xs partition p    // same as (xs filter p, xs filterNot p)
    xs takeWhile p    // the longest prefix consisting of elements that satisfy p
    xs dropWhile p    // the remainder of the list after any leading element satisfying p have been removed
    xs span p         // same as (xs takeWhile p, xs dropWhile p)
    
    List(x1, ..., xn) reduceLeft op    // (...(x1 op x2) op x3) op ...) op xn
    List(x1, ..., xn).foldLeft(z)(op)  // (...( z op x1) op x2) op ...) op xn
    List(x1, ..., xn) reduceRight op   // x1 op (... (x{n-1} op xn) ...)
    List(x1, ..., xn).foldRight(z)(op) // x1 op (... (    xn op  z) ...)
    
    xs exists p    // true if there is at least one element for which predicate p is true
    xs forall p    // true if p(x) is true for all elements
    xs zip ys      // returns a list of pairs which groups elements with same index together
    xs unzip       // opposite of zip: returns a pair of two lists
    xs.flatMap f   // applies the function to all elements and concatenates the result
    xs.sum         // sum of elements of the numeric collection
    xs.product     // product of elements of the numeric collection
    xs.max         // maximum of collection
    xs.min         // minimum of collection
    xs.flatten     // flattens a collection of collection into a single-level collection
    xs groupBy f   // returns a map which points to a list of elements
    xs distinct    // sequence of distinct entries (removes duplicates)

    x +: xs  // creates a new collection with leading element x
    xs :+ x  // creates a new collection with trailing element x

    // Operations on maps
    val myMap = Map("I" -> 1, "V" -> 5, "X" -> 10)  // create a map
    myMap("I")      // => 1  
    myMap("A")      // => java.util.NoSuchElementException  
    myMap get "A"   // => None 
    myMap get "I"   // => Some(1)
    myMap.updated("V", 15)  // returns a new map where "V" maps to 15 (entry is updated)
                            // if the key ("V" here) does not exist, a new entry is added

    // Operations on Streams
    val xs = Stream(1, 2, 3)
    val xs = Stream.cons(1, Stream.cons(2, Stream.cons(3, Stream.empty))) // same as above
    (1 to 1000).toStream // => Stream(1, ?)
    x #:: xs // Same as Stream.cons(x, xs)
             // In the Stream's cons operator, the second parameter (the tail)
             // is defined as a "call by name" parameter.
             // Note that x::xs always produces a List
```

## Pairs (similar for larger Tuples)
```scala
    val pair = ("answer", 42)   // type: (String, Int)
    val (label, value) = pair   // label = "answer", value = 42  
    pair._1 // "answer"  
    pair._2 // 42  
```

## Ordering

There is already a class in the standard library that represents orderings: `scala.math.Ordering[T]` which contains
comparison functions such as `lt()` and `gt()` for standard types. Types with a single natural ordering should inherit from 
the trait `scala.math.Ordered[T]`.
```scala
    import math.Ordering  

    def msort[T](xs: List[T])(implicit ord: Ordering) = { ...}  
    msort(fruits)(Ordering.String)  
    msort(fruits)   // the compiler figures out the right ordering  
```

## For-Comprehensions

A for-comprehension is syntactic sugar for `map`, `flatMap` and `filter` operations on collections.

The general form is `for (s) yield e`

- `s` is a sequence of generators and filters
- `p <- e` is a generator
- `if f` is a filter
- If there are several generators (equivalent of a nested loop), the last generator varies faster than the first
- You can use `{ s }` instead of `( s )` if you want to use multiple lines without requiring semicolons
- `e` is an element of the resulting collection

### Example 1
```scala
    // list all combinations of numbers x and y where x is drawn from
    // 1 to M and y is drawn from 1 to N
    for (x <- 1 to M; y <- 1 to N)
      yield (x,y)
```

is equivalent to
```scala        
    (1 to M) flatMap (x => (1 to N) map (y => (x, y)))
```

### Translation Rules

A for-expression looks like a traditional for loop but works differently internally

`for (x <- e1) yield e2` is translated to `e1.map(x => e2)`

`for (x <- e1 if f) yield e2` is translated to `for (x <- e1.filter(x => f)) yield e2`

`for (x <- e1; y <- e2) yield e3` is translated to `e1.flatMap(x => for (y <- e2) yield e3)`

This means you can use a for-comprehension for your own type, as long
as you define `map`, `flatMap` and `filter`.

For more, see [lecture 6.5](https://class.coursera.org/progfun-2012-001/lecture/111).

### Example 2
```scala
    for {  
      i <- 1 until n  
      j <- 1 until i  
      if isPrime(i + j)  
    } yield (i, j)  
```

is equivalent to
```scala
    for (i <- 1 until n; j <- 1 until i if isPrime(i + j))
        yield (i, j)  
```

is equivalent to
```scala
    (1 until n).flatMap(i => (1 until i).filter(j => isPrime(i + j)).map(j => (i, j)))
```


# Design Patterns

## Reactive SheetCheat

---


## Partial Functions

A subtype of trait `Function1` that is well defined on a subset of its domain.
```scala
trait PartialFunction[-A, +R] extends Function1[-A, +R] {
  def apply(x: A): R
  def isDefinedAt(x: A): Boolean
}
```

Every concrete implementation of PartialFunction has the usual `apply` method along with a boolean method `isDefinedAt`.

**Important:** An implementation of PartialFunction can return `true` for `isDefinedAt` but still end up throwing RuntimeException (like MatchError in pattern-matching implementation).

A concise way of constructing partial functions is shown in the following example:

```scala
trait Coin {}
case class Gold() extends Coin {}
case class Silver() extends Coin {}

val pf: PartialFunction[Coin, String] = {
  case Gold() => "a golden coin"
  // no case for Silver(), because we're only interested in Gold()
}

println(pf.isDefinedAt(Gold()))   // true 
println(pf.isDefinedAt(Silver())) // false
println(pf(Gold()))               // a golden coin
println(pf(Silver()))             // throws a scala.MatchError
```


## For-Comprehension and Pattern Matching

A general For-Comprehension is described in Scala Cheat Sheet here: [https://github.com/lrytz/progfun-wiki/blob/gh-pages/CheatSheet.md](https://github.com/lrytz/progfun-wiki/blob/gh-pages/CheatSheet.md). One can also use Patterns inside for-expression. The simplest form of for-expression pattern looks like

```scala
for { pat <- expr} yield e
```

where `pat` is a pattern containing a single variable `x`. We translate the `pat <- expr` part of the expression to

```scala
x <- expr withFilter {
  case pat => true
  case _ => false
} map {
  case pat => x
}
```

The remaining parts are translated to ` map, flatMap, withFilter` according to standard for-comprehension rules.


## Random Generators with For-Expressions

The `map` and `flatMap` methods can be overridden to make a for-expression versatile, for example to generate random elements from an arbitrary collection like lists, sets etc. Define the following trait `Generator` to do this.

```scala
trait Generator[+T] { self =>
  def generate: T
  def map[S](f: T => S) : Generator[S] = new Generator[S] {
    def generate = f(self.generate)
  }
  def flatMap[S](f: T => Generator[S]) : Generator[S] = new Generator[S] {
    def generate = f(self.generate).generate
  }
}
```

Let's define a basic integer random generator as 

```scala
val integers = new Generator[Int] {
  val rand = new java.util.Random
  def generate = rand.nextInt()
}
```

With these definition, and a basic definition of `integer` generator, we can map it to other domains like `booleans, pairs, intervals` using for-expression magic


```scala
val booleans = for {x <- integers} yield x > 0
val pairs = for {x <- integers; y<- integers} yield (x, y)
def interval(lo: Int, hi: Int) : Generator[Int] = for { x <- integers } yield lo + math.abs(x) % (hi - lo)
```


## Monads

A monad is a parametric type `M[T]` with two operations: `flatMap` and `unit`. 

```scala
trait M[T] {
  def flatMap[U](f: T => M[U]) : M[U]
  def unit[T](x: T) : M[T]
}
```

These operations must satisfy three important properties:

1. **Associativity:** `(x flatMap f) flatMap g == x flatMap (y => f(y) flatMap g)`
2. **Left unit:** `unit(x) flatMap f == f(x)`

3. **Right unit:** `m flatMap unit == m`

Many standard Scala Objects like `List, Set, Option, Gen` are monads with identical implementation of `flatMap` and specialized implementation of `unit`. An example of non-monad is a special `Try` object that fails with a non-fatal exception because it fails to satisfy Left unit (See lectures). 


## Monads and For-Expression

Monads help simplify for-expressions. 

**Associativity** helps us "inline" nested for-expressions and write something like

```scala
for { x <- e1; y <- e2(x) ... }
```

**Right unit** helps us eliminate for-expression using the identity

```scala
for{x <- m} yield x == m
```


## Pure functional programming

In a pure functional state, programs are side-effect free, and the concept of time isn't important (i.e. redoing the same steps in the same order produces the same result).

When evaluating a pure functional expression using the substitution model, no matter the evaluation order of the various sub-expressions, the result will be the same (some ways may take longer than others). An exception may be in the case where a sub-expression is never evaluated (e.g. second argument) but whose evaluation would loop forever.


## Mutable state

In a reactive system, some states eventually need to be changed in a mutable fashion. An object has a state if its behavior has a history. Every form of mutable state is constructed from variables:

```scala
var x: String = "abc"
x = "hi"
var nb = 42
```

The use of a stateful expression can complexify things. For a start, the evaluation order may matter. Also, the concept of identity and change gets more complex. When are two expressions considered the same? In the following (pure functional) example, x and y are always the same (concept of <b>referential transparency</b>):

```scala
val x = E; val y = E
val x = E; val y = x
```

But when a stateful variable is involved, the concept of equality is not as straightforward. "Being the same" is defined by the property of **operational equivalence**. x and y are operationally equivalent if no possible test can distinguish between them.

Considering two variables x and y, if you can create a function f so that f(x, y) returns a different result than f(x, x) then x and y are different. If no such function exist x and y are the same.

As a consequence, the substitution model ceases to be valid when using assignments.


## Loops

Variables and assignments are enough to model all programs with mutable states and loops in essence are not required. <b>Loops can be modeled using functions and lazy evaluation</b>. So, the expression

```scala
while (condition) { command }
```

can be modeled using function <tt>WHILE</tt> as

```scala
def WHILE(condition: => Boolean)(command: => Unit): Unit = 
    if (condition) {
        command
        WHILE(condition)(command)
    }
    else ()
```

**Note:**
* Both **condition** and **command** are **passed by name**
* **WHILE** is **tail recursive**


## For loop

The treatment of for loops is similar to the <b>For-Comprehensions</b> commonly used in functional programming. The general expression for <tt>for loop</tt> equivalent in Scala is

```scala
for(v1 <- e1; v2 <- e2; ...; v_n <- e_n) command
```

Note a few subtle differences from a For-expression. There is no `yield` expression, `command` can contain mutable states and `e1, e2, ..., e_n` are expressions over arbitrary Scala collections. This for loop is translated by Scala using a **foreach** combinator defined over any arbitrary collection. The signature for **foreach** over collection **T** looks like this

```scala
def foreach(f: T => Unit) : Unit
```

Using foreach, the general for loop is recursively translated as follows:

```scala
for(v1 <- e1; v2 <- e2; ...; v_n <- e_n) command = 
    e1 foreach (v1 => for(v2 <- e2; ...; v_n <- e_n) command)
```


## Monads and Effect

Monads and their operations like flatMap help us handle programs with side-effects (like exceptions) elegantly. This is best demonstrated by a Try-expression. <b>Note: </b> Try-expression is not strictly a Monad because it does not satisfy all three laws of Monad mentioned above. Although, it still helps handle expressions with exceptions. 


#### Try

The parametric Try class as defined in Scala.util looks like this:

```scala
abstract class Try[T]
case class Success[T](elem: T) extends Try[T]
case class Failure(t: Throwable) extends Try[Nothing]
```

`Try[T]` can either be `Success[T]` or `Failure(t: Throwable)`

```scala
import scala.util.{Try, Success, Failure}

def answerToLife(nb: Int) : Try[Int] = {
  if (nb == 42) Success(nb)
  else Failure(new Exception("WRONG"))
}

answerToLife(42) match {
  case Success(t)           => t        // returns 42
  case failure @ Failure(e) => failure  // returns Failure(java.Lang.Exception: WRONG)
}
```


Now consider a sequence of scala method calls:

```scala
val o1 = SomeTrait()
val o2 = o1.f1()
val o3 = o2.f2()
```

All of these method calls are synchronous, blocking and the sequence computes to completion as long as none of the intermediate methods throw an exception. But what if one of the methods, say `f2` does throw an exception? The `Try` class defined above helps handle these exceptions elegantly, provided we change return types of all methods `f1, f2, ...` to `Try[T]`. Because then, the sequence of method calls translates into an elegant for-comprehension:

```scala
val o1 = SomeTrait()
val ans = for {
    o2 <- o1.f1();
    o3 <- o2.f2()
} yield o3
```

This transformation is possible because `Try` satisfies 2 properties related to `flatMap` and `unit` of a **monad**. If any of the intermediate methods `f1, f2` throws and exception, value of `ans` becomes `Failure`. Otherwise, it becomes `Success[T]`.


## Monads and Latency

The Try Class in previous section worked on synchronous computation. Synchronous programs with side effects block the subsequent instructions as long as the current computation runs. Blocking on expensive computation might render the entire program slow!. **Future** is a type of monad the helps handle exceptions and latency and turns the program in a non-blocking asynchronous program.


#### Future

Future trait is defined in scala.concurrent as:

```scala
trait Future[T] {
    def onComplete(callback: Try[T] => Unit)
    (implicit executor: ExecutionContext): Unit
}
```

The Future trait contains a method `onComplete` which itself takes a method, `callback` to be called as soon as the value of current Future is available. The insight into working of Future can be obtained by looking at its companion object:

```scala
object Future {
    def apply(body: => T)(implicit context: ExecutionContext): Future[T]
}
```

This object has an apply method that starts an asynchronous computation in current context, returns a `Future` object. We can then subscribe to this `Future` object to be notified when the computation finishes.

```scala
import scala.util.{Try, Success, Failure}
import scala.concurrent._
import ExecutionContext.Implicits.global

// The function to be run asyncronously
val answerToLife: Future[Int] = future {
  42
}

// These are various callback functions that can be defined  
answerToLife onComplete {
  case Success(result) => result
  case Failure(t) => println("An error has occured: " + t.getMessage)
}

answerToLife onSuccess {
  case result => result
}
  
answerToLife onFailure {
  case t => println("An error has occured: " + t.getMessage)
}

answerToLife    // only works if the future is completed
```


#### Combinators on Future

A `Future` is a `Monad` and has `map`, `filter`, `flatMap` defined on it. In addition, Scala's Futures define two additional methods:

```scala
def recover(f: PartialFunction[Throwable, T]): Future[T]
def recoverWith(f: PartialFunction[Throwable, Future[T]]): Future[T]
```

These functions return robust features in case current features fail. 

Finally, a `Future` extends from a trait called `Awaitable` that has two blocking methods, `ready` and `result` which take the value 'out of' the Future. The signatures of these methods are

```scala
trait Awaitable[T] extends AnyRef {
    abstract def ready(t: Duration): Unit
    abstract def result(t: Duration): T
}
```

Both these methods block the current execution for a duration of `t`. If the future completes its execution, they return: `result` returns the actual value of the computation, while `ready` returns a Unit. If the future fails to complete within time t, the methods throw a `TimeoutException`.

`Await` can be used to wait for a future with a specified timeout, e.g.

```scala
userInput: Future[String] = ...
Await.result(userInput, 10 seconds)   // waits for user input for 10 seconds, after which throws a TimeoutException
```


#### async and await

Async and await allow to run some part of the code aynchronously. The following code computes asynchronously any future inside the `await` block

```scala
import scala.async.Async._

def retry(noTimes: Int)(block: => Future[T]): Future[T] = async {
  var i = 0;
  var result: Try[T] = Failure(new Exception("Problem!"))
  while (i < noTimes && result.isFailure) {
    result = await { Try(block) }
    i += 1
  }
  result.get
}
```


#### Promises

A Promise is a monad which can complete a future, with a value if successful (thus completing the promise) or with an exception on failure (failing the promise).

```scala
trait Promise[T] {
  def future: Future[T]
  def complete(result: Try[T]): Unit  // to call when the promise is completed
  def tryComplete(result: Try[T]): Boolean
}
```

It is used as follows:

```scala
val p = Promise[T]          // defines a promise
p.future                    // returns a future that will complete when p.complete() is called
p.complete(Try[T])          // completes the future
p success T                 // successfully completes the promise
p failure (new <Exception>) // failed with an exception
```

## Observables
Observables are asynchronous streams of data. Contrary to Futures, they can return multiple values.

```scala
trait Observable[T] {
  def Subscribe(observer: Observer[T]): Subscription
}

trait Observer[T] {
  def onNext(value: T): Unit            // callback function when there's a next value
  def onError(error: Throwable): Unit  // callback function where there's an error
  def onCompleted(): Unit              // callback function when there is no more value
}

trait Subscription {
  def unsubscribe(): Unit    // to call when we're not interested in recieving any more values
}
```

Observables can be used as follows:

```scala
import rx.lang.scala._

val ticks: Observable[Long] = Observable.interval(1 seconds)
val evens: Observable[Long] = ticks.filter(s => s%2 == 0)

val bugs: Observable[Seq[Long]] = ticks.buffer(2, 1)
val s = bugs.subscribe(b=>println(b))

s.unsubscribe()
```

Some observable functions (see more at http://rxscala.github.io/scaladoc/index.html#rx.lang.scala.Observable):

- `Observable[T].flatMap(T => Observable[T]): Observable[T]` merges a list of observables into a single observable in a non-deterministic order
- `Observable[T].concat(T => Observable[T]): Observable[T]` merges a list of observables into a single observable, putting the results of the first observable first, etc.
- `groupBy[K](keySelector: T=>K): Observable[(K, Observable[T])]` returns an observable of observables, where the element are grouped by the key returned by `keySelector`

#### Subscriptions

Subscriptions are returned by Observables to allow to unsubscribe. With hot observables, all subscribers share the same source, which produces results independent of subscribers. With cold observables each subscriber has its own private source. If there is no subscriber no computation is performed.

Subscriptions have several subtypes: `BooleanSubscription` (was the subscription unsubscribed or not?), `CompositeSubscription` (collection of subscriptions that will be unsubscribed all at once), `MultipleAssignmentSubscription` (always has a single subscription at a time)

```scala
val subscription = Subscription { println("Bye") }
subscription.unsubscribe() // prints the message
subscription.unsubscribe() // doesn't print it again
```

#### Creating Rx Streams

Using the following constructor that takes an Observer and returns a Subscription

```scala
object Observable {
  def apply[T](subscribe: Observer[T] => Subscription): Observable[T]
}
```

It is possible to create several observables. The following functions suppose they are part of an Observable type (calls to `subscribe(...)` implicitely mean `this.subscribe(...)`):

```scala
// Observable never: never returns anything
def never(): Observable[Nothing] = Observable[Nothing](observer => { Subscription {} })

// Observable error: returns an error
def apply[T](error: Throwable): Observable[T] =
    Observable[T](observer => {
    observer.onError(error)
    Subscription {}
  }
)

// Observable startWith: prepends some elements in front of an Observable
def startWith(ss: T*): Observable[T] = {
    Observable[T](observer => {
    for(s <- ss) observer onNext(s)
    subscribe(observer)
  })
})

// filter: filters results based on a predicate
def filter(p: T=>Boolean): Observable[T] = {
  Observable[T](observer => {
    subscribe(
      (t: T) => { if(p(t)) observer.onNext(t) },
      (e: Thowable) => { observer.onError(e) },
      () => { observer.onCompleted() }
    )
  })
}

// map: create an observable of a different type given a mapping function
def map(f: T=>S): Observable[S] = {
  Observable[S](observer => {
    subscribe(
      (t: T) => { observer.onNext(f(t)) },
      (e: Thowable) => { observer.onError(e) },
      () => { observer.onCompleted() }
    )
  }
}

// Turns a Future into an Observable with just one value
def from(f: Future[T])(implicit execContext: ExecutionContext): Observable[T] = {
    val subject = AsyncSubject[T]()
    f onComplete {
      case Failure(e) => { subject.onError(e) }
      case Success(c) => { subject.onNext(c); subject.onCompleted() }
    }
    subject
}
```


#### Blocking Observables

`Observable.toBlockingObservable()` returns a blocking observable (to use with care). Everything else is non-blocking.

```scala
val xs: Observable[Long] = Observable.interval(1 second).take(5)
val ys: List[Long] = xs.toBlockingObservable.toList
```

#### Schedulers

Schedulers allow to run a block of code in a separate thread. The Subscription returned by its constructor allows to stop the scheduler.

```scala
trait Observable[T] {
  def observeOn(scheduler: Scheduler): Observable[T]
}

trait Scheduler {
  def schedule(work: => Unit): Subscription
  def schedule(work: Scheduler => Subscription): Subscription
  def schedule(work: (=>Unit)=>Unit): Subscription
}

val scheduler = Scheduler.NewThreadScheduler
val subscription = scheduler.schedule { // schedules the block on another thread
  println("Hello world")
}


// Converts an iterable into an observable
// works even with an infinite iterable
def from[T](seq: Iterable[T])
           (implicit scheduler: Scheduler): Obserable[T] = {
   Observable[T](observer => {
     val it = seq.iterator()
     scheduler.schedule(self => {     // the block between { ... } is run in a separate thread
       if (it.hasnext) { observer.onNext(it.next()); self() } // calling self() schedules the block of code to be executed again
       else { observer.onCompleted() }
     })
   })
}
```

## Actors

Actors represent objects and their interactions, resembling human organizations. They are useful to deal with the complexity of writing multi-threaded applications (with their synchronizations, deadlocks, etc.)

```scala
type Receive = PartialFunction[Any, Unit]

trait Actor {
  def receive: Receive
}
```

An actor has the following properties:
- It is an object with an identity
- It has a behavior
- It only interacts using asynchronous message

Note: to use Actors in Eclipse you need to run a Run Configuration whose main class is `akka.Main` and whose Program argument is the full Main class name

An actor can be used as follows:

```scala
import akka.actor._

class Counter extends Actor {
  var count = 0
  def receive = {
    case "incr" => count += 1
    case ("get", customer: ActorRef) => customer ! count // '!' means sends the message 'count' to the customer
    case "get" => sender ! count // same as above, except sender means the sender of the message
  }
}
```

#### The Actor's Context

The Actor type describes the behavior (represented by a Receive, which is a PartialFunction), the execution is done by its ActorContext. An Actor can change its behavior by either pushing a new behavior on top of a stack or just purely replace the old behavior.

```scala
trait ActorContext {
  def become(behavior: Receive, discardOld: Boolean = true): Unit // changes the behavior
  def unbecome(): Unit                                            // reverts to the previous behavior
  def actorOf(p: Props, name: String): ActorRef                   // creates a new actor
  def stop(a: ActorRef): Unit                                     // stops an actor
  def watch(target: ActorRed): ActorRef                           // watches whenever an Actor is stopped
  def unwatch(target: ActorRed): ActorRef                         // unwatches
  def parent: ActorRef                                            // the Actor's parent
  def child(name: String): Option[ActorRef]                       // returns a child if it exists
  def children: Iterable[ActorRef]                                // returns all supervised children
}

class myActor extends Actor {
   ...
   context.parent ! aMessage // sends a message to the parent Actor
   context.stop(self)        // stops oneself
   ...
}
```


The following example is changing the Actor's behavior any time the amount is changed. The upside of this method is that 1) the state change is explicit and done by calling `context.become()` and 2) the state is scoped to the current behavior.

```scala
class Counter extends Actor {
  def counter(n: Int): Receive = {
    case "incr" => context.become(counter(n + 1))
    case "get" => sender ! n
  }
  def receive = counter(0)
}
```

#### Children and hierarchy

Each Actor can create children actors, creating a hierarchy.

```scala
class Main extends Actor {
  val counter = context.actorOf(Props[Counter], "counter")  // creates a Counter actor named "counter"

  counter ! "incr"
  counter ! "incr"
  counter ! "incr"
  counter ! "get"

  def receive = {	// receives the messages from Counter
    case count: Int =>
      println(s"count was $count")
      context.stop(self)
    }
  }
}
```

Each actor maintains a list of the actors it created:
- the child is added to the list when context.actorOf returns
- the child is removed when Terminated is received
- an actor name is available IF there is no such child. Actors are identified by their names, so they must be unique.

#### Message Processing Semantics

There is no direct access to an actor behavior. Only messages can be sent to known addresses (`ActorRef`). Those addresses can be either be oneself (`self`), the address returned when creating a new actor, or when received by a message (e.g. `sender`)

Actors are completely insulated from each other except for messages they send each other. Their computation can be run concurrently. However, a specific actor is single-threaded - its messages are received sequentially. Processing a message is the atomic unit of execution and cannot be interrupted.

It is good practice to define an Actor's messages in its companion object. Here, each operation is effectively synchronized as all messages are serialized.

```scala
object BankAccount {
  case class Deposit(amount: BigInt) {
    require(amount > 0)
  }
  case class Withdraw(amount: BigInt) {
    require(amount > 0)
  }
  case object Done
  case object Failed
}

class BankAccount extends Actor {
  import BankAccount._

  var balance = BigInt(0)

  def receive = {
    case Deposit(amount) => balance += amount
                            sender ! Done
    case Withdraw(amount) if amount <= balance => balance -= amount
                                                  sender ! Done
    case _ => sender ! Failed
  }
}
```

Note that `pipeTo` can be used to foward a message (`theAccount deposit(500) pipeTo sender`)

Because communication is through messages, there is no delivery guarantee. Hence the need of messages of acknowledgement and/or repeat. There are various strategies to deal with this:
- at-most-once: send a message, without guarantee it will be received
- at-least-once: keep sending messages until an ack is received
- exactly-once: keep sending messages until an ack is received, but the recipient will only process the first message

You can call `context.setReceiveTimeout(10.seconds)` that sets a timeout:

```scala
def receive = {
  case Done => ...
  case ReceiveTimeout => ... 
}
```

The Akka library also includes a scheduler that sends a message or executes a block of code after a certain delay:

```scala
trait Scheduler {
  def scheduleOnce(delay: FiniteDuration, target: ActorRef, msg: Any)
  def scheduleOnce(delay: FiniteDuration)(block: => Unit)
}
```

#### Designing Actor Systems

When designing an Actor system, it is useful to:
- visualize a room full of people (i.e. the Actors)
- consider the goal to achieve 
- split the goal into subtasks that can be assigned to the various actors
- who needs to talk to whom?
- remember that you can easily create new Actors, even short-lived ones
- watch out for any blocking part
- prefer immutable data structures that can safely be shared
- do not refer to actor state from code running asynchronously

Consider a Web bot that recursively download content (down to a certain depth):
- one Client Actor, which is sending download requests
- one Receptionist Actor, responsible for accepting incoming download requests from Clients. The Receptionist forwards the request to the Controller
- one Controller Actor, noting the pages already downloaded and dispatching the download jobs to Getter actors
- one or more Getter Actors whose job is to download a URL, check its links and tell the Controller about those links
- each message between the Controller and the Getter contains the depth level
- once this is done, the Controller notifies the Receptionist, who remembers the Client who asked for that request and notifies it

#### Testing Actor Systems

Tests can only verify externally observable effects. Akka's TestProbe allows to check that:

```scala
implicit val system = ActorSystem("TestSys")
val myActor = system.actorOf(Props[MyActor])
val p = TestProbe()
p.send(myActor, "Message")
p.exceptMsg("Ack")
p.send(myActor, "Message")
p.expectNoMsg(1.second)
system.shutdown()
```

It can also be run from inside TestProbe:

```scala
new TestKit(ActorSystem("TestSys")) with ImplicitSender {
  val myActor = system.actorOf(Props[MyActor])
  myActor ! "Message"
  expectMsg("Ack")
  send(myActor, "Message")
  expectNoMsg(1.second)
  system.shutdown()
}
```

You can use dependency injection when the system relies from external sources, like overriding factory methods that work as follows:
- have a method that will call `Props[MyActor]`
- its result is called by context.actorOf()
- the test can define a "fake Actor" (`object FakeMyActor extends MyActor { ... }`) that will override the method

You should start first the "leaves" actors and work your way to the parent actors.

#### Logging Actor Systems 

You can mix in your actor with `ActorLogging`, and use various log methods such as `log.debug` or `log.info`.
  
To see all the messages the actor is receiving, you can also define `receive` method as a `LoggingReceive`. 

```scala
def receive: Receive = LoggingReceive {
  case Replicate =>
  case Snapshot =>  
}
```
  
To see the log messages turn on akka debug level by adding the following in your run configuration. 

    -Dakka.loglevel=DEBUG -Dakka.actor.debug.receive=on

Alternatively, create a file named `src/test/resources/application.conf` with the following content:

    akka {
      loglevel = "DEBUG"
      actor {
        debug {
          receive = on
        }
      }
    }

#### Failure handling with Actors

What happens when an error happens with an actor? Where shall failures go? With the Actor models, Actors work together in teams (systems) and individual failures are handled by the team leader.

Resilience demands containment (i.e. the failure is isolated so that it cannot spread to other components) and delegation of failure (i.e. it is handled by someone else and not the failed component)

In the Supervisor model, the Supervisor needs to create its subordinates and will handle the exceptions encountered by its children. If a child fails, the supervisor may decide to stop it (`stop` message) or to restart it to get it back to a known good state and initial behavior (in Akka, the ActorRef stays valid after a restart).

An actor can decide a strategy by overriding `supervisorStrategy`, e.g.

```scala
class myActor extends Actor {
  override val supervisorStrategy = OneForOneStrategy(maxNrOfRetries = 5) {
     case _: Exception => SupervisorStrategy.Restart
  }
}
```

#### Lifecycle of an Actor

- An Actor will have its context create a child Actor, and gets `preStart()` called.
- In case of a failure, the supervisor gets consulted. The supervisor can stop the child or restart it (a restart is not externally visible). In case of a restart, the child Actor's `preRestart()` gets called. A new instance of the actor is created, after which its `postRestart()` method gets called. No message gets processed between the failure and the restart.
- An actor can be restarted several times.
- An actor can finally be stopped. It sends Stop to the context and its `postStop()` method will be called.

An Actor has the following methods that can be overridden:

```scala
trait Actor {
  def preStart(): Unit
  def preRestart(reason: Throwable, message: Option[Any]): Unit // the default behavior is to stop all children
  def postRestart(reason: Throwable): Unit                      // the default behavior is to call preStart()
  def postStop(): Unit
}
```

#### Lifecycle Monitoring

To remove the ambiguity where a message doesn't get a response because the recipient stopped or because the network is down, Akka supports Lifecycle Monitoring, aka DeathWatch:
- an Actor registers its interest using `context.watch(target)`
- it will receive a `Terminated(target)` message when the target stops
- it will not receive any direct messages from the target thereafter

The watcher receives a `Terminated(actor: ActorRef)` message:
- It is a special message that our code cannot send
- It comes with two implicit boolean flags: `existenceConfirmed` (was the watch sent when the target was still existing?) and `addressTerminated` (the watched actor was detected as unreachable)
- Terminated messages are handled by the actor context, so cannot be forwarded

#### The Error Kernel pattern

Keep important data near the root, delegate the risk to the leaves
- restarts are recursive
- as a result, restarts are more frequent near the leaves
- avoid restarting Actors with important states

#### EventStream

Because Actors can only send messages to a known address, the EventStream allows publication of messages to an unknown audience

```scala
trait EventStream {
  def subscribe(subscriber: ActorRef, topic: Class[_]): Boolean
  def unsubscribe(subscriber: ActorRef, topic: Class[_]): Boolean
  def unsubscribe(subscriber: ActorRef): Unit
  def publish(event: AnyRef): Unit
}


class MyActor extends Actor {
  context.system.eventStream.subscribe(self, classOf[LogEvent])
  def receive = {
    case e: LogEvent => ...
  }
  override def postStop(): Unit = {
    context.system.eventStream.unsubscribe(self)
  }
}
```

Unhanlded messages are passed to the Actor's `unhandled(message: Any)` method.

#### Persistent Actor State

The state of an Actor can be stored on disk to prevent data loss in case of a system failure.

There are two ways for persisting state:
- in-place updates mimics what is stored on the disk. This solution allows a fast recovery and limits the space used on the disk.
- persist changes in append-only fashion. This solution allows fast updates on the disk. Because changes are immutable they can be freely be replicated. Finally it allows to analyze the history of a state.
  - Command-Sourcing: persists the messages before processing them, persist acknowledgement when processed. Recovery works by sending the messages to the actor. A persistent Channel discards the messages already sent to the other actors
  - Event-Sourcing: Generate change requests (events) instead of modifying the local state. The events are sent to the log that stores them. The actor can either update its state when sending the event to the log or wait for the log to contact it back (in which case it can buffer any message while waiting for the log).
  - In both cases, immutable snapshots can be made at certain points of time. Recovery only applies recent changes to the latest snapshot.

Each strategy have their upsides and downsides in terms of performance to change the state, recover the state, etc.

The `stash` trait allows to buffer, e.g.

```scala
class MyActor extends Actor with Stash {
  var state: State = ...
  def receive = {
    case NewState(text) if !state.disabled =>
      ... // sends the event to the log
      context.become(waiting, discardOld = false)
  }
  def waiting(): Receive = {
    case e: Event =>
      state = state.updated(e)  // updates the state
      context.unbecome();       // reverts to the previous behavior
      unstashAll()              // processes all the stashed messages
    case _ => stash()           // stashes any message while waiting
  }
}
```

## Clusters

Actors are designed to be distributed. Which means they could be run on different cores, CPUs or even machines.

When actors are distributed across a network, several problems can arise. To begin with, data sharing can only be by value and not by reference. Other networking: low bandwidth, partial failure (some messages never make it)

On a network, Actors have a path that allow to reach them. An `ActorPath` is the full name (e.g. akka.tcp://HelloWorld@198.2.12.10:6565/user/greeter or akka://HelloWorld/user/greeter), whether the actor exists or not, whereas `ActorRef` points to an actor which was started and contains a UID (e.g. akka://HelloWorld/user/greeter#234235234).

You can use `context.actorSelection(path) ! Identify((path, sender))` to convert an ActorPath into an ActorRef. An `ActorIdentity((path: ActorPath, client: ActorRef), ref: Option(ActorRef)` message is then sent back with `ref` being the ActorRef if there is any.

It is also possible to get using `context.actorSelection("...")` which can take a local ("child/grandchild", "../sibling"), full path ("/user/myactor") or wildcards ("/user/controllers/*")

#### Creating clusters

A cluster is a set of nodes where all nodes know about each other and work to collaborate on a common task. A node can join a cluster by either sending a join request to any node of the cluster. It is accepted if all the nodes agree and know about the new node.

The akka-cluster module must be installed and properly configured (akka.actor.provider = akka.cluster.ClusterActorRefProvider). To start a cluster, a node must start a cluster and join it:

```scala
class MyActor extends Actor {
  val cluster = Cluster(context.system)
  cluster.subscribe(self, classOf[ClusterEvent.MemberUp])
  cluster.join(cluster.selfAddress)

  def receive = {
    case ClusterEvent.MemberUp(member) =>
      if (member.address != cluster.selfAddress) {
        // someone joined
      }
  }
}

class MyOtherActor extends Actor {
  val cluster = Cluster(context.system)
  cluster.subscribe(self, classOf[ClusterEvent.MemberUp])
  val main = cluster.selfAddress.copy(port = Some(2552)) // default port
  cluster.join(cluster.selfAddress)

  def receive = {
    case ClusterEvent.MemberRemoved(m, _) => if (m.address == main) context.stop(self)
  }
}
```

It is possible to create a new actor on a remote node

```scala
val node: Address = ...
val props = Props[MyClass].withDeploy(Deploy(scope = RemoteScope(node)))
val controller = context.actorOf(props, "myclass")
```

#### Eventual Consistency

- Strong consistency: after an update completes, all reads will return the updated value
- Weak consistency: after an update, conditions need to be met until reads return the updated value (inconsistency window)
- Eventual Consistency (a form of weak consistency): once no more updates are made to an object there is a time after which all reads return the last written value.

In a cluster, the data is propagated through messages. Which means that collaborating actors can be at most eventually consistent.

#### Actor Composition

Since an Actor is only defined by its accepted message types, its structure may change over time.

Various patterns can be used with actors:

- The Customer Pattern: typical request/reply, where the customer address is included in the original request
- Interceptors: one-way proxy that does not need to keep state (e.g. a log)
- The Ask Pattern: create a temporary one-off ask actor for receiving an email (you can use `import.pattern.ask` and the `?` send message method)
- Result Aggregation: aggregate results from multiple actors
- Risk Delegation: create a subordinate to perform a task that may fail
- Facade: used for translation, validation, rate limitation, access control, etc.

Here is a code sniplet using the ask and aggregation patterns:

```scala
def receive = {
  case Message =>
  val response = for {
    result1 <- (actor1 ? Message1).mapTo[MyClass1]
    result2 <- (actor2 ? Message2).mapTo[MyClass2]  // only called when result1 is received
  } yield ...

  response pipeTo sender
}
```

#### Scalability

Asynchronous messages passing enables vertical scalability (running the computation in parallel in the same node)
Location transparency enables horizontal scalability (running the computation on a cluster of multiple nodes)

Low performance means the system is slow for a single client (high latency)
Low scalability means the system is fast when used by a single client (low latency) but slow when used by many clients (low bandwidth)

With actors, scalability can be achieved by running several stateless replicas concurrently. The incoming messages are dispatched through routing. Routing actor(s) can either be stateful (e.g. round robin, adaptive routing) or stateless (e.g. random, consistent hashing)

- In Adaptive Routing (stateful), routees tell the router about their queue sizes on a regular basis.
- In Consistent Hashing (stateless), the router is splitting incoming messages based on some criterion

Stateful actors can be recovered based on a persisted state, but this means that 1) only one instance must be active at all time and 2) the routing is always to the active instance, buffering messages during recovery.

#### Responsiveness

Responsiveness is the ability to respond within a given time limit. If the goal of resilience is to be available, responsiveness implies resilience to overload scenarios.

Several patterns can be implemented to achieve responsiveness:

1) Exploit parallelism, e.g.

```scala
def receive = {
  case Message =>
    val result1 = (actor1 ? Message1).mapTo[MyClass1]
    val result2 = (actor2 ? Message2).mapTo[MyClass2]  // both calls are run in parallel

    val response = for (r1 <- result1, r2 <- result2) yield { ... }
    response pipeTo sender
}
```

2) Load vs. Responsiveness: When incoming request rate rises, latency typically rises. Hence the need to avoid dependency of processing cost on load and add parallelism elastically, resizing routers.

However, any system has its limits in which case processing gets backlogged.

3) The Circuit Breaker pattern (use akka `CircuitBreaker`) filters the number of requests that can come in when the sytem is under too much load, so that one subsystem being swamped does not affect the other subsystems.

4) With the Bulkheading patterns, one separates computing intensive parts from client-facing parts (e.g. on different nodes), the latter being able to run even if the backend fails.

```scala
Props[MyActor].withDispatcher("compute-jobs")  // tells to run the actor on a different thread

// If not, actors run on the default dispatcher
akka.actor.default-dispatcher {
  executor = "fork-join-executor"
  fork-join-executor {
    parallelism-min = 8
    parallelism-max = 64
    parallelism-factor = 3.0
  }
}

compute-jobs.fork-join-executor {
  parallelism-min = 4
  parallelism-max = 4
}
```

5) Use the Active-Active Configuration. Detecting failures takes time (usually a timeout). When this is not acceptable, instant failover is possible in active-active configurations where 3 nodes process the same request in parallel and send their reponses to the requester. Once the requester receives 2 matching results it considers it has its answer, and will proactively restart a node if it fails to respond within a certain time.


