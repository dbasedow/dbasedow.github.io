---
layout: post
title: Implementing logic other than AND in Prospecter
date:   2014-09-15 15:32:00
---

When I first started work on [Prospecter](https://dbasedow.github.io/prospecter/) I focused only on combining query
conditions with AND. Doing this meant that matching would be very easy and efficient. I was able to construct a bit mask
during query registration that has one bit per condition.

    A & B & C & D -> 1111
    A & B & ~C & D -> 1101

The literal symbols represent conditions in a query. Throughout this post I will use the operators &, | and ~ for AND,
OR and NOT.

A posting in the index then simply consisted of a query id and the bit to set when this posting was encountered.
Supporting not logic was working as well without changing the matching algorithms at all.

Matching was very simple, the process looked like this:

    bits = 0000
    if A is found: bits[0] = 1
    if B is found: bits[1] = 1
    if C is found: bits[2] = 1
    if D is found: bits[3] = 1

    if bits == mask:
        match

If we look at the two examples from above we see that if C is found and the third bit is set, the second bit mask will
not match.

But what about OR?
------------------
Of course I had to add support for OR queries at some point. I also had a rough idea of how to do it: convert queries
to conjunctive normal form, a conjunction of disjunction or ANDing of ORs.

    (A & B) | C is equivalent to (A | C) & (B | C)

So it is still possible to use the same bit mask. For every side of an AND there are two bits in the mask. So for the
example the mask is simply: 11. The example will result in four postings, two for condition C (set both the first and
the second bit), one for A (set the first bit) and one for B (set the second bit).

There is a good open source library for converting arbitrary logical expressions to CNF:
[aima-java](https://code.google.com/p/aima-java/) the example code from
[Artificial Intelligence - A Modern Approach 3rd Edition](http://aima.cs.berkeley.edu/). It contains a lot of other
stuff and the propositional logic package relied heavily on the concrete PropositionSymbol class which does some very
restrictive symbol name checking in the constructor. So I forked a [stripped down, slightly refactored version of
aima-java](https://github.com/dbasedow/aima-propositional-logic).

But there is a new problem: negations don't work anymore. What would be the mask for (~A | B)? Apparently there is no
way around tracking negations separately in some way. The problem is, a second bit mask is not enough. A bit mask could
only work if no more than one negation was allowed per disjunction. Not really an option.

De Morgan to the rescue
-----------------------
Thankfully Augustus De Morgan proved that:

    ~A | ~B <=> ~(A & B)

So it is possible to use another "sub" bit mask to keep track of negations in a clause and determine if **all** negated
conditions for that clause have been encountered in the document. Since the position in the clause doesn't matter it is
not even necessary to use bit masks, a simple counter is enough.

For every clause that contains negated conditions Prospecter creates a special counter that keeps track of how many
negated conditions are inside the clause.

The postings for negated conditions need to be distinguished from non-negated conditions, there are two obvious
approaches: use a separate index structure in every field, use the existing structures but add a flag to the posting
indicating if the posting is for a negated condition. I went with the first approach, just because it is much easier
to debug. I will probably move to the second approach in the near future to minimize overhead.

During the testing phase negations are ignored first, if query already matches
the regular mask there is no need to consider additional negations. But if the mask does not match, the negation counter
is fetched and compared to the counts determined during query registration. For every bit with negations the actual
matched negation count is compared with the amount of negations in the clause.

    A & B & (~C | ~D | ~E) -> bit mask: 111, negation count: {3=>3} (at bit 3 there are 3 negations)

If C, D or E are fulfilled a counter is incremented. So if all three are fulfilled the counter would be at 3 otherwise
it will be less. The negation count for bit 3 is 3 so if the actual count is anything less than 3 it means there are
(actualCount - originalCount) not fulfilled negated conditions and therefore the bit can be set to 1 in the normal bit
set.

An example
----------

    A & B & (~C | ~D | ~E) -> bit mask: 111, negation count: {3=>3}
    initilize bitset to 000, counter to {1: 0, 2: 0, 3: 0}
    A is true -> bitset: 100
    B is true -> bitset: 110
    C is true -> counter: {1: 0, 2: 0, 3: 1}
    E is true -> counter: {1: 0, 2: 0, 3: 2}
    test bit set against bit mask: 111 != 110
    check negated condition counts position 3: 2 < 3 -> bitset: 111
    test bit set against bit mask: 111 == 111

And now Prospecter can deal with arbitrary logic. [Try it out](https://dbasedow.github.io/prospecter/getting-started/)

These changes described here took waaaaaay longer than expected.