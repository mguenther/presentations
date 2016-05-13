## Property-Based Testing

Markus Günther

[markus.guenther@gmail.com](mailto:markus.guenther@gmail.com) | [habitat47.de](http://www.habitat47.de) | [@mguenther](https://twitter.com/mguenther)

---

### Part I: A Motivating Example

----

### For some reason we are tasked with the implementation of a custom `add` function

----

### Of course, we want to test our implementation

----

```scala
class AddUnitTest extends FlatSpec with Matchers {

  "Adder" should "yield 4 when I add 1 + 3" in {
    add1(1, 3) shouldBe 4
  }

  "Adder" should "yield 4 when I add 2 + 2" in {
    add1(2, 2) shouldBe 4
  }
}
```

----

### Some time later, ops comes back with errors in production wrt. `add`

----

```scala
class AddUnitTest extends FlatSpec with Matchers {

  [...]

  "Adder" should "yield 2 when I add -1 + 3" in {
    add1(-1, 3) shouldBe 2
  }
}
```

----

### Boom!

```
[info] Adder
[info] - should yield 2 when I add -1 + 3 *** FAILED ***
[info]   4 was not equal to 2 (AddUnitTest.scala:17)
[info] ScalaCheck
[info] Passed: Total 0, Failed 0, Errors 0, Passed 0
[info] ScalaTest
[info] Run completed in 212 milliseconds.
[info] Total number of tests run: 3
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 2, failed 1, canceled 0, ignored 0, pending 0
[info] *** 1 TEST FAILED ***
```

----

### What happened?

----

### Let us take a look at the implementation

```scala
object Adder {

  def add1(a: Int, b: Int): Int =
    4
}
```

----

### TDD Best Practices

**Write the minimal code that will make the test pass.**

* Just write code that makes the test pass.
* Code written at this stage will not be 100% final.
* Do not write the perfect code at this stage.

---

### Part II: Property-Based Testing

----

### What are the requirements for `add`?

----

### Property: Adding two numbers should not depend on parameter order

```scala
forAll {
  (a: Int, b: Int) => {
    add(a, b) == add(b, a)
  }
}
```

----

### Refactoring `add` - take #1

```scala
object Adder {

  def add1(a: Int, b: Int): Int =
    a * b
}
```

----

### Property: Adding 1 twice to a number is the same as adding 2 once to the same number.

```scala
forAll {
  (a: Int) => {
    add(add(a, 1), 1) == add(a, 2)
  }
}
```

----

### Let's see if our tests pass

```
+ adding two numbers should not depend on parameter order: OK [...]
! adding 1 twice is the same as adding 2 once: Falsified      [...]
[info] > ARG_0: 1
ScalaCheck
Failed: Total 2, Failed 1, Errors 0, Passed 1
```

----

### Refactoring `add` - take #2

```scala
object Adder {

  def add1(a: Int, b: Int): Int =
    0
}
```

----

### Tests pass, but we do not check if the result is somehow connected to the input.

----

### Property: Adding zero is the same as doing nothing

```scala
forAll {
  (a: Int) => {
    add(a, 0) == a
  }
}
```

---

### A motivating example

```scala
object Fizzbuzz extends App {

  def fizzbuzz(n: Int): String = n match {
    case _ if n % 15 == 0 => "FizzBuzz"
    case _ if n % 3 == 0 => "Fizz"
    case _ if n % 5 == 0 => "Buzz"
    case _ => n.toString
  }

  println((1 to 100).map(fizzbuzz(_)).mkString(" "))
}
```

----

### How do we test our implementation?

----

```scala
class FizzbuzzUnitSpec extends FlatSpec with Matchers {
  "Fizzbuzz" should 
    "yield 'Fizz' for a number that is not divisible by 5 but by 3" in {
      Fizzbuzz.fizzbuzz(3) shouldBe "Fizz"
  }
  "Fizzbuzz" should 
    "yield 'Buzz' for a number that is not divisible by 3 but by 5" in {
      Fizzbuzz.fizzbuzz(5) shouldBe "Buzz"
  }
  "Fizzbuzz" should 
    "yield 'FizzBuzz' for a number that is divisible by 15" in {
      Fizzbuzz.fizzbuzz(15) shouldBe "FizzBuzz"
  }
  "Fizzbuzz" should 
    "yield identity for a number that is not divisible by 3 or by 5" in {
      Fizzbuzz.fizzbuzz(1) shouldBe "1"
  }
}
```

----

### What about these?

* 38
* 60
* 99
* 12932
* 293402
* -23424
* 0
* `Int.MaxValue`

----

### What are the properties that `fizzbuzz` must satisfy?

* Any number that is wholly divisible by 3 must be translated to `Fizz`.
* Any number that is wholly divisible by 5 must be translated to `Buzz`.
* Any number that is wholly divisible by 15 must be translated to `FizzBuzz`.
* All other numbers are returned as-is.

----

### How can we generate such numbers?

```scala
val numberGen = Gen.choose(Int.MinValue / 15, Int.MaxValue / 15)
val divisibleByThreeNotFive = numberGen
   .suchThat(n => n % 5 != 0)
   .map(n => n * 3)
```

----

### `Gen` provides a rich set of basic generators

* `choose`
* `oneOf`
* `lzy`
* `listOf`
* `listOfN`
* `alphaNum`
* `alphaStr`
* `identifier`
* ...

----

### Generators

```scala
prop((n: Int) => (fizzbuzz(n) mustEqual "Fizz"))
  .setGen(divisibleByThreeNotFive)
```

----

### The output should look like this.

```
[info] FizzbuzzSpec
[info] 
[info]   FizzBuzz should
[info]     + yield 'Fizz' for a number wholly divisible by 3
[info] 
[info] Total for specification FizzbuzzSpec
[info] Finished in 96 ms
[info] 1 examples, 100 expectations, 0 failure, 0 error
[info] 
[info] ScalaCheck
[info] Passed: Total 0, Failed 0, Errors 0, Passed 0
```

---

### How can we turn this into a property-based test?

```scala
def dbShouldReturnPreviouslySavedUser = {
  val user = User("jon doe", 64293)
  val userId = db.insert(user)
  db.load(id) == Some(user)
}
```

----

### Replace free variables with properties.

```scala
forAll { (name: String, postcode: Int) =>
  val user = User(name, postcode)
  val userId = db.insert(user)
  db.load(id) == Some(user)
}
```

----

### scalacheck knows how to generate `String`s and `Int`s

```scala
forAll { (name: String, postcode: Int) =>
  val user = User(name, postcode)
  val userId = db.insert(user)
  db.load(id) == Some(user)
}
```

* Invalid username: `""`
* Invalid postcode: -1

----

### Use types to be more precise!

```scala
class Postcode private(val value: Int) extends AnyVal

object Postcode {
  
  def fromInt(i: Int): Option[Postcode] =
    if (i >= 10000 || i < 100000)
      Some(new Postcode(i))
    else
      None
}
```

----

### Invest in generators for your domain objects

```scala
def genUsername: Gen[Username] =
  Gen.nonEmptyListOf(Gen.alphaChar)
     .map(Username.fromString)

def genPostcode: Gen[Postcode] =
  Gen.choose(10000, 99999)
     .map(Postcode.fromInt)
```

----

### With this scalacheck knows how to generate `Username`s and `Postcode`s

```scala
forAll { (u: Username, p: Postcode) => ... }

forAll { (u1: Username, u2: Username) => ... }

forAll { (p: List[Postcode] => ... }
```