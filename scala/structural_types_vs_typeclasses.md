Structural Types vs Typeclasses
==============================

Continuing on the application of [typeclasses](https://github.com/priimak/til/blob/master/scala/type_classes_1.md) 
to the problem of gluing different parts of the code together we now look at the
so called _Structural Types_ in Scala. Structural types can be thought of as a variant of [Duck Typing](https://en.wikipedia.org/wiki/Duck_typing) 
for the statically typed languages. Consider a following example `java.lang.String` and `scala.List` both have
method `length`. We would like to create a function that given anything that has method `length`
returns an "area" of the thing by squaring its _length_. While this particular computation is 
contrived it is easy to imagine something much more meaningful which is to happen inside of such 
function. One possible solution is to create two functions with two different aguments just to 
accommodate `String` and `List` types.
```scala
def area(x: String): Int = x.length * x.length
def area(x: List[Any]): Int = x.length * x.length
```
This is unsatisfactory for the several reasons. First, because we have code duplication and second (and 
more important) because these functions cannot accept other things that might possess method `length`. 

One possible solution is to introduce a type class
```scala
import simulacrum.typeclass

@typeclass trait Measurable[-A] {
  def len(x: A): Int
}

implicit val measurableString: Measurable[String] = (x: String) => x.length
implicit val measurableList: Measurable[List[Any]] = (x: List[Any]) => x.length

def area[A: Measurable](x: A): Int = Measurable[A].len(x) * Measurable[A].len(x)
```

This is a very flexible solution since it can essentially coerce other types into making them
look like they have ability compute length if they have some kind of measure that we care about
but perhaps under another name. For example that other type might have method `distance()`. To
accommodate it we introduce another `implicit val`. However, for objects that do have the same
method, for example `length` we can use structure type like so
```scala
def area(x: { def length: Int }): Int = x.length * x.length
```
The advantage of this is that it will accept anything that contains method `length`. Defined 
structure might become complex and thus not very suitable to be placed directly in the function
signature. In the case we can use type alias.
```scala
type ObjectWithLength = {
  def length: Int
}

def area(x: ObjectWithLength): Int = x.length * x.length
```
One of the difference from use typeclasses is that when method `area(ObjectWithLength)` is called
method `length` is accessed through reflection and thus is a bit slower. Lets find out exactly 
how much slower that is by using following program. 
```scala
import scala.util.Random
import simulacrum.typeclass

@typeclass trait Measurable[-A] {
  def len(x: A): Int
}

implicit val measurableString: Measurable[String] = (x: String) => x.length
implicit val measurableList: Measurable[List[Any]] = (x: List[Any]) => x.length

def reflection_length(x: {def length: Int}): Int = x.length
def typeclass_length[A: Measurable](x: A): Int = Measurable[A].len(x)

def stdev(xs: List[Long]): Double = {
  val av = xs.sum.toDouble / xs.length
  Math.sqrt(xs.map(x => Math.pow(x - av, 2)).sum / xs.length)
}

def main(args: Array[String]): Unit = {
  val rns = new Random()
  var times = (1 to 15).map(_ => {
    val lst: List[String] = (1 to 4000000).map(_ => s"hello ${rns.nextInt()}").toList
    val startAt = System.currentTimeMillis()
    val res = lst.map(reflection_length(_)).sum
    //val res = lst.map(typeclass_length(_)).length
    val endAt = System.currentTimeMillis()
    println(s"res = ${res} took ${endAt - startAt}")
    endAt - startAt
  })

  // drop first two iterations as they affected by JVM warm up
  times = times.drop(2)

  println(s"${times.sum.toDouble / times.length} ± ${stdev(times.toList)}")
}

```
With following build.sbt
```scala
name := "test"
version := "0.1"
scalaVersion := "2.13.3"
scalacOptions += "-Ymacro-annotations"
libraryDependencies += "org.typelevel" %% "simulacrum" % "1.0.0"
```
Running it on Ubuntu with Java 11.0.8 and disabled GC like so
```shell script
$ sbt -J-Xmx24G -J-XX:+UnlockExperimentalVMOptions -J-XX:+UseEpsilonGC run
```
We obtain following results.

| Type           | Average compute time | Relative time |
|----------------|----------------------|---------------|
| Structure Type | 81.5 &plusmn; 3.2    | 1.4           |
| Typeclass      | 56.8 &plusmn; 2.5    | 1             |

If calling method `typeclass_length(...)` takes 1 unit of time then calling 
`reflection_length(...)` takes 1.4 times of that. Thus, we can see use of Structure types
is about 40% slower than using typeclasses. Nevertheless, there are plenty of use cases where
this difference it not important and in those cases use of Structural Types is perfectly 
justifiable.