---
title:	"AI and News Flashes: The Observer Pattern"
date:	2016-04-27
excerpt: What do you do when you are asked to create a notification system where the objects to be notified get added/removed at runtime? You stop giving a shit about them, and code for the interface.
---
#### The Observer Pattern: 
Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

#### Principles:
1. Strive for loosely coupled designs between objects that interact.

## when will I need it?
So there is a smart "NewsGetter Machine". It fetches news from different sources on its own (these machines will one day start coding as well :cry:). This news is being consumed by two (for now) "Online Newspapers". You are given the task to implement a way for these newspapers to get the latest news as it comes and display it on their sites. The super intelligent AI guys who built the NewsGetter also implemented a `news_flash()` method. This method gets called every time the NewsGetter gets some new news.

_Oh, also, there is a 100% chance of more newspapers starting to use the **NewsGetter Machine** as their primary source of news (because AI is the future)._

You, being you, coded the implementation in under 5 minutes:
{% highlight python %}
class NewsGetterMachine(object):
	def __init__(self):
		self.news = None

	def get_news():
		return self.news

	def news_flash():
		news = self.get_news()

		news_paper_a.update(news)
		news_paper_b.update(news)
{% endhighlight %}

Genius! Every time there is a news flash, `news_flash()` gets called, which gets the latest news and you update the newspapers. Simple and sweet. And extensible. No. No? No.

What about the stuff we learnt about in our [Strategy Pattern post]({% post_url 2016-04-26-design-patterns-strategy %})? Looks like **we coded concrete implementations inside the `news_flash()` method. Now every time we get another newspaper that wants to use the `NewsGetter`, we will have to modify our code. That is bad. We should encapsulate what we know will vary.**


## Observer Pattern to the rescue!
Time to use some [design principles](#principles).

In the Observer Pattern we have two entities, **Subject** and **Observer**. As is not clear by the names, **Subject is the _thing_ that holds state** (information, like news in our case) and **Observer is the _thing_ that wants to get notified when the state of the Subject changes**, because the Observer needs to do some crazy things based on the change. It is the Subject's responsibility to notify Observer's whenever it's state changes.

Implementing this for our case, we will define two classes: `Subject` and `Observer`. 

- The `Subject` class will implement methods to `register`, `remove` and `notify` Observers.
- The `Observer` class implements the `update` method that is called whenever the Subject notifies the observer.

While we are at it, we will also define a `NewsPaper` class that implements a `display_news` method that is called whenever the observer's `update` method is called. This is done in order to standardize the APIs that newspapers will create. Any newspaper that needs to integrate with the `NewsGetterMachine` will need to inherit from `Observer` _and_ `Newspaper` classes and implement the unimplemented methods.

Here is our definition of the superclasses:
{% highlight python %}
class Subject(object):
	def __init__(self):
		self.observer_list = []

	def register_observer(obs):
		self.observer_list.append(obs)

	def remove_observer(obs):
		if obs in self.observer_list:
			self.observer_list.remove(obs)

	def notify_observers(updated_news):
		for observer in observer_list:
			observer.update(updated_news)

class Observer(object):
	def update(updated_news):
		raise NotImplementedError

class Newspaper(object):
	def display_news():
		raise NotImplementedError
{% endhighlight %}

And we integrate them with our `NewsGetterMachine` and new `NewspaperA` and `NewspaperB` classes like so:
{% highlight python %}
class NewsGetterMachine(Subject):
	...

	def news_flash():
		news = self.get_news()
		self.notify_observers(news)

	# Just a temporary function to set news for our tests
	def set_news(news):
		self.news = news
		self.news_flash()

class NewspaperA(Newspaper, Observer):
	# Register Newspaper with the NewsGetterMachine
	def __init__(self, news_getter_machine_subject):
		self.subject = news_getter_machine_subject
		news_getter_machine_subject.register_observer(self)

	def update(updated_news):
		# Do stuff specific to NewspaperA
		# like maybe change the news (welcome to the real world)
		...

		self.display_news(updated_news)

	def display_news(updated_news):
		print updated_news

class NewspaperB(Newspaper, Observer):
	# Register Newspaper with the NewsGetterMachine
	def __init__(self, news_getter_machine_subject):
		self.subject = news_getter_machine_subject
		news_getter_machine_subject.register_observer(self)

	def update(updated_news):
		# Do stuff specific to NewspaperB
		...

		self.display_news(updated_news)

	def display_news(updated_news):
		print updated_news
{% endhighlight %}

This was the Observer pattern. Notice how we were **Pushing** the news when we notified observers? This is sometimes not desirable. In that case we can go with a **Pulling** implementation. The call to `notify_observers` is made without any extra information. The observers, when they receive the notification, can call `get_news()` method of the `NewsGetterMachine` to fetch latest news if they want to.

## how did we benefit?
Now we can add any number of newspapers and just register them with the `NewsGetterMachine` and they will get the updated news! We can also register or remove observers at runtime. 

Also the implementation of the newspapers, and how they display the news, is all abstracted away from the `NewsGetterMachine`. All this machine knows is that there are some observers that it has to notify by calling the `update` method on them.

**For an excellent, pro-level, implementation of the Observer pattern, take a look at [Django's Signals](https://docs.djangoproject.com/en/1.9/topics/signals/)**