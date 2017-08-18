---
layout: post
title:  "Design for a JSON-based logging system"
date:   2017-08-18 18:00:00 +0200
categories: Qt JSON logging software design
---

# Preface

We all have been there: 

Things in our programs go wrong and it is debug time.
Or people tell us that things go wrong, but can't tell what exactly they did to trigger the error.
Or one of the dozen other cases occur where it would be simply nice to have a peek into the inner workings of our program without firing up the _gdb_ (or any other debugger of your choice).

So we naturally just throw a few lines of code in there that print some additional information about what is going on.
Then it becomes a lot of lines and stuff starts to get messy.
Multi-threading makes sure that everyting gets mixed up a bit and finding the one important line in a thousand lines of unformatted output becomes the search for the needle in a haystack.

"Proper logging for the rescue", we yell and turn to the internet for a ready-to-use solution.
But nothing really seem to fit. Some systems are just pimped 'print' instructions and others have the perceived complexity of a mars mission.

Looks like we write our own logging system, like everyone before us. 

# The Aims and the Not-Aims

This article will deal with a very basic design of a logging system for use in custom applications.
While it will focus on the implementation using C++11 (or newer) and _Qt_ it's principles should be easily transferrable to other languages and frameworks.

The main focus lies on
* __Simplicity__
  It should be very easy to understand how this system works and to use it.

* __Flexibility__
  Adapting the system to custom use cases should be as straight forward as possible.
  Extending an already existing implementation must be a low-effort procedure.

* __Ease of implementation__
  The implementation of such a logging system must not become a huge chore.

To achieve this a few sacrifices have to be made:

* __Speed__
  The resulting system will only be sufficiently fast. 
  Don't even try to build something like this into a real-time system.

* __Memory efficiency__
  Log messages may become comparatively large and if you are tight on memory, this might not be the way to go.

In summary the presented design will be more suitable for small to medium desktop applications than for mobile development, drivers or _MegaCorp's Superscaling World Domination Project_.

Please note that the proposed design is neither completely fleshed out nor perfect in any way.
It's purpose is to inspire, not to present a ready to go solution.

# The Basic Structural Design

To get a basic logging system, one needs a few components.
Here is a (somewhat close to) UML sketch:

![A sketch of the logging system](/images/LogSystem_UML.svg)

The green components are somewhat fixed, while the lilac ones have to be completely custom-made.
Remember that the _Q…_-data types are part of the _Qt_ framework.
In case you don't use this framework you will naturally have to use appropriate replacements. 

Let's have a closer look at the core components.

## Log Manager

Here is the root of our logging system.

Since usually one application wide instance is sufficient, a [_Singleton_](https://en.wikipedia.org/wiki/Singleton_pattern) might be appropriate. 
This is hinted by the inclusion of the `getInstance`-method.
  (I am aware that a lot of people frown upon this pattern and they have good arguments for that. It's optional.)

When requested the `Log Manager` will create new `Logger`s, and add `Log Renderer`s to keep track of them and provide acces to them if requested.
  (The reason why the `Log Manager` does not actually create `Log Renderer`s itself is because the latter may become rather heterogenous as will be discussed below. Unless you want to go full _Java_ and build a [_factory_](https://en.wikipedia.org/wiki/Factory_method_pattern)…)

## Logger

This is where most of the action happens.

One can have one or multiple `Logger`s for different use cases. 
(E.g. one per subsystem, or per severity level, or per user, or … you get the idea.)
Each of these `Logger`s can be connected to `Log Renderer`s via an [observer pattern](https://en.wikipedia.org/wiki/Observer_pattern).
In this setup the `Log Renderer`s act as the _observers_ and the notification of these is integrated into the `log(…)`-method.
Also when calling `log(…)`, a new `Log Message` is created and stored within the `Logger`.

## Log Message

## Log Renderer

# The Life-Cycle

# Finer Details

* Logging levels
* handling binary data
* shortcuts for certain logging scenarios
* Log history, filtering and sorting

# Implementation Notes

* use monotonic clock to provide timestamps
* use shared pointers


