Type Classes in Scala
=====================

Motivation
----------
Consider a case where we use an external library that provides objects `JPEGImage`
<a id="JPEGImage"></a>
```scala
case class JPEGImage(width: Int, height: Int)
```
To use these objects we have a method `numberOfPixels(JPEGImage)` that accepts
instances of this class and computes total number of pixels
```scala
def numberOfPixels(image: JPEGImage): Int =  image.width * image.height
```
Now we bring another library that provides class `TIFFImage` which we want to 
use along side with `JPEGImage`.
<a id="TIFFImage"></a>
```scala
case class TIFFImage(private val x: Int, private val y: Int) {
  def dimensions(): java.awt.Dimension = new java.awt.Dimension(x, y)
}
```
These two classes do not share a common ancestor besides maybe `Any` or `java.lang.Object`.
Additionally `TIFFImage` unlike `JPEGImage` does not provide direct access to `width`
and `height` of the image. Instead it provides method `dimensions()`.

Thus the question is can we modify our code and method `numberOfPixels(...)` in particular 
so that it can accept either one of these two classes. This is particularly important
because we are unlikely to modify these external libraries and thus have to work around
whatever functionality they provide. We also want to preserve compile-time type safety
of our code, which means that we do not want `numberOfPixels(...)` to accept arbitrary
objects and then use reflection to find instance of what class was actually passed. Such
code would lead to runtime exceptions which we obviously would like to avoid.

Type Classes
-----------------------------
Type classes offer a solution of the above mentioned problem with minimal additional 
code [[1](#ref1),[2](#ref2),[3](#ref3)].
Word "_Classes_" here refers to the concept that several very different types can be made
to belong to a certain class of objects, i.e. a _class of things that posses certain common
properties_. Scala does not have a dedicated syntax for type classes, however, we can use
_implicits_ and _implict scope_ [[4](#ref4)] to emulate such behaviour.

In Scala, type class is defined as trait that posses certain functions and an implicit 
objects or values that instances of this trait for all types that are made to be a member 
of this type class.

In our case we define type class `ImageDimensions` trait with a function that for a given
image can provide us its dimensions
```scala
trait ImageDimensions[A] {
  def getDimensions(image: A): java.awt.Dimension
}
```
Now we need to make two `implicit` implementations of this trait. One for `JPEGImage` and
another one for `TIFFImage`.
```scala
object ImageDimensions {
  implicit object JPEGImageDimensions extends ImageDimensions[JPEGImage] {
    override def getDimensions(image: JPEGImage): Dimension =
      new java.awt.Dimension(image.width, image.height)
  }

  implicit object TIFFImageDimensions extends ImageDimensions[TIFFImage] {
    override def getDimensions(image: TIFFImage): Dimension = image.dimensions()
  }
}
```
We place them inside of companion object `ImageDimensions` which leads to importing
of `JPEGImageDimensions` and `TIFFImageDimensions` into implicit scope every
time `ImageDimensions` is imported
```scala
import ImageDimensions
```
We can now modify function `numberOfPixels(...)` like so
```scala
def numberOfPixels[A](image: A)(implicit imageDimensions: ImageDimensions[A]): Int =
  imageDimensions.getDimensions(image).width * imageDimensions.getDimensions(image).height
```
At the first glance it appears that you can pass any object `A` into this method. That, however, is 
not the case! You can only pass instances of class `A` for which there exist in the implicit scope
an object or a value of type `ImageDimensions[A]`. In our case only instances of 
`JPEGImage` or `TIFFImage` will be accepted by this method. Thus we can now do
```scala
val a = numberOfPixels(JPEGImage(800, 600))
val b = numberOfPixels(TIFFImage(456, 500))
```
There is another form of defining `numberOfPixels(...)` that accomplishes exactly the same but
makes the fact that we are restricting type `A` to a particular class (a type class so to speak) 
more explicit.
```scala
def numberOfPixels[A: ImageDimensions](image: A): Int = {
  val imageDimensions = implicitly[ImageDimensions[A]]
  imageDimensions.getDimensions(image).width * imageDimensions.getDimensions(image).height
}
```
This is known as _context bound syntax_ [[5](#ref5)]. 
It means that type `A` that can be passed to this function can only be such for which there exist an
implicit value of type `ImageDimensions[A]`. Internally in the function body to obtain this value we have to call 
`implicitly[ImageDimensions[A]]` which pulls instance of `ImageDimensions[A]` from implicit context. 

This can be simplified farther by providing easier access to the implicit instance of `ImageDimensions[A]`.
 That is done by adding method `apply(...)` to the `ImageDimensions` companion object
```scala
object ImageDimensions {
  ...
  def apply[A](implicit imageDimensions: ImageDimensions[A]): ImageDimensions[A] = imageDimensions
}
```
Then function `numberOfPixels(...)` can be rewritten like so
```scala
def numberOfPixels[A: ImageDimensions](image: A): Int =
  ImageDimensions[A].getDimensions(image).width * ImageDimensions[A].getDimensions(image).height
```
Additionally we can coerce both `JPEGImage` and `TIFFImage` classes into another class that 
has method `getDimensions()` by introducing implicit class
```scala
implicit class ImageDimensions[A: ImageDimensions](image: A) {
  def getDimensions(): java.awt.Dimension = ImageDimensions[A].getDimensions(image)
}
```
Then we can call method `getDimensions()` on the original image classes as if they have this method
```scala
val d = TIFFImage(800, 600).getDimensions()
```

Complete Example
----------------
Below is a complete code in the `xyz.priimak.example.image` package that defines this type class.
We assume that we already have classes [JPEGImage](#JPEGImage) and [TIFFImage](#TIFFImage) provided 
by perhaps some other third-party libraries.
```scala
package xyz.priimak.example

import java.awt.Dimension

package object image {
  trait ImageDimensions[A] {
    def getDimensions(image: A): java.awt.Dimension
  }

  implicit class ImageDimensionsOps[A: ImageDimensions](image: A) {
    def getDimensions(): java.awt.Dimension = ImageDimensions[A].getDimensions(image)
  }

  object ImageDimensions {
    implicit object JPEGImageDimensions extends ImageDimensions[JPEGImage] {
      override def getDimensions(image: JPEGImage): Dimension = new Dimension(image.width, image.height)
    }

    implicit object TIFFImageDimensions extends ImageDimensions[TIFFImage] {
      override def getDimensions(image: TIFFImage): Dimension = image.dimensions()
    }

    def apply[A](implicit imageDimensions: ImageDimensions[A]): ImageDimensions[A] = imageDimensions
  }
}
```

References
----------
1. <a id="ref1"></a>[_How to make ad-hoc polymorphism less ad hoc_](http://homepages.inf.ed.ac.uk/wadler/papers/class/class.ps) - 
    original article that introduced type-classes in Haskell.
2. <a id="ref2"></a>[_Tutorial on typeclasses in Scala_](https://scalac.io/typeclasses-in-scala/) -  another good tutorial 
on type classes in Scala.
3. <a id="ref3"></a>[_Note on Type class_](https://nrinaudo.github.io/scala-best-practices/definitions/type_class.html) - a very 
    brief introduction to Scala type classes.
4. <a id="ref4"></a>[_Revisiting implicits without import tax_](http://eed3si9n.com/revisiting-implicits-without-import-tax) - on
    understanding of implicit scope.
5. <a id="ref5"></a>[_What is context bound syntax_](https://docs.scala-lang.org/tutorials/FAQ/context-bounds.html#what-is-a-context-bound) -
    from the offical documentation.
