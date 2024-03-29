# Actionable interpreter #

## What is this about ##

This little package provides an extension to the default interpreter class
from [xstate][xstate] project.

Added functionality:  
Allow functional [actions][actions] (i.e. all non built-in actions) to raise
events with the interpreter via returned value, without having to have a reference
to an interpreter instance in advance.

## Why is this needed ##

Xstate is organized to keep Machine definition (states, context, actions,
etc.) separate from interpreter that runs that machine. This allows for great
insulation of logic but also is inconvenient is certain use cases.

In particular, when event is generated by machine in some fashion dynamically
and therefore must be issued by a function that is a part of the machine
definition.

In the original Xstate spec, state [transition][transition] is happening by
following mechanics:

1. Event is external and is fed to the machine by interpreter (invoking send()
   or sendTo() of the default interpreter).
2. Event raised by one of the built-in actions - send(), raise().
3. One event can produce different transitions by implementing conditional
   transitions (guards).

The trouble comes when you need to have different transitions based on
information that is internal the machine (state + context). Namely, if you want
to raise the event from your action's code.

Lets consider a machine:

               ---> B
    I --> A --|
               ---> C

After transitioning to A from an external event you want the machine to end up
in either state B or C, based on some internal information (context) and
incoming event.

One solution would be to use guards intensively with **Null Events** and
**onEntry** actions, but that involves a lot of duplication and is just messy.
onEntry actions process event's side-effects, since guards should be pure,
duplication comes from the fact that every guard must evaluate conditions
independently, so if logic is not some straightforward value check, you either
repeat it for every guard or you save intermediary calculations into context in
some onEntry action - messy.

Even then, there's a particular (common) scenario where guards won't work - if
the choice of the transition is delayed. For example, after reaching state A,
the machine makes an async request (to database, for example) and then decides
on which transition to take based on the results of the request.

Another solution is to make a choice inside A's onEntry action and then invoke
send() with the appropriate event on the interpreter. The obvious downside is that
with this route the action function must have a way to reference the instance of
the interpreter. Which in practice means it have to rely on some global
variable.

This package provides a solution for both of those scenarios.

## Installation ##

Regular and straightforward.

``` bash
npm install --save @trulyacerbic/xstate.actionable-interpreter
```

Then in the code replace import of the default interpreter with the one from
this package:  

Replace...

``` js
const { Interpreter } = require('xstate/lib/interpreter');
```

... with

``` js
const { Interpreter } = require('@trulyacerbic/xstate.actionable-interpreter');
```

## Usage ##

In your action function, return the event you want interpreter to raise as a
return value:

* return an object, a string or a number - the definition is exactly the same as
  the standard [event][events]. This event will be put to the internal queue of
  the interpreter and raised after all current actions executed.
* return a Promise. When and if this promise successfully resolves, the
  resolving value will be used as a new event to be send to the interpreter.

[xstate]: https://xstate.js.org/docs/
[actions]: https://xstate.js.org/docs/guides/actions.html
[events]: https://xstate.js.org/docs/guides/events.html#sending-events
[transition]: https://xstate.js.org/docs/guides/transitions.html#machine-transition-method
