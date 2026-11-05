---
layout: post
title: 'Do not ask "Is A a B?": Inheritance and the Is-A Trap'
date: 2026-11-05
---
 
## When everyone agrees, and nobody listens

The history of inheritance is a history of experts cautioning against its use. In 1994, the Gang of Four listed "Favor object composition over class inheritance" as one of their early points in *Design Patterns*. In 2013, Sean Parent sharpened the rhetoric in his talk titled "Inheritance Is the Base Class of Evil". And that is just to name a few. Some newer languages have chosen to side-step the problem entirely: Go and Rust do not offer inheritance at all, using different mechanisms entirely to support runtime polymorphism and code reuse. 
 
And yet, from all I can tell, inheritance is still ubiquitous. I'm not just talking legacy code, but code written recently, in organizations that I have worked for, in open source projects, in technical articles, and your favorite LLM does it too. And it usually takes precisely the shape the critics warn about: deep, sprawling class hierarchies nobody can navigate, base classes nobody dares to touch, and derived classes that refuse to fulfill the contract the base class promised. 
 
I think a lot of this damage can be traced back to one particular guideline. Rather than explaining why inheritance is problematic, it tells developers when they should use it: *inheritance models an is-a relationship.* 
 
## A guideline or a definition?

Every introductory course to object-oriented programming teaches this in some form: if A *is a* B, A inherits from B; if A *has a* B, A contains a B as a member variable.
 
This is so pervasive that it has effectively stopped being a guideline. "Inheritance" and "is-a relationship" are routinely used as synonyms, and "inheritance models an is-a relationship" is stated as a fact rather than as advice. It is completely natural then to derive a design test: in order to decide whether A should inherit from B, check whether the statement "A is a B" is true. 
 
This heuristic has some obvious appeal. The test is short, needs no technical background, makes some degree of intuitive sense, and can be answered by looking at the problem domain instead of the code. It is not surprising then that this is the test that most developers use in practice to decide whether to use inheritance. But this is precisely the problem. It turns a technical decision about type hierarchies into a question about natural English - and natural English frequently leads to the wrong answer. 
 
## What does "is a" really mean?

In plain English, "A is a B" is a statement about classification: every A satisfies the description of a B. 

In programming, a type is not a description. It is a set of values plus a set of operations, and each operation comes with promises. Inheriting from B means guaranteeing that A keeps all of B's promises.
 
The problem is that those two things do often enough coincide to make the test feel reliable, and diverge often enough to make it dangerous. Let's quickly go through a few examples that pass the naive test, but where inheritance falls on its nose. 
 
### Square and Rectangle 
 
If there is one thing all geometry teachers can agree on, it's that a square is a rectangle. So: 
 
```cpp 
class Rectangle { 
 public: 
  virtual ~Rectangle() = default; 
  virtual void setWidth(double w) { width_ = w; } 
  virtual void setHeight(double h) { height_ = h; } 
  double area() const { return width_ * height_; } 
 private: 
  double width_ = 0.0, height_ = 0.0; 
}; 
 
class Square : public Rectangle { 
 public: 
  void setWidth(double w) override { Rectangle::setWidth(w); Rectangle::setHeight(w); } 
  void setHeight(double h) override { Rectangle::setWidth(h); Rectangle::setHeight(h); } 
}; 
 
void resize(Rectangle& r) { 
  r.setWidth(2.0); 
  r.setHeight(3.0); 
  assert(r.area() == 6.0); // fails for every Square 
} 
``` 
 
`Rectangle` promises that its width and height can be changed independently. `Square` cannot keep that promise, so `Square` is not a subtype of `Rectangle`, regardless of what the geometry teacher says. 
 
Classes similar to this are quite common, and there is a common bargaining tactic as well: Technically, the `area()` function never promised to return the product of the width and height you just set. But accepting this means weakening the base class's contract until almost nothing is left. A rectangle whose area may or may not follow from its explicitly set dimensions is not what any user of the class - such as the person writing the `resize()` function - expects, and not something you can reasonably program against. A contract that holds only because it was hollowed out until no subclass could break it is not a useful contract. 
 
Note though what happens if both classes did not offer ways to mutate their members: none of the setters, just a constructor and `area()`. Now `Square` is actually a reasonable subtype of `Rectangle`. The answer to "Is a square a rectangle?" did not change. The answer to "Could my Square class inherit from Rectangle?" did. 
 
