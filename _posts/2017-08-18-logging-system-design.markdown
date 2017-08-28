---
layout: post
title:  "Design for a JSON-based logging system"
date:   2017-08-18 18:00:00 +0200
category: programming
tags: Qt JSON logging software design
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
The reason why the `Log Manager` does not actually create `Log Renderer`s itself in this sketch is because the latter may become rather heterogenous as will be discussed below.
In _Java_ one would build a [_factory_](https://en.wikipedia.org/wiki/Factory_method_pattern) for this reason, while in C++ it can be done with template magic.

## Logger

This is where most of the action happens.

One can have one or multiple `Logger`s for different use cases.
(E.g. one per subsystem, or per severity level, or per user, or … you get the idea.)
Each of these `Logger`s can be connected to `Log Renderer`s via an [observer pattern](https://en.wikipedia.org/wiki/Observer_pattern).
In this setup the `Log Renderer`s act as the _observers_ and the notification of these is integrated into the `log(…)`-method.
Also when calling `log(…)`, a new `Log Message` is created and stored within the `Logger`.

When creating a new message the `Logger` may insert additional information on its own like a timestamp or the subsystem name.

## Log Message

In this class the flexibility comes into play.

The `Log Message` contains a `QJson Object` which allows us to store arbitrary information and additional data in such a message.
It is the concrete implementations decision which fields there are and of which type they are.
The important part is that these meanings have to be clear throughout the logging system.
For practicality reasons, additional `QJson Value`s can be attached to messages to modify them further.
In an ideal case however, the messages are never exposed outside the logging system and all the interactions regarding them happen through the `Logger` and `Log Renderer`s.

### Possible Structure of a Log Message

To aid the understanding of the flexibility a bit more, a basic example of a `Log Message` in JSON representation shall be given.

``` JSON
{
    "timestamp": 1503322242,
    "source": "Model Subsystem",
    "text": "Attempted division by 0",
    "details": {
        "function": "divide",
        "arguments": {
            "dividend": 12.65,
            "divisor": 0
        },
    },
    "severity": 2
}
```

While this is only one possibility of many, it highlights how features that are usually to be expected in log messages can be represented.
* The _timestamp_ is given as [_Unix time_](https://en.wikipedia.org/wiki/Unix_time) which makes representation very easy and helps a lot with sorting and filtering.
* The _source_ in this case may be the name of the `Logger` that produced the message, which also is named after the subsystem it represents.
* The _text_ holds the short error message itself.
* The _details_ object here contains the name of the called function and its arguments. Feel free to expand this into a full stack trace or represent it as an array of strings or whatever you need - go wild!
* A _severity_ allows to figure out how grave the event is and can, for example, influence rendering properties like color and font size.

If we now decide to upgrade our logging system to have a new feature, it can be inserted without any trouble by adding more fields in the JSON.

## Log Renderer

Time to display what we logged.

The rendering interacts with the `Logger` as discussed above.
For that purpose an abstract (or in C++ terms: virtual) class is used.
A concrete `Log Renderer` implementation obtains the `Log Message` via notification and decides on its own what to do with it.

This may include anything from simple formatting and printing to buffering and writing to long term storage or sending them somewhere via the internet.
The possibilities are endless.

It should be noted that renderers should ignore JSON fields in the message they do not know about, and should provide meaningful default values for fields they required but were not included in the message.
This way a very robust system can be achieved and the same message can be handled by the most different kind of rendereres.

# The Life-Cycle

In this section a examplary logging system life cycle will be outlined to illustrate the interactions between the parts a bit more.

1. The application starts and a `Log Manager` gets instantiated.
2. Each subsystem creates a `Logger`. The graphics subsystem also creates a custom `Log Renderer` specialized for displaying log messages in a sorted fashion.
3. The `Log Renderer` gets registered at all `Loggers`.
4. During their life time, the subsystems use the `log(…)` method of their respective `Logger` to generate `Log Messages` that get forwarded the `Log Renderer` who in turn displays them.
5. On subsytem shutdown, the subsystems `Logger` unregisters the `Log Renderer` and gets destroyed.
6. On shutdown of the graphics system the `Log Renderer` gets unregistered from all remaining `Loggers`. From this point on, no more messages can be displayed.
7. The application shuts down and the `Log Manager` gets destroyed.

# Finer Details

A few specific topics that come to mind when talking about logging shall not be omitted here.

## Log Levels

Usually a log level (aka _severity_) refers to a representation of how urgent, important or impactful a message is.
Commonly people use a fixed set of values like _Debug_, _Verbose_, _Info_, _Default_, _Error_, _Critical_ or something alike.
When representing these as pure strings it becomes more readable on the outset, but harder to sort.
By using a numerical representation sorting and filtering is far easier but readability suffers; And what if one suddenly needs to include a new log level in between?

The best compromise seems to define enumerations and find a good policy of dividing up the range of available numbers (since most people, for the sake of simplicity, use a (32 or 64 bit) integer anyway).

A posibility would be to define the _default_ in the middle of the number range (in case of signed integers: 0) and insert all other levels by further bisecting the available number ranges.
Something along the lines of

```
enum LogLevel {
    Lowest      = INT_MIN,
    Verbose     = INT_MIN / 2,      // = Default + Lowest / 2
    Default     = 0,                // = INT_MAX + INT_MIN / 2
    Error       = INT_MAX / 2,      // = Highest + Default / 2
    Critical    = (INT_MAX / 4) * 3 // = Error + Highest / 2
    Highest     = INT_MAX
}
/*
 * Of course you might write down proper pre-calculated values instead
 * of these convoluted calculations.
 * They have the advantage that the compiler can evaluate these expressions at
 * compile time.
 * But unlike pre-calculated numbers, these are not portable between
 * architectures with different bit sizes.
 */
```

An alternative approach would be to hold these log levels in a map of numeric keys and string values.
This offers a high flexibility when it comes to dynamically adding new log levels on the fly.
It is also advantageous to reserve one log level to express an _undefined_ level that may result from incomplete data or parsing errors.

## Handling Binary Data

Let's assume you want to attach a [_BLOB_](https://en.wikipedia.org/wiki/Binary_large_object) to your log message (like a screenshot or a memory dump).
How do you embed this into JSON?

One approach is to just encode it as [_Base64_](https://en.wikipedia.org/wiki/Base64) and insert it as a string.
Maybe it can be better represented as a JSON object on its own as well.
Finally there is a variant called [_BSON_](http://bsonspec.org/) (short for binary JSON) that can handle such things.
But in this case you will have to use a specialized library to read / write your log message objects, since default JSON libraries do not support it.

## Additional Miscellaneous Considerations

* For handling common logging scenarios, it might be a good idea to extend the `Logger` to offer shortcuts for frequent tasks.
* The `Logger` may retain the `Log Message`s to keep a log history and may push all previous messages to a newly registered `Log Renderer`.
* Sorting and filtering messages should always be the renderers task.

# Implementation Notes

The original implementation of this system is part of a commercial application so it may not be published here.
What can be shared however are some implementation specific notes regarding the realization of such a system in C++ and _Qt_.

* It is highly recommendable to use the monotonic system clock to provide timestamps. This guarantees that each timestamp is unique and can double-serve as a message ID. The impact on precision is negligible since the timestamp resolution is usually far greater than the frequency with which your application can create log messages. For pretty rendering, timestamps are usually converted into local time anyway and one rarely is interested in more then microsecond resolution.
* Using shared pointers is mandatory. Trying to keep track of all the objects flying around via raw pointers will be hard and make everything far more complicated, especially when handling `Log Message`s. Handing around pure objects is no option either because if your message grow larger, you wil incur a noticable perfomace penalty. _Qt_ offers its own pretty (and standard-compatible) shared pointers and there is no excuse to not use them.
* The log levels handle wonderfully when combined with `QEnum` since it allows handling the enumerations name as `QString` without additional magic required on the application programmers side. This makes rendering them a pure bliss.

---

# Article History

* 21.08.2017 - Finished writing the initial version
* 22.08.2017 - Corrected phrasing by removing redundant use of the  same word and fixing misplaced underscore
* 28.08.2017 - After implementing a quick mockup a few observations and design changes had been made to further reduce the implementation difficulty. The changed text reflects this.
