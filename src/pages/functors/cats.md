## Functors in Cats

Let's look at the implementation of functors in Cats.
We'll follow the usual pattern of looking at
the three main aspects of the implementation:
the *type class*, the *instances*, and the *interface*.

### The *Functor* Type Class

The functor type class is [`cats.Functor`][cats.Functor].
We obtain instances using the standard `Functor.apply`
method on the companion object.
As usual, default instances are arranged by type in
the [`cats.instances`][cats.instances] package:

```tut:book:silent
import cats.Functor
import cats.instances.list._
import cats.instances.option._
```

```tut:book
val list1 = List(1, 2, 3)
val list2 = Functor[List].map(list1)(_ * 2)

val option1 = Option(123)
val option2 = Functor[Option].map(option1)(_.toString)
```

`Functor` also provides the `lift` method,
which converts a function of type `A => B`
to one that operates over a functor and has type `F[A] => F[B]`:

```tut:book
val func = (x: Int) => x + 1

val lifted = Functor[Option].lift(func)

lifted(Option(1))
```

### *Functor* Syntax

The main method provided by the syntax for `Functor` is `map`.
It's difficult to demonstrate this with `Options` and `Lists`
as they have their own built-in `map` operations.
If there is a built-in method it will always be called
in preference to an extension method.
Instead we will use *functions* as our example:

```tut:book:silent
import cats.instances.function._
import cats.syntax.functor._
```

```tut:book
val func1 = (a: Int) => a + 1
val func2 = (a: Int) => a * 2
val func3 = func1.map(func2)

func3(123)
```

Other methods are available but we won't discuss them here.
`Functors` are more important to us
as building blocks for later abstractions
than they are as a tool for direct use.

### Instances for Custom Types

We can define a functor simply by defining its map method.
Here's an example of a `Functor` for `Option`,
even though such a thing already exists in [`cats.instances`][cats.instances]:

```tut:book:silent
implicit val optionFunctor = new Functor[Option] {
  def map[A, B](value: Option[A])(func: A => B): Option[B] =
    value.map(func)
}
```

The implementation is trivial---we simply call `Option's` `map` method.

Sometimes we need to inject dependencies into our instances.
For example, if we had to define a custom `Functor` for `Future`,
we would need to account for
the implicit `ExecutionContext` parameter on `future.map`.
We can't add extra parameters to `functor.map`
so we have to account for the dependency when we create the instance:

```tut:book:silent
import scala.concurrent.{Future, ExecutionContext}

implicit def futureFunctor(implicit ec: ExecutionContext) =
  new Functor[Future] {
    def map[A, B](value: Future[A])(func: A => B): Future[B] =
      value.map(func)
  }
```

### Exercise: Branching out with Functors

Write a `Functor` for the following binary tree data type.
Verify that the code works as expected on instances of `Branch` and `Leaf`:

```tut:book:silent
sealed trait Tree[+A]
final case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
final case class Leaf[A](value: A) extends Tree[A]
```

<div class="solution">
The semantics are similar to writing a `Functor` for `List`.
We recurse over the data structure, applying the function to every `Leaf` we find.
The functor laws intuitively require us to retain the same structure
with the same pattern of `Branch` and `Leaf` nodes:

```tut:book:silent
import cats.Functor
import cats.syntax.functor._

implicit val treeFunctor = new Functor[Tree] {
  def map[A, B](tree: Tree[A])(func: A => B): Tree[B] =
    tree match {
      case Branch(left, right) =>
        Branch(map(left)(func), map(right)(func))
      case Leaf(value) =>
        Leaf(func(value))
    }
}
```

Let's use our `Functor` to transform some `Trees`:

```tut:book:fail
Branch(Leaf(10), Leaf(20)).map(_ * 2)
```

Oops! This is the same invariance problem we saw with `Monoids`.
The compiler can't find a `Functor` instance for `Leaf`.
Let's add some smart constructors to compensate:

```tut:book:silent
def branch[A](left: Tree[A], right: Tree[A]): Tree[A] =
  Branch(left, right)

def leaf[A](value: A): Tree[A] =
  Leaf(value)
```

Now we can use our `Functor` properly:

```tut:book
leaf(100).map(_ * 2)

branch(leaf(10), leaf(20)).map(_ * 2)
```
</div>