Circle and Ellipse have the same problem, just with rounded corners: an ellipse can be stretched along one axis, a circle cannot. 
 
### Unit vectors 
 
Staying in the realm of geometry: a unit vector is a vector. If `UnitVector3` inherits from `Vector3`, it will likely get some useful functions for free, such as rotate. Unfortunately, it would also get functions like `operator+=`, `operator*=` and `setX()`, all of which can break the unit vector's class invariant that the length is 1. The subclass can override them to throw, to renormalize silently, or to do nothing, and each option surprises someone who was handed a `Vector3` reference. Unlike with the previous example, there is probably no way to salvage the inheritance, since a basic 3D-vector would always expose ways to alter its length. 
 
### Penguins 
 
A penguin is a bird. If `Bird` has a `fly()` function, `Penguin::fly()` has to either throw an error or quietly do nothing. A common band-aid is to introduce a `FlyingBird` intermediate class and derive only birds that can actually fly from it. This means that instead of using intuitive biological classification as our inheritance hierarchy, we have to reorganize it around capabilities instead. 
 
As you keep following this approach, your inheritance hierarchy takes on a much different shape. Consider bats, which fly but aren't birds, so flying actually should move into a separate `FlyingAnimal` interface. Birds lay eggs, but so do reptiles, fish and even the platypus, so we need an `EggLayingAnimal` interface as well. Penguins swim, but pigeons do not, so swimming becomes yet another capability. What started as a straightforward organization by biological taxonomy has turned into a collection of overlapping behavioral interfaces, with every concrete type inheriting from whichever combination happens to apply. 
 
As long as the number of capabilities remains manageable, the resulting design might even be workable, but notice how far we have moved from the original is-a relationship. The inheritance hierarchy no longer describes what something is, but what it can do. 
 
### The deeper problem

The "is-a" test casts an extremely wide net. Pretty much everything is something else, and usually multiple things. A manager is an employee, an employee is a person, and a person is a legal entity. A company is a legal entity, too. A savings account is an account, a joint account is an account, and a joint savings account is both. The real world is full of such classifications, and they overlap in ways that have little to do with what would be a good way to structure our code. Following the is-a test literally leads to inheritance hierarchies that are sprawling, deep, tangled, difficult to maintain, and typically do not contribute any useful capabilities to the code base. 
 
## The fine print

Now, none of this is news to people who define "is-a" carefully. The careful definitions do judge those cases correctly. In fact, most of the above examples come from textbooks or technical articles attempting to explain the subtleties of the "is-a" guideline.[^examples] The thing is that they do it by completely throwing out the idiomatic English language meaning of "is a". 
 
The more elaborate definitions typically go something like this: is-a actually means *behaves like* a, or *can be used wherever a base-class object is expected*, or *holds with respect to the operations of the type*. Under any of these definitions, a mutable square is not a mutable rectangle, and it may not inherit. 
 
But notice how this definition requires you to already know the operations and their contracts before you can answer the question whether "is-a" applies. At this point the guideline simply stops being helpful. It is not an appropriate shortcut to figuring out whether inheritance should be used, and instead all the work is done by the fine print. The phrase "is a" contributes nothing but a false sense that you might be able to give a quick answer via your intuition. 
 
If a guideline has to be read in a meaning that its own words do not have, it is time to retire the guideline. 
 
## The Liskov substitution principle

As a matter of fact, the fine print already has a name of its own: the Liskov substitution principle. One reason I suspect why this principle is not applied as enthusiastically in practice is that the original formulation is worded in a rather academic fashion: 
 
*Subtype Requirement*: Let ϕ(x) be a property provable about objects x of type T. Then ϕ(y) should be true for objects y of type S where S is a subtype of T. 
 
That doesn't quite roll off the tongue. More informally it means that, if A is a subtype of B, then anything you can rely on for a B, you can rely on for an A. Or, putting it in contract terms: 
 
- A's preconditions are no stronger than B's. 
- A's postconditions are no weaker than B's. 
- A preserves all of B's invariants. 
 
All of these are technical questions about operations and contracts, not about ontology. They correctly determine whether inheritance can be used for the mutable and immutable rectangles by looking at the functions and contracts these types expose, without requiring alternative names for those classes. They also give the right answer for `UnitVector3` and `Penguin` before the first bug report. 
 
If your inheritance relationships satisfy the LSP, inheritance is safe in the sense that code written against B will not be surprised by the behavior of an A. The next question is whether you want the rest of what inheritance implies. 
 
## What inheritance gives us

Inheritance is one of the strongest forms of coupling programming languages offer. A subclass depends on its base class's interface, its contracts, and, with implementation inheritance, on its class invariants. The base class cannot change without considering every subclass. 
 
This argument is sometimes levied against the use of inheritance in principle. But coupling is not bad in itself. Any abstraction in programming introduces some degree of coupling. Coupling is bad when it ties together things that may want to change independently. It costs nothing until one side needs to change without the other, or needs a feature that the other side cannot support. If two concepts that you are modelling in code truly are this tightly bound, and you have good reason to expect them to stay that way, the coupling simply states a fact about your design. In such a case, inheritance may significantly simplify your code as the alternative then is typically composition plus a big stack of forwarding functions. 
 
## Inherit interfaces, not implementation?

A popular compromise tries to keep the good parts of inheritance and drop the bad ones: inherit only from pure interfaces, never from classes with existing implementation. This rule definitely leads to better outcomes than plain "is-a", but it is still aimed at the wrong target. 
 
It removes implementation coupling, meaning there is no shared state across various layers of inheritance, and there can be no broken class invariants for member variables. A lot of genuinely terrible code becomes impossible to write. 
 
It also removes some properly useful or at least inoffensive patterns along with it. There are cases where it is possible to extend a class cleanly without interfering with either side's class invariants. There are also genuine use cases where default behavior that is common to all subclasses may be defined. When the rule is followed strictly, the end result may be a class that forwards twenty methods by hand to a member, which is an unfortunate solution if a simple inheritance design would have worked. 
 
On the other hand, interface inheritance is still subtyping, and the hierarchy might still be chosen by asking whether A is a B. Nothing in this approach prevents the following:
 
```java 
interface Bird { void fly(); } 
 
class Penguin implements Bird { 
  @Override 
  public void fly() { throw new UnsupportedOperationException(); } 
} 
``` 
 
This type of misdesign even made its way into Java's standard library. `Collections.unmodifiableList` returns a `List`, and `List` has an `add()` function. But since an unmodifiable list cannot support that operation, calling `add()` throws `UnsupportedOperationException`. 
 
Defenders might argue that this is not technically an LSP violation, because the documented contract of `List` explicitly allows implementations to leave certain operations, including `add()`, unsupported. But consider what happened here: the contract of `List` itself had to be weakened to accommodate implementations that cannot fulfill the expectations one would naturally have of a list. Most containers with equivalent capabilities in other programming languages have no such failure mode. Without unmodifiable lists, there would have been no necessity to make `add()` an optional operation in the first place. 
 
This is the penguin, in real life, in one of the most widely used libraries there is. 
 
The rule treats a common symptom: violated class invariants due to implementation coupling. The cause, choosing subtype relationships by natural language classification rather than by behavior, is not addressed.

## Actually check what you are getting
 
Inheritance is a tool with a specific purpose: strong coupling between types. Before using it, ask the questions that actually decide whether that is what you want: 
 
1. Does every promise B makes still hold for A, for every operation? 
2. Can B's contract stay as strong as its users expect? If you have to quietly drop some of B's promises before A can keep them, you are not modelling a subtype, you are redefining B to make room for A. 
3. Do A and B change together, and do you expect them to keep doing so? 
 
Much of the time, one or more of these answers will be no. This is the intended outcome - inheritance should be seen as a specialized tool that is to be used very deliberately. It is not a beginner feature to reach for as a default. But if all three answers are yes, use inheritance, implementation included if it helps. There is no need to apologize for this in code review. Strong coupling between things that are strongly coupled is not a smell, but accurate design. 
 
What you should never do is take the shortcut of asking "Is A a B?". This question returns far too many false positives.

[^examples]: The Square/Rectangle example is from [this article on the Liskov Substitution Principle](https://web.archive.org/web/20151128004108/http://www.objectmentor.com/resources/articles/lsp.pdf) by Robert "Uncle Bob" Martin. The Circle/Ellipse example has [its own Wikipedia page](https://en.wikipedia.org/wiki/Circle%E2%80%93ellipse_problem). The Penguin/Bird example is from Scott Meyers' book *Effective C++*.
