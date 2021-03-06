---
layout: post
categories: [Parrot, GSOC, JavaScript]
title: GSoC Idea - JavaScript Compiler
---

To start off the GSoC season, I'm going to post an idea which is near and dear
to my heart: JavaScript on Parrot.

## Problem Statement

> Implement a JavaScript compiler on Parrot, written either mostly or entirely
> in JavaScript. The student may start with an existing project (such as
> Jaspers), or create a new project from scratch. The successful project will
> attempt to use as much existing JavaScript code as possible to implement the
> compiler. This can include parser generator utilities such as PEG.js or
> jison. If the student can find a suitable compiler frontend written in
> JavaScript, he/she could opt to implement a PIR/PBC backend for the existing
> frontend. Jaspers is a project which has not proceeded far past the design
> state which intends to use the PEG.js parser library and an existing
> JavaScript language grammar to produce such a compiler. Other projects such
> as Cafe, provide much of the needed compiler infrastructure but would
> require a suitable PIR/PBC backend. The overall difficulty of this project
> will be determined by the path the student wishes to take, and the amount of
> existing JavaScript the student is able to reuse.

We can distill this down to a simple statement: Parrot wants a JavaScript
compiler. I wrote about [this very idea back in December][js_in_js], that post
includes a lot more details about the project.

[js_in_js]: /2010/12/07/javascript_on_parrot_plan.html

## JavaScript on Parrot

The goal of this project is really a simple one. We want a JavaScript
compiler, preferrably one which is written mostly in JavaScript and
boostrapped. My thesis on the idea is that JavaScript hackers want to write
JavaScript, not C or NQP or any other "standard" compiler implementation
language. There are a lot of coders in the world who know JavaScript but may
not know any of the other languages and technologies which are normally used
to build compilers.

This project is an interesting one because the decisions made by the student
can have a dramatic effect on the overall project difficulty. If the student
takes some pre-existing solutions and meerly adds in a PIR code generator
backend, that is a relatively easy project. For instance, if we started out
with a project like [cafe][] and added in a PIR code backend, that would be
pretty quick. Dukeleto started putting together a project called [Jaspers][]
which aims to be very similar. Jaspers would like to use the existing PEG.js
parser generator library and an existing JavaScript language grammar to
implement a JavaScript compiler for Parrot.

[cafe]: https://github.com/zaach/cafe
[Jaspers]: https://github.com/leto/jaspers

## Difficulty

This project can really span the gammut of difficulty levels, as I mentioned
before. A "working" JavaScript compiler is better than no compiler, but not
as good as a "good" compiler, or a "fast" compiler. My point is, this is the
kind of project which can expand dramatically if the first few stages go more
quickly than normal, but can still produce working results in the span of a
few months if things go very very poorly.

Here are some ideas and points for consideration:

### Start with a working JavaScript compiler

*Difficulty*: 1/5

start with an existing, working compiler like cafe and implement a new PIR
code generator backend. This is the easiest variant of this project to do,
although there is some room to expand if you finish early. Basically, the
project is "finished" when your compiler can be used to compile its own source
code without the need for an external JS compiler to boostrap from.

### Start with an existing JavaScript parser

*Difficulty*: 3/5 - 4/5

Start with an existing JavaScript parser library such as [jison][] or
[PEG.js][peg_js] and implement the rest of the compiler. You could parse
directly to an existing intermediate format like PAST, and let PAST worry
about the dirty details of backend code generation. At least, PAST can do the
work in stage 1 of a two-stage bootstrapped build.

[jison]: https://github.com/zaach/jison
[peg_js]: http://pegjs.majda.cz/

Where this gets difficult is that you need to have working code running on
Parrot in order to generate PAST. So you would likely need a two-stage
bootstrapped built and you would need to rely on an existing JS compiler
(Rhino, node.js, etc) to get the process started. Basically, you use an
existing compiler to generate the "real" compiler, which runs on Parrot and
understands PAST.

Don't pick this variant of the project unless you're good at thinking in
circles!

### Implement a Prototype-Based Object Model

Difficulty: 1/5 (cumulative)

JavaScript's prototype-based object semantics don't mesh very well with
Parrots default object model. For a basic implementation following either of
the above methods, you could treat JavaScript objects as a basic subclass of
Parrot's Hash type and implement workarounds for the `new` operator and
other details. Things could get a little bit messy when you start to implement
invokable objects like method calls, but things wouldn't be too bad.

Instead of just doing a basic workaround to add JS semantics on top of
Parrots object model you could create a new object model from the ground up to
support JavaScript. This could become a bit of a pain in the backside, because
you may end up needing to avoid or fix (or get other developers to fix)
limitations in Parrot which might hamper alternative object models from being
used. I don't suspect there are many such problems, but I would be a fool to
say there are none.

In terms of performance, a custom object model is the only real way to go.
A basic object model wrapper would get the project done faster and with less
effort, but a proper object model would actually bring a big performance
improvement and help Parrot in the long run.

## Deliverables

In addition to the basic compiler system and maybe the object model, there are
other things you are going to need to provide: unit tests, a build
infrastructure, and documentation. In addition there is the overhead of GSoC
participation itself: blog posts and regular meetings with your mentor.

In the end, no matter what you end up producing or how well it works, your
project should be suitable for other hackers to get involved in. The pieces
should be in place for users to get, build, test, install, and use your
compiler. The project should also be accessible so we can start building a
community of hackers to improve and maintain it after GSoC is over.

## How To Get Started

For the aspiring student, if you're interested in doing an implementation of
JavaScript-on-Parrot, you should get in contact with us right away on the
mailing list or on IRC to talk about it and start planning out the details.
Even where the difficulty is listed as 1/5 above, creating a compiler is
not a trivial project, even if it is significantly easier to do it in the
Parrot ecosystem than it may be with traditional compiler-building tools.

## Who Should Apply

This project is for students who have at least basic knowledge about
compilers, know (and maybe even enjoy) JavaScript, and are willing to put in a
bit more effort than the average student.

This project will likely require coding in JavaScript primarily and a little
bit of PIR. If you bootstrap, you may end up writing bits in NQP or Winxed
too. Unless you go all out writing the new object model, you probably won't
write any C.

