## At the Heart of `for-yield`

**Markus Günther**

Rhein-Main Scala Enthusiasts (2016/02/02)

[markus.guenther@gmail.com](mailto:markus.guenther@gmail.com) | [habitat47.de](http://www.habitat47.de) | [@mguenther](https://twitter.com/mguenther)

---

### Suppose we want to collect course invitations for users based on a set of available courses.

----

## Imperative Java Version

```java
public List<Invitation> collectInvitations() {
    List<String> courses = Arrays.asList(
        "Functional Programming Principles in Scala",
        "Principles of Reactive Programming");

    List<Invitation> invitations = new ArrayList<>();

    for (int i = 0; i < courses.size(); i++) {
        String course = courses.get(i);
        Invitation invitation = new Invitation(
            "Dear user, we'd like to invite to " +
            "our course " + course + ".");
        invitations.add(invitation);
    }
    return Collections.unmodifiableList(invitations);
}
```

----

## Translation to Scala

```scala
def collectInvitations(): List[Invitation] = {
  val courses = List(
    "Functional Programming Principles in Scala", 
    "Principles of Reactive Programming")
  var invitations = List()

  for (i <- 0 until courses.length) {
    val course = course(i)
    val invitation = new Invitation(
      "Dear user, we'd like to invite you to our " +
      "course " + course + ".")
    invitations ::= invitation
  }

  invitations
}
```

----

## Extracting the `map` aspect

```scala
def map(courses: List[String], f: (String) => Invitation) = {
  var invitations: List[Invitation] = List()
  for (i <- 0 until courses.length) {
    invitations ::= f(courses(i))
  }
  invitations
}

def collectInvitations(): List[Invitation] = {
  val courses = List(...)
  val f = { (course: String) => new Invitation(
    "Dear user, we'd like to invite you " +
    "to our course " + course + ".") }
  map(courses, f)
}
```

----

## Using `map` from `List[+T]`

```scala
def collectInvitations(): List[Invitation] = {
  val courses = List(
    "Functional Programming Principles in Scala", 
    "Principles of Reactive Programming")
  courses.map { (course: String) => new Invitation(
    "Dear user, we'd like to invite you " +
    "to our course " + course + ".") }
}
```

---

### Suppose we want to explicitly address a user by her name.

----

### The imperative version does not compose well.

```java
public List<Invitation> collectInvitationsForUsers() {
  List<String> courses = Arrays.asList(...);
  List<String> users = Arrays.asList("Brian", "Miranda", "Mark");
  List<Invitation> invitations = new ArrayList<>();

  for (int i = 0; i < users.size(); i++) {
    for (int j = 0; j < courses.size(); j++) {
      Invitation invitation = new Invitation(
        "Dear " + user.get(i) + ", we'd like to invite " +
        "you to our course " + course.get(j) + ".");
      invitations.add(invitation);;
    }
  }
  return Collections.unmodifiableList(invitations);
}
```

----

## Are nested `map`s better?

```scala
def collectInvitations(): List[Invitation] = {
  val courses = List(...)
  val users = List("Brian", "Miranda", "Mark")

  users.map { (name: String) => {
      courses.map { (course: String) => new Invitation(
        "Dear " + name + ", we'd like to invite you " +
        "to our course " + course + ".") }
    }
  }
}
```

----

#### Well, for starters, this doesn't even compile. :)

```
scala> error: type mismatch;
found   : List[List[Invitation]
required: List[Invitation]
          users.map { (name: String) =>
```

----

### `flatten` removes the nested list.

```scala
def collectInvitations(): List[Invitation] = {
  val courses = List(...)
  val users = List("Brian", "Miranda", "Mark")

  users.map { (name: String) => {
      courses.map { (course: String) => new Invitation(
        "Dear " + name + ", we'd like to invite you " +
        "to our course " + course + ".") }
    }
  }.flatten
}
```

----

## map + flatten = flatMap

----

### Use nested `flatMap`s with a **final** `map`!

```scala
def collectInvitations(): List[Invitation] = {
  val courses = List(...)
  val users = List("Brian", "Miranda", "Mark")

  users.flatMap { (name: String) =>
    courses.map { (course: String) => 
      new Invitation(
        "Dear " + name + ", we'd like to invite " +
        "you to our course " + course + ".") 
    }
  }
}
```

----

## `for`-comprehensions are syntactic sugar for this.

```scala
def collectInvitations(): List[Invitation] = {
  val courses = List(...)
  val users = List("Brian", "Miranda", "Mark")

  for {
    course <- courses
    user   <- users
  } yield {
    new Invitation(
      "Dear " + user + ", we'd like to invite " +
      "you to our course " + A + ".")
  }
}
```

----

### A simple `for-yield`-combination

```scala

for { x <- List(1,2,3) } yield { x*x }
```

### will be desugared to `map`

```scala

List(1,2,3).map {x => x*x}
```

----

### A `for`

```scala
for(x <- List(1,2,3))
  println(x)
```

### will be desugared to `foreach`

```scala

List(1,2,3).foreach{x => println(x)}
```

----

### Nested `for-yield`s

```scala
for(
  x <- List(1,2,3); 
  y <- List(4,5,6))
yield x*y
```

### will be desugared to `flatMap`s and `map`s

```scala
List(1,2,3).flatMap { x =>
  List(4,5,6).map { y =>
    x*y
  }
}```

---

### `for`-comprehensions work with every type that implements `map` and `flatMap`

##### and `filter`, `withFilter`, `foreach`

----

**Option[+T]**

**Try[+T]**

**Future[+T]**

**Either[+A, +B]**

**List[+A]**

**Stream[+A]**

