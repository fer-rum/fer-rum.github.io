---
layout: post
title: "On Error handling"
category: meta-programming
tags: error handling
---

**Note:** In the following I will use ther term *error* in the broadest sense possible.
There will be no distinction with regard towards *failure*, *fault*, *happy little accidents* or whatever other term is currently en-vougue for different variations of *stuff did not go as planned (or was not planned for at all)*.

# The Outset

Some weeks ago I wrote my article about JSON-based logging.
As a follow-up, I had to show that it can be done quite easily and wrote an example implementation.
All went along smooth until I wanted to add a bit more resilience to the framework.

So, about this error handling thing…

I have come across the topic a few times in the past, like every software author has.
Usually, time or motivation limitations seem to prevent people from exploring the topic further.
After a few days of pondering and even more days of reading and asking around, it dawned on me that

1. Error handling is not trivial (duh!) and
2. No one really seems to have a solution how to *do it well*.

## Define: 'Do it well'

Everyone will have a very different opinion on when error handling is *done well*.
Program does not crash in the most common use cases? That is sufficient for most people.
Others will argue that proper error handling is only achieved if (and only if) the program executes in a well-defined manner for every possible input.
These are two extremes of a broad spectrum.
From my point of view error handling is *done well* when

1. All occurences of errors are detected as early as possible.
2. If the program can fully recover from an error, it will.
3. If the program can only recover with reduced functionality, it will
  * either recover as much as possible (if it is considered a critical piece of infrastructure)
  * or may terminate in an orderly fashion (if it is considered non-critical).
3. If the program can not recover from an error it will terminate in an orderly fashion.
4. If the program terminates or reduces its functionality due to an error,
it must make an appropriate effort to convey to the user what went wrong and why.

In any case the *user* could also be another program or a piece of hardware.
Such programs might keep on running for a long time seemingly unaffected until the error emerges in the form of e.g. corrupt data, inconsistent behaviour or sudden system crashes.

# The Crime Scene

The application has crashed, the server is down, the cores stuck in a deadlock or an endless loop.
Evil Error, the agent of chaos has prevailed again and dealt our sweet dreams of an intact software world a mortal blow.

Who is to blame?

## Suspect 0: The Strategies

Having to handle *happy little accidents* during the execution of code is probably as old as code itself.
Different handling strategies have evolved along with the programming languages.
Four of these strategies seem to be prevalent and can appear either isolated, combined or wrapped and masked as something else.

1. **Using global variables for error indication**
A special globally visible variable is reserved and the currently executing function uses it to indicate a problem.
A good example for this approach would be the *errno* variable(defined in *error.h*) that can be found in *C*. (Since *C11* it is no longer global, but thread-local.)

2. **Return them as part of the function call**
Error codes can be returned either directly as a result of the function call or using [output parameters](https://en.wikipedia.org/wiki/Parameter_(computer_programming)#Output_parameter), if the language permits.

3. **Throw an exception**
At some point in the past, the concept of *exceptions* became fashionable.
People started to cram it in every language they could think of, whether the basic concepts of the language supported such modifications or not.
It took developers a few decades to figure out that this holy grail was probably a cup of mercury as well.
The main problem here is that these Exceptions mess with the control flow of the application by making it non-obvious, which is fundamentally the same thing for which *goto* was shunned until it became nearly extinct (apart from a few niches where it is still relevant).

4. **Handle them as types on their own**
The basic idea in this concept is that errors are objects like everything else in OOP-world.
This way, the errors stop being something special and become a more normal piece of the everyday programming live. Or do they?
Especially in combination with a language with a solid type concept, this can be rather promising.
Just return a composite (or a type-algebraic disjunction) of error- and intended return type and you are all set. Or are you?
While this approach appears sound, I could not yet convince myself that errors *are* types.

## Suspect 1: The Languages

The fundamental issue with all these strategies is that neither enforces error handling.
Programmers can *always* choose to just ignore anything that goes wrong and merrily let the application run its course until it eventually crashes or yields wrong results.
While this seems bad from a perfectionist's perspective it is reasonable;
Nobody wants to write several hundred lines of error handling for a simple `'Hello World'`.
What makes it so ugly is that it can be done implicitly by *just not doing any error handling*.

Languages mostly use their already available non-error-related means to express errors.
This results in strategies covered by 1), 2) and 4).
For good reason, one rarely extends an existing language to entice completely new constructs.
Newly created languages seem to prefer going with strategy 4) instead of offering means to express and handle errors.
This would be a valid approach if (and only if) errors were actually types, but, again, I remain sceptical that this conveys the whole truth.

