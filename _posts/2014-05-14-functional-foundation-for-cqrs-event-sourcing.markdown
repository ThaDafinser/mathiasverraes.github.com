---
title: "A Functional Foundation for CQRS/ES"
slug: functional-foundation-for-cqrs-event-sourcing
date: 2014-05-14
layout: post
published: true
tags: [blog]
abstract: "A CQRS architecture can be seen as a set of referentially transparent functions that model decisions and interpretation."
image: "/img/posts/2014-05-14-functional-foundation-for-cqrs/circle-diagram.png"
---


I've been dabbling with Functional Programming languages for a while now. Although I can't say I'm at all proficient, it has affected my programming style. Even if my code is mostly OOP, my whiteboard drawings look more functional. I'm [thinking about functions a lot](/2014/01/domain-driven-design-is-linguistic/). So it made sense to describe the CQRS-architecture of my system as a set of functions.


Greg Young describes Command Query Responsibility Segregation as "using two objects where previously there was only one". The idea is as ingenious as it is simple, but I feel the quote doesn't do it justice. Rinat Abdullin proposed in a [Twitter conversation](https://twitter.com/abdullin/status/465747026953908225) with Eric Evans, the idea that CQRS enables you to denormalize your model. The value lies in the fact that, because you have independent read and write models, you can reason more clearly about your model. The separation opens up opportunities to capture deeper insights. You can now optimise the two sides separately.

Even though you can use CQRS without Event Sourcing, this post assumes Domain Events all the way down. In a brainstorming session with [@stijnvnh](https://twitter.com/stijnvnh), we identified four (conceptual) functions.



## Protection


```
f(history, command) -> events
(where history is the series of past events for a single Aggregate instance)
```

Commands and Domain Events are first class citizens of our model. Aggregates no longer expose state, but only events. Aggregates are at the core of the decision making process. They decide what the outcome of a Command is, and express that outcome with Domain Events. The history of that particular Aggregate instance, influences the decision, so we must inject it. If you want to split the reconstitution of the Aggregate instance from the handling of the command, you can use currying. This helps to keep infrastructural and model concerns separated:

```f(history)(command) -> events```

'Protection' refers to the primary goal of the Aggregates: protecting its business rules, making sure that the user can not cause the system to go in an inconsistent or illegal state. When a Command violates the business rules, the operation should fail:

```f(history, command) -> either<exception|events>```

(The error handling part of FP is still a bit terra incognita for me though.)

The exact same idea, can be expressed in BDD-style scenario's: Given this history, when this command is received, then these events should happen.

## Interpretation

```f(events) -> state```

As the Aggregates do not expose, something else must. The function above is a Projector. It takes (a stream of) events, and projects these into something that represents a subset of the perceived state of our system. That state makes up our read models for further consumption. I like the idea of "interpretation" here: Two different Projectors could have a completely different interpretation of what the meaning of a set of behaviours is. This is similar to how a journalist or politician interprets a series of events, and comes up with a different conclusion, or at least a different focus, than her colleagues.

Of course these interpretations are based in the domain. The beauty of the decoupling from the write model, is that when the domain evolves, or we gain deeper insight, it's now easy to shift the interpretation. Make a new projector and a new read model, and replay the history of the system into them.

You can express the projection as a left fold:

```f(events) -> foldLeft(apply, events, init)```


## Intention

```f(state) -> command```

One of the major consumers of the read models, is usually an end user. Many applications need to somehow report state (projected in the previous section). The user looks at the state, and makes decisions. The user then makes his intention known to domain model, by sending commands. And it is an intention indeed, because, as we saw earlier, the Aggregate might refuse the command later.

Note that we don't present the user with the list of events, (unless that temporality is important). Humans are pretty bad at deducing state from events -- so never base your opinion on a single journalist or politician! In the case of our application, we give the user ready-made, automated interpretations.

It's the only function presented here that is not pure. Human users are influenced by lots of global state, they have ideas, principles, mood swings, and [biases](/2014/01/domain-driven-design-is-linguistic/). It's like they have a mind of their own! Modelling the end user is pretty much impossible...

## Automation

```f(events) -> command```

... but sometimes it is! Humans in organisations behave predictably in certain circumstances. Many tasks in an organisation are repeatable: "Whenever X and Y have happened, Z needs to happen." Automating these business processes, making them faster and safer, is one of the appeals of enterprise software. Process Managers take on this job: they are event listeners, that react to certain events or combinations of events. They may hold intermediate state, so internally, they look like projectors. But the state is invisible. The only observable output are new commands.

In some cases, a Process Manager can output events straight away. This makes sense when there are no invariants other than what the Process Manager can take care of on its own. But most of the time, you'll want a proper Command that can be refused by an Aggregate.

<img style="float:left;margin-right: 10px" src="/img/posts/2014-05-14-functional-foundation-for-cqrs/circle-diagram-small.png" alt="Functional Foundation for CQRS">


## Conclusion

What is the point of all of this? Thinking about [how systems work](http://verraes.net/2013/08/john-gall-systemantics-the-systems-bible/), is one my hobbies. Having a good theoretical foundation, has often helped me when designing systems. I may not end up doing things according to the theory, but the theory forces me to think hard about what I'm doing.

Pure functions are deterministic for the same inputs and outputs. Independent of shared mutable state, a pure function can isolated in a separate process, on a separate server, doin' his thang. See where this is going?

## Update June 8, 2014

I wrote that user intention can not be represented as a pure function, but of course you simply need to pass in the state of the human, like this:

```f(brain, state) -> command```

## Update Dec 17, 2016

My model completely lacked Queries. Having state without being able to query it is rather useless. Like Events and Commands, a [Query is just a message](/2015/01/messaging-flavours/). It fits in the Query Model (duh) aka Read Model like this: `f(history, query) -> state` where state is a subset of the all the state that the projector writes. In CQRS, I think ideally, a projector should have **private state**, meaning that the projector is the only one that read and write its own (logical) private database. All clientss get state by sending one of the projector's supported Query messages. Another type of query is a Subscription, as sometimes a client is interested in a stream of (filtered) events as opposed to state.

So this represents my current thinking here:

{% highlight haskell %}
handle :: [Event] -> Command -> [Event]
subscribe :: [Event] -> SubscriptionQuery -> [Event]
process :: [Event] -> [Command|Event]
query :: [Event] -> Query -> State
{% endhighlight %}

or with intermediate states made explicit:

{% highlight haskell %}
project :: State -> Event -> State
query' :: State -> Query -> State
{% endhighlight %}

And modelling the human:

{% highlight haskell %}
learn :: State -> Brain -> Query
decide :: State -> Brain -> [Command]
{% endhighlight %}


## Read More

- [Domain Events](/2014/11/domain-events/)
- If you like the circular representation of CQRS, I have a lot more of them in my slides for [Fighting Bottlenecks with CQRS](/2013/12/fighting-bottlenecks-with-cqrs/)
- [Domain-Driven Design is Linguistic](/2014/01/domain-driven-design-is-linguistic/)
- [Messaging Flavours](/2015/01/messaging-flavours/)


