---
layout: post
title: Scala 2.12&#58; Changes in Future API
---

There are a couple of changes in Scala Future API with 2.12.x release. I believe new `transform` method and `never` object are the most important improvements. In this post I will explain how they improve the usage of Futures.

### transform

Let's assume that we have a `Future[Int]` and we need to

  - increment our `Int` if our Future is successful
  - map `CannotGetException` to `defaultValue`
  - map other exceptions to `OtherException`

The problem with the old `transform` is that it cannot transform a failed `Future` to a successful one. It maps a `Throwable` to another `Throwable` in case of a failed `Future`. That's why we cannot do this by using old `transform`, we could solve this with `map` and `recover` like below:

{% highlight scala %}
val originalFuture: Future[Int] = getIntFuture()
val newFuture: Future[Int] = originalFuture
  .map(value => value + 1)
  .recover {
    case t: CannotGetException => defaultValue
    case _ => throw new OtherException
  }
{% endhighlight %}

New `transform` method differs from the old one in that it takes a function from `Try[T]` to `Try[S]` instead of taking two separate functions for successful and failed cases.

Thanks to this difference we can write with only new `transform` rather than using two combinators:

{% highlight scala %}
val originalFuture: Future[Int] = getIntFuture()
val newFuture: Future[Int] = originalFuture transform {
  case Success(value) => Success(value + 1)
  case Failure(t: CannotGetException) => Success(defaultValue)
  case _ => Failure(new OtherException)
}
{% endhighlight %}

In addition, new `transform` method is also used to [implement](https://github.com/scala/scala/blob/v2.12.0-RC1/src/library/scala/concurrent/Future.scala#L287) some of the existing methods of 2.12 API like `recover` and `map`:

{% highlight scala %}
def map[S](f: T => S)
  (implicit executor: ExecutionContext): Future[S] =
  transform(_ map f)
{% endhighlight %}

### never
[`Future.never`](http://www.scala-lang.org/api/2.12.x/scala/concurrent/Future$$never$.html) is a never completed Future. It is a `Future[Nothing]` that overrides all necessary methods in order not to be completed. For example its `map` implementation equals to `this` and doesn't use parameter `f` to not keep a reference to it.

{% highlight scala %}
override def map[S](f: Nothing => S)
  (implicit executor: ExecutionContext): Future[S] = this
{% endhighlight %}

An example use case and explanation about how it prevents memory leaks can be found on [Viktor Klang's blog](https://github.com/viktorklang/blog).
