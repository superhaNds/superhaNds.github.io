---
layout: post
title: Algebraic Structures in Programming
---

Algebraic structures such as groups and monoids are quite often constructed
in programming with various language. Programmers have been implementing
such structures even though they did not know their formal definition
and they didn't need to since the patterns that arise are quite natural.

Some languages make use of such concepts explicitly and perhaps even have 
libraries named after their algebraic equivalent. Moreover, it is easier 
and more useful to define them in certain programming languages. Haskell 
is known for using extensive mathematical terminology, usually by referencing 
concepts from category theory present in the theoretical background of the
language.

Let us examine some basic algebraic structures from the point of view of
Java, a language in which it is rare to see such concepts implemented or used.
It is also worth noting that since Java 8, it is easier to define them
since we can now use functions as data through functional interfaces.

## Semigroups ##

One of the simplest structures is a [semigroup](https://en.wikipedia.org/wiki/Semigroup).
It a set `S` equipped with a binary operation closed under `S`. This operation has
one rule, namely it must be associative. Associativity says that the order in which
we perform the operation does not affect the result. Symbolically, if `*` is the name
of the operation and `x, y, z` are in our set, then `x * (y  * z) = (x * y) * z`.

In Java, we can define a semigroup as a class parameterized by a type with one attribute,
the operation. The libraries of `java.util.function` have already defined what we need.

```java
public class Semigroup<A> {
    
    private final BinaryOperator<A> bop; 

    public Semigroup(BinaryOperator<A> bop) {
        this.bop = bop;
    }
    
    public A mult(A a , A b) {
        return bop.apply(a, b);
    }
    
    public BinaryOperator<A> getOp() {
        return bop;
    }    
}
```

The parameter `A` represents the set while the `BinaryOperator` is a binary function that 
given two elements of `A`, it returns the result. The operation must be associative, but it
is of course impossible to verify this in a programming language (or is it?).

This structure captures a very general pattern, i.e., associativity of an operation. Associativity
can be quite useful in parallel computations because we would know the order does not affect the 
result. Examples are addition, multiplication on real numbers and string concatenation.

## Monoids ##

The next structure is called a monoid. A monoid is a semigroup with an identity or neutral element.
An identity element is an element `e` such that `x * e = x = e * x`. In other words, it does not
change the value.

For an implementation, we can extend the `Semigroup` class and add a single element that represents
the identity element.

```java
public class Monoid<A> extends Semigroup<A> {
    
    private final A id;
    
    public Monoid(A id, BinaryOperator<A> bop) {
        super(bop);
        this.id = id;
    }
    
    public A getId() {
        return id;
    }
}
```
In multiplication, the identity is `1` whereas in addition, it is `0`. In string concatenation,
it's the empty string. Let's define a few instances.

```java
Monoid<Integer> intMult = new Monoid<>(1, (a, b) -> a * b);
Monoid<String> stringConcat = new Monoid<>("", String::concat);
```

The functional interfaces are used here to define a lambda expression or anonymous function.
This is very natural in functional programming, but might seem strange for now. The point is
we want to pass around functions as data too and this sets the stage for very powerful
high level programming. For the string monoid, we use method inference.

Booleans are also monoids under logical OR and AND.

```java
Monoid<Boolean> boolAnd = new Monoid<>(true, (p, q) -> p && q);
Monoid<Boolean> boolOr = new Monoid<>(false, (p, q) -> p || q);
```

## Groups ##

Lastly, we will see groups. A group is a monoid with an inverse. Elements in the carrier set
should have a unique inverse, which when combined with the binary operation returns the
identity element. Symbolically, `x * inv(x) = e = inv(x) * x`.

We model the inverse by a unary function that returns the inverse element of the argument.

```java
public class Group<A> extends Monoid<A> {
    
    private final UnaryOperator<A> inv;
    
    public Group(UnaryOperator<A> inv, A id, BinaryOperator<A> bop) {
        super(id, bop);
        this.inv = inv;
    }

    public UnaryOperator<A> getInverse() {
        return inv;
    }
}
```

We can create the group with addition, the inverse is the negation of a given integer.

```java
Group<Integer> intGroup = new Group<>(x -> x * (- 1), 0, (a, b) -> a + b);
```

## Operations using these structures

What can we do with these algebraic structures? They are very general and abstract so
we don't talk about specific elements of data types since they are parametrically 
polymorphic. There are a couple interesting patterns that now have nice implementations
in Java.

An important and useful application is folding. Folding or reducing takes a collection
of elements and combines all of them somehow to return a single element. All these
structures have a binary operator, thus we can fold a list using it.

```java
public static <A> A sconcat(Semigroup<A> sem, List<A> as) {
    return as.parallelStream()
             .reduce(sem.getOp())
             .orElse(null);        
}
```

Given a semigroup and a list of elements in its set, we can reduce it by applying
the operator succesively to all elements. Associativity of the operator allows us
to use parallelism, since the order does not affect the result (assuming these
are pure computations). Here, null will be returned if there are no elements.

Another general pattern is to apply the operator to a single element n times

```java
public static <A> A stimes(Semigroup<A> sem, A a, int n) {
    return sconcat(sem, Collections.nCopies(n, a));
}
```

Finally, we can reduce monoids using the identity element.

```java
public static<A> A mconcat(Monoid<A> mon, List<A> as) {
    return as.parallelStream()
             .reduce(mon.getId(), mon.getOp());
}
```

Reduce is called fold in Haskell and is very common to use it. The function `mconcat`
can be used to implement a summing or product function of a list of numbers. Since it
would start adding the elements on the identity element.

Next time you are faced with implementing an EDSL or something that has you defining
operators between two elements of a type, consider if it is associative and if there
is an identity. Usually, operators that add sequences of some kind are associative,
for example, in turtle graphics, it is common to define operators that combine
programs and those form at least a semigroup. Moreover, combinators used for
parsing libraries have algebraic structures.