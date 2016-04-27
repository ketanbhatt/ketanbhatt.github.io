---
title:	"Preventing Rubber Ducks from Flying: The Strategy Pattern"
date:	2016-04-26
excerpt: What do you do when the OO concepts you have abided by since your school days land you in trouble? You relearn them.
---
> _Master:_ So Grasshopper, should effort go into reuse **above** maintainability and extensibility?
>
> _Student:_ Master, I believe that there is truth in this.
>
> _Master:_ I can see that you still have much to learn.
>
> -- -- <cite>Head First Design Patterns</cite>

#### The Strategy Pattern: 
Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from the clients that use it.

#### Principles:
1. Encapsulate what varies: Identify the aspects of your application that vary and separate them from what stays the same.
2. Favor composition over inheritance.
3. Program to interfaces, not implementations.

## when will I need it?
So you are a developer and you are making a _Duck Simulator_* program. You are a nice person and so you use Inheritance. You have a `Duck` superclass that defines some attributes and methods. This class is inherited by other special `DuckType` classes. Like so:

{% highlight python %}
class Duck(object):
	def quack():
		print "Quack! Quack!"

	def swim():
		print "Yaay I am swimming!"

	def display():
		raise NotImplementedError

class MallardDuck(Duck):
	def display():
		print "I look like a Mallard"

class RedheadDuck(Duck):
	def display():
		print "I look like a Redhead"
{% endhighlight %}

`quack()` and `swim()` are implemented in the superclass while `display()` is implemented in the child classes as each duck looks different. 

Now **you are asked to add a new feature, the ducks can now fly**. Because you used inheritance, you just added the method `fly()` to the superclass :D

{% highlight python %}
class Duck(object):
	...

	def fly():
		print "I am flying!"
{% endhighlight %}

**But that proved disastrous** because a certain Duck that wasn't supposed to be flying started doing air acrobatics.

{% highlight python %}
class RubberDuck(Duck):
	def display():
		print "I look like a Rubber Duck"

	def quack():
		# Rubber duck doesn't quack, it squeaks
		print "Squeak! Squeak!"
{% endhighlight %}

So what? You can easily override `fly()` in `RubberDuck`. But what if we add another Duck `WoodenDuck`? You will again override `fly()` and `quack()` (because wooden ducks don't quack). 

_What you thought was great for reuse (Inheritance) turned out to be a nightmare for maintenance._


## Strategy Pattern to the rescue!
Time to use some [design principles](#principles).

We identify that `flying` and `quacking` are varying behaviors, so we will abstract them out to interfaces. 

We will create `FlyBehavior` and `QuackBehavior` superclasses. Each behavior will have a set of classes associated with it. We have this structure now:

{% highlight python %}
# Flying Behavior
class FlyBehavior(object):
	def fly():
		raise NotImplementedError

class FlyWithWings(FlyBehavior):
	def fly():
		print "I am flying!"

class FlyNoWay(FlyBehavior):
	def fly():
		print "I can't fly"

# Quacking Behavior
class QuackBehavior(object):
	def quack():
		raise NotImplementedError

class Quack(QuackBehavior):
	def quack():
		print "Quack! Quack!"

class Squeak(QuackBehavior):
	def quack():
		print "Squeak! Squeak!"

class MuteQuack(QuackBehavior):
	def quack():
		print ":("
{% endhighlight %}

And we integrate this with the `Duck` class and it's child classes like so:

{% highlight python %}
class Duck(object):
	...

	def __init__(self):
		self.fly_behavior = FlyBehavior()
		self.quack_behavior = QuackBehavior()

	def perform_fly():
		self.fly_behavior.fly()

	def perform_quack():
		self.quack_behavior.quack()

class MallardDuck(Duck):
	def __init__(self):
		self.fly_behavior = FlyWithWings()
		self.quack_behavior = Quack()

	def display():
		print "I look like a Mallard"

class RubberDuck(Duck):
	def __init__(self):
		self.fly_behavior = FlyNoWay()
		self.quack_behavior = Squeak()

	def display():
		print "I look like a Rubber Duck"
{% endhighlight %}

You see the ingenuity of the approach? **Now you can easily add a new Duck type and give it any behavior you want, without making any change to the superclass `Duck` or without adding any new methods to the child class.** This also prevents disasters like flying rubber ducks.

We took implementation away from the child classes and moved it to classes of their own from where they can be reused. _Now not only can you reuse the `Quack` class inside of any `Duck` type class, but you can also separately use it to, maybe, imitate a duck sound._

You can also add/change behaviors at runtime! Just add a setter method to `Duck` like:
{% highlight python %}
class Duck(object):
	...
	def set_flying_behavior(fb):
		self.flying_behavior = fb
{% endhighlight %}

And now we can call this method to set/modify flying behavior of a duck at runtime.

## but the pattern [talks](#the-strategy-pattern) about algorithms?
See it like this: each set of behaviors (`FlyingBehavor` --> `FlyWithWings`, `FlyNoWay`) are like a family of algorithms, and they are _interchangeable_.

## how did we benefit?
We now have a _composition_. Instead of inheriting behaviors, our `Duck` classes are composed of the appropriate behavior classes. This is one of the points we mentioned in the [design principles](#principles).

Composition enables us to encapsulate related behavior together, allowing them to be reused later. And also the behavior can be changed at runtime.

Also, anytime we have to add a new way of flying (`FlyWithRockets`), all we have to do is add a new child for `FlyBehavior` and chill.

*_All examples are taken from the [Head First Design Patterns](http://shop.oreilly.com/product/9780596007126.do)_