When designing languages we currently seem to be a bit entrenched on mapping everything to the concepts of functions and objects. (Yes, I completely ignore AOP here. This is a very different *frankensteinian* beast.)

## Suspect 2: The Programmer (and Its Natural Environment)

If you ask developers about the ratio between lines of code and handled workload they will come up with something like

`80% of the programs workload is handled by 20% of the code.`

Inversely, the remaining 80% of the codebase are only there to handle 20% of the workload.
The numbers may vary, but it is always a rather big percentage versus a rather small percentage (which I arbitrarily define as at least 25% apart).
The 80/20 estimation seems to be somewhat common.
This might be related to the [Pareto principle](https://en.wikipedia.org/wiki/Pareto_principle), but I found no sources actually linking the intuition, the principle and at least statistical or empiric evidence.

It also seems to be consensus to attribute a very significant chunk of the *much code, little workload* section to error handling (Although it is even harder to come up with reliable numbers for that estimation).
The reality seems to be that often enough, people tend to implement the high-workload parts of code that are related to the actual functionality but omit the rest either completely or in significant parts.

Several reasons come to mind:
* In companies, especially smaller ones, economic pressure can force the developers to get as much functionality done as fast as possible.
No time remains for doing proper error handling and checking the code for potential pitfalls.
* Academic projects suffer from a similar symptom. The code needs to be working as fast as possible to get the associated paper ready before the deadline. No one expects this code to be used anywhere else.
* And last but not least there are the spare-time open source projects which are mostly driven by the pure desire to get something done. Such motivation is a limited resource as well and often not sufficient to cover the boring task of seaching for all the places where the code might break.

# My Conclusions

It is not as dark as I made it look.
People write a lot of beautiful code, be it well managed companies, engaged academics or enthusiastic hobbyists.
But it is getting pretty cloudy and the raindrops start falling.
We continuously stack more and more software on top of each other.
Sometimes a piece of the foundation gives way and then we can observe how even the greatest software monuments crumble.

There are a few (fuzzy) points I want to make here:

* Languages should focus more on the handling of anormal application states. This would probably include introducing appropriate syntax and semantics. Dealing with *things that can go wrong* is a growing core issue in an ever more complex software world.
* Ignoring errors should only be done explicitly. The compiler should always warn about ignored errors and offers an option to treat ignored errors as compile errors (in case you need to write critical code).
* Business and Reseach practicises need to adapt more to the need of developers to have the time (and ressources) available to review, rewrite and rethink their code. Sadly we can not easily increase the motivation of unfunded open-source projects, except by offering languages in which error handling is easy and fun to do.
* We need to continue seaching for error handling strategies, for we might not have found our programmer-paradise yet, but it might still be out there somewhere.

# Further Reading

Here are a few of the sources which I consulted and found to add something to the overall topic.
A lot of the finer details covered in these were omitted to keep the text at a reasonable length.

* *Weimer* and *Necula* wrote a [quite extensive survey](http://web.eecs.umich.edu/~weimerw/p/weimer-toplas2008.pdf) on the topic of exceptions and their impact on programming, examining the pitfalls of this concept and offering approaches to counteract them.

* *Joe Duffy* wrote a [rather long blog post](http://joeduffyblog.com/2016/02/07/the-error-model/) in which he explored the topic in a similar fashion.
Since his approach focuses on *Microsoft*'s *Midori* project, he naturally proposes a different solution.
His initial detailed analysis of the topic is a good read, whether you agree with him or not.

* [*The Rust Documentation*](https://doc.rust-lang.org/1.12.0/book/error-handling.html) offers a condensed approach of how error handling and type system could come together.

* *Gavin King* [also discussed](https://ceylon-lang.org/blog/2015/12/14/failure/) how typing and error handling can go hand in hand - at least most of the time.

* *Laurence Tratt* discussed the impact of error handling on code size [in a blog post](http://tratt.net/laurie/blog/entries/a_proposal_for_error_handling.html) with regards to (mostly) *C* and went with the language-extension approach to tackle the issue.

# Article History
* 06.11.2017 - Published initial Version
* 07.11.2017 - After first feedback a Typos got fixed and unclear phrasing was improved
