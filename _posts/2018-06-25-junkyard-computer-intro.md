---
layout: post
title:  "Building Computing Machines from Ground Up: Intro"
date:   2018-06-25 01:00:00 +0400
categories: computer
---
_revision 0.8.3_

_Note: revisions below 1.0.0 are work in progress and thus are subjects to
change.  I nevertheless decided to publish them as soon as they reached more
or less stable form to facilitate feedback and to speed up overall processing
of notes._

In this series of posts I will discuss building digital computers from scratch:
possible approaches, trade-offs and shortcomings of various technologies
available to reasonably dedicated person. Also I will try to navigate perilous
bogs of software development for such machines.

Foreword
--------

I believe almost everyone has been told at least once that all you need to
build a digital computer is a bunch of basic logic elements. I have always
found this statement extremely dishonest. While technically true it clearly
leaves something important out,  just like saying "All you need to build Taj
Mahal is sufficiently big pile of rocks" -- correct in a broad sense, but
obviously some know-how should be present as well, also it would be nice to get
some tools too.  In my experience this know-how of ways to transform a tray of
logic elements into a computer is surprisingly hard to come by and largely
should not be sought from a source which is fond of generic statements. It
remained a mystery for me for quite a time. If only I had a book or at least a
pamphlet describing basics without going too deep into details and
optimizations! Alas, at those times I was unable to find such guide.

One day I realised, that since at some point in time building computers
from discrete elements was the only way to go, with modern technology and
knowledge it should be easy to reproduce those accomplishments of old. And
chances are that someone has already done that.  And so I googled for it and
guess what? It turned out that people were indeed building computers from
scratch!

