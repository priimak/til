Magnet Pattern
==============

Motivation
----------

Consider a case where we have a certain parametrized type and we wish to pass object of that type to a
function where only certain parameter types are acceptable. Normally this is impossible to ensure at 
compile time due to a type erasure in Java and by proxy in Scala [1]. For example, let say we would 
like to have a function `count(...)` that accepts either list of integers or list of strings and in 
the first case returns sum of all integers in the list and in second case sum of length of each 
string. We can try do define these functions like so:
```scala
def count(strings: List[String]): Int = strings.map(_.length).sum
def count(strings: List[Int]): Int = strings.sum
```
Attempt to compile this code, however, will lead to an error
```
Error:(3, 9) double definition:
def count(strings: List[String]): Int at line 2 and
def count(strings: List[Int]): Int at line 3
have same type after erasure: (strings: List): Int
    def count(strings: List[Int]): Int = strings.sum
```
The reason for that is _type erasure_. After compilation both functions will have exactly the same 
signature which is naturally not allowed. We can remedy this problem by performing type check at the 
runtime.
```scala
def count[T](list: List[T]): Int = {
  if (list.isEmpty)
    0
  else if (list.head.isInstanceOf[Int])
    list.map(_.asInstanceOf[Int]).sum
  else if (list.head.isInstanceOf[String])
    list.map(_.asInstanceOf[String]).map(_.length).sum
  else
    throw new IllegalArgumentException
}
```
This does work but is unsatisfactory since error _can_ happen at runtime. An alternative solution that
preserves compile time type safety exist and is know as _magnet pattern_.

Magnet Pattern
--------------
First we define a _magnet_ trait.
```scala
sealed trait CountingMagnet {
  def apply(): Int
}
```
Define a single function
```scala
def count(countingMagnet: CountingMagnet): Int = countingMagnet()
```
And provide two implicit functions that can create instance of `CountingMagnet` from either 
of these two types `List[String]` and `List[Int]`.
```scala
implicit def counterOverListOfStrings(strings: List[String]) = new CountingMagnet {
  override def apply(): Int = strings.map(_.length).sum
}

implicit def counterOverListOfInts(ints: List[Int]) = new CountingMagnet {
  override def apply(): Int = ints.sum
}
```
When these two functions are in scope we can do calls
```scala
count(List("hello", "world!"))
count(List(1, 2, 3))
```
We, however, will not be able to compile code
```scala
count(List(List(), List()))
```
since there is no implicit conversion from `List[List[Any]]` to `CountingMagnet`.
If in this case we wish to simply count of number of lists then we can do so by creating new 
implicit function
```scala
implicit def counterOverListOfLists(ints: List[List[Any]]) = new CountingMagnet {
  override def apply(): Int = ints.map(_.length).sum
}
```
One obvious disadvantage of this pattern is that it is impossible to understand what types are 
acceptable when calling `count(...)` just by looking at the function signature. To know that one
needs to know exactly what implicit function are in scope that can coerce specific types into
`CountingMagnet`. 

References
----------
1. <a href="https://www.baeldung.com/java-type-erasure">Java Type Erasure</a>
2. <a href="http://blog.madhukaraphatak.com/scala-magnet-pattern/">Scala Magnet Pattern</a> blog post.
3. <a href="http://allaboutscala.com/tutorials/chapter-5-traits/the-magnet-pattern/">The Magnet Pattern</a> tutorial.

