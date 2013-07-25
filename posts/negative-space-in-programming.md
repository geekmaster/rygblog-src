-title=Negative space in programming
-time=2010-12-12 02:27:27
There's a lot of material out there about code; how to design it, write it, document it, build it and test it. It is, after all, the primary artifact we programmers produce \(okay, that and long rants about why we're right and almost everyone else is wrong\). Considerably less attention is paid to what's not there \- the design alternatives that were rejected, the features that were omitted, the interfaces that got changed because they were too complicated/subtle to explain properly in the docs, the lack of excessive dependencies that helps keep build times down, the tests that aren't necessary because certain types of errors are made impossible by the design of a system.

I think that's a mistake. In my experience, the main way we actually give our programs shape is not by putting things in, but by leaving them out. An elegant program is not one that checks off all the bullet points from some arbitrary feature list; it's one that solves the problem it's meant to solve and does so concisely. Its quality is not that it does what it's supposed to; it's that it does almost nothing else. And make no mistake, picking the right problem to solve in the first place is hard, and more of an art than a science. If you've ever worked on a big program, you've probably experienced this first\-hand: You want to "just quickly add that one feature", only to discover hours \(or days, or even weeks\) later that it turns out to be the straw that breaks the camel's back. At that point, there's usually little to do except take one last glance at the wreckage, sigh, and revert your changes.

So here's my first point: Next time you get into that situation, write down what you tried and why it didn't work. No need to turn it into an essay; a few short paragraphs are usually plenty. If it's something directly related to design choices in the code, put it into comments in that piece of code; if the issues stem from the architecture of your system, put it into your Docs/Wiki or at least write a mail. But make sure it's documented somewhere; when working on any piece of code, knowing what doesn't work is at least as important as knowing what does. The latter is usually well\-documented \(or at least known\), but the former often isn't \- no one even knows the brick wall is there until the first person runs into it and gets a bloody nose.

Second point: If you're thinking about rewriting a piece of code, be very aware that the problem is not one of deleting X lines of code and writing Y lines to replace it. Nor is it one of understanding the original data structures and implementation strategies and improving of them; even if the approach is fine and you're just re\-implementing the same idea with better code, it's never that simple. What I've consistently found to be the biggest problem in replacing code is all the unspoken assumptions surrounding it: the API calls it doesn't use because they don't quite do the right thing, the unspecified behavior or side\-effects that other pieces of code rely on, the problems it doesn't need to deal with because they're avoided by design.

When reading code, looking at what a program does \(and how it does it\) is instructive. But figuring out what it doesn't do \(and why\) can be positively enlightening!