The first thing I found was [MyCPU project](http://mycpu.eu/): impressive, but
at the moment of finding described in too few details. While posing more
questions than actually answering it did two important things. First of all it
showed me that it is generally feasible to design and build your own computer
from discrete components. Second, it provided a nice entry point to DIY CPU
web-ring which in turn had many projects to consider and draw inspiration and
understanding from. Then my "a-ha" moment happened when I found
[CPUville](http://www.cpuville.com/Projects/Original-CPU/Original-CPU-home.html)
-- now I understood enough to try my hand at designing my own computer. It
turned out, that the process had much more detours and compromises than I
imagined, but finally I got myself several simulated circuits which could
perform general purpose computations.  In this series of posts I am planning to
share what I have found out while working on various aspects of these crude
computers. Since I have still to come up with the most logical and
non-contradictory way to organize my enormous pile of computer notes I will
simply publish them as soon as any part is ready.

Now a few words of caution. This series is intended to be a mostly entertaining
and a bit educational reading piece. It describes a possible path to the goal,
but one should not consider this path to be the best one, it is merely the path
taken. In most cases there are better ways, tools and techniques to achieve
this or that and I will try my best to point out potential pitfalls and
inefficiencies, but if I fail to do that  please don't blame me. Also I am not
intending to provide a comprehensive, ready to copy design, but rather to show
how a design could be produced. And finally, if you decide to build something
please don't forget safety practices when working with tools or power sources
and if you do and something goes wrong don't blame me and say that I didn't
warn you.

Before I begin with actual design let me examine some preliminary
questions. And first of all,


A Word About Compromises
------------------------

Building computers at home when one is past initial hurdle of understanding how
a computer should generally be put together precesses around compromises. If
you are not racing to get a computer ready to some date and to build it to some
specific capacity (well, at least I hope that no one builds computers from
scratch out of pressing need) you have a seemingly endless number of
possibilities to consider. Which operations to implement in hardware? Stacks or
registers? Which basic components to use? Should it be binary or ternary would
be even cooler? Consideration and testing of available options took quite a
time for me, some avenues had to be eventually abandoned due to the lack of
resources to follow them, some decisions were made by brutally excluding most
of the options and authoritatively saying that this particular one will be the
way to follow.  Thus I had to temporarily abandon the idea of building ternary
machine out of wooden cogwheels and marbles and eventually gravitated towards
medium scale integration circuits. I will briefly describe the path which led
me to this rather dull decision below -- it was less straightforward than one
would expect, but before that a word on what should be considered a general
purpose computer.

What to consider a computer
---------------------------

Or in other words, how to determine when the goal is reached. Surprisingly
there is a room to answer this question since both machines that answer
positively to "But can it run Crysis?" and "Can it land a spaceship on the
Moon?" should be considered computers. Peculiarly enough, the latter one is
probably a billion times simpler than the former one.  Thus the first problem
I had to deal with was to define a computer for myself. Since building a
computer capable of running Crysis even at grains of FPS appears to be somewhat
hard task and navigation computer feels a bit like a very complicated special
purpose device (its software is rather narrow-ranged and does not get modified
much, especially not when performing maneuvering in space) I have
voluntaristically decided to consider a device a computer if it could be
programmed to compute sines of eight bit fixed point numbers in reasonable
amount of time. Sines have tons of uses everywhere so it does not feel like a
special purpose to me and I could always go to cosine from that, or even dare
simple simulations like Lorenz weather model. Reasonable amount of time was set
to 60 seconds which is about three times faster than I can do the same task
with pen and paper. After some more thought I have added a soft definition
which expands the original one by removing the 60 seconds constrain out of fear
to rule out possible electromechanical machines.  Such definition feels like
the closest thing to the spirit of computers: compute some number and probably
do something about it. Also I'd like to specially stress that the _compute_
part is essential otherwise a memory device with trigonometric table would
qualify as a computer too.

Parts choice
------------

It is important to note, that this question is very low-level one, since as
long as at least basic logic elements are present there is no need to care how
exactly they are implemented and it is sufficient to think on proper level of
abstraction, yet I believe half of the fun of making a computer from the ground
up lies in choosing the most bizarre basis ever. (In the Taj Mahal analogy it
would be akin to considering rolling your own rocks with occasional
detours to amend cosmological constants).

The options at hand are almost endless, starting from mundane logic gates,
progressing to DIY logic gates and then to DIY building blocks for DIY logic
gates. Then at some point one realizes that the gates should not necessarily be
electric ones: pneumatic (or steam) or mechanical gates would be fun to work
with! In fact even if you constrain yourself to electrically driven devices you
will still have a lot to consider, so much that I had to split the very basic
considerations in a separate post. Here I'll simply provide a summary: I
decided to go with medium-scale integrated circuits with limited number of DIY
parts when such parts allowed to reduce complexity.  I did it after noticing
that I was thinking in terms of MSIs anyway: I realised that trying a hand with
them would be still rather educational. Now I had just one small obstacle to
overcome before I could start modelling and that was

Instruction set
---------------

Having overcome the first obstacle of basic parts choice, I have immediately
ran into another one: instruction set. Since my goal was a very rudimentary,
but universal computer, I was expecting to have no less than eight instructions
and no more than sixteen. I thought that having too few would overcomplicate
programming and having too many would overburden the design and would
potentially lead to more annoying hardware bugs. So obviously I had to choose
the most important instructions from some real or imaginary set. Having too
much choice once again turned against me.  It turned out, that I didn't
actually realize which secondary instructions would be useful and which could
be safely left out. It was obvious that at least a conditional jump is a must,
but how badly I will be affected by leaving out other types of jumps?  What
about extra arithmetic operations?  Extra logic? It was a task of finding
balance between usability and orthogonality of instruction set and amount of
work required to build CPU: the more instructions are there, the more wires I
will have to solder in the end of the day. Sure thing I could pick up a well
known architecture and replicate it, something old and relatively simple, like
6502 or 8080, but that would have required tremendous amount of repetitious
effort and would have likely shadowed the important parts. So it was crucial to
trim down instruction set for the most useful subset which would provide just
enough features. I have spent quite some time poring down on instruction sets
of early 8-bit CPUs trying to boil them down to something small. This question
led me to development of several crude emulators, assemblers and
macroprocessors to check various ideas.

Time was passing, my set of crude instruments kept growing, but I was still far
from answering my original question -- what instruction set to use for the CPU
of my future computer? At least I was not choosing between stack machine and
register machine. Finally at some point I decided, that it was time to stop
considering options and start acting, dropped all instruction sets I have
considered so far and decided to continue with BrainF--k. Why BF? It is
compact, Turing-complete and quite well-known, and I thought it should not be
much worse in the end than any other concoction. In retrospective I should have
probably gone with subleq: despite being even more minimalistic, it allows for
easier higher-level language implementation -- writing a compiler to BrainF--k
turned out to be more tedious task than I liked, which required jumping through
too many loops without having proper jumps. Yet BF issues didn't bother me much
since I decided to stick with it  just long enough till I can emulate richer
CPU like 8080 on one BF computer or on a cluster of such machines.

With this decision all preliminary questions were finally answered and gone and
all that was left was to design a computer and build some tools! Here the
preparation ends and begins the real journey which is the subject of another
story.
