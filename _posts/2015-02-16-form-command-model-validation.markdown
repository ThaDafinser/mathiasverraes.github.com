---
title: "Form, Command, and Model Validation"
slug: form-command-model-validation
date: 2015-02-16
layout: post
published: true
tags: [blog]
abstract: "Make a clear distinction between different layers of validation."
image: "http://verraes.net/img/posts/2015-02-16-form-command-model-validation/smokesignals.jpg"
---

Many of the frameworks I've worked with, promise to separate responsibilities with MVC. In practice, they end up coupling everything to everything. The forms are coupled to the models, and there's a grand unified validation layer. This may be convenient at first, but it breaks down for larger systems, and creates headaches when having to support multiple clients. My approach is to clearly separate the validation for the form itself, from the Command validation and the model validation.



## Commands

<img style="float:right;margin-left: 10px" src="/img/posts/2015-02-16-form-command-model-validation/smokesignals-small.jpg" alt="Smoke Signals">

[A Command is a message that represents the intention of the user](/2015/01/messaging-flavours/). Command validation happens after the form is submitted and before the Command is passed to the model. The question to answer here is "Does this look like a valid Foo Command?" It guarantees that the "birthday" field contains an actual date, or that "zipcode" looks like a zipcode. It doesn't guarantee set constraints, like username uniqueness. In other words, you validate the message, not whether you should execute the message.
 
The Command object should use Value Objects. They guarantee their own consistency, so the Command delegates validation to them. Note that we're not trying to inform the user of validation errors. We simply throw exceptions. The assumption here is that either the form prevents malformed values, or the user is trying to bypass form validation. We don't return friendly error messages to attackers. 


## Model

The model is the "official" source of truth. It doesn't worry whether the "birthday" field looks like a date and whether the "zipcode" looks like a zipcode, because it can trust the Command. The model is responsible for protecting its own invariants, business rules, and constraints. It can reliably[^1] prevent duplicate usernames, account overdrafts, and all kinds of other business concerns. It fails hard when executing the Command would violate those concerns.

The Commands, the Value Objects, and the objects that make up the model, are decoupled from the framework and the persistence. We can effectively transport them to a different context, such as a different framework. They are independently testable with unit tests.  

## Forms

Form validation is a UX concern: you add it to help the user fill in the values quickly and correctly. Nowadays, users expect forms to give them instant visual feedback, and rightly so. Client side is the only option. Of course, this client side validation can call back to the server to validate things that the client cannot know, such as "is this username available?". 

By the time the submit button is pressed, some of the data may already be stale, for example another user might have registered the same name. The form validation stops most of the cases, but it can never be relied on. Not only because it lags behind on the server, but also because it is easy to spoof requests and bypass the validation. 

I'm not aiming to make the form validation as complete as possible, at least not initially. Rather, I try to look at it from the point of view of the user. The things that they are most likely to get wrong, are the things I try to guide them into doing correctly. Ideally, I watch or monitor how a real life user fills in the form, keep track of the most frequent model errors they trigger, and focus on preventing those first.

## Conclusion

**The fundamental idea here is that "helping the user", "validating the message", and "protecting the model's consistency", are three completely independent concerns.**
 
Yes, you will end up with something that looks like duplication. For example, both the form and the model will have a concept of "account overdraft" and what it means. These are however different concepts, because the goal is different: the model protects business rules, the form helps the end user. I feel that this separation of responsibilities [trumps the minor cost of duplication](/2014/08/dry-is-about-knowledge/). 
   

[^1]: In some cases, I might choose eventual consistency: Accept the small risk that usernames might clash, detect the conflicts after the events are persisted, and deal with them somehow. 


## Read more

- [Messaging Flavours](/2015/01/messaging-flavours/)
- [DRY is about Knowledge](/2014/08/dry-is-about-knowledge/)
- [Decoupling (Symfony2) Forms from Entities](/2013/04/decoupling-symfony2-forms-from-entities/)