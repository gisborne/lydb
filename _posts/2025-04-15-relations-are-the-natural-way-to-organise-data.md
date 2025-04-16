---
layout: post
---

# Why Relations are the Natural Way to Organise Data

# TL;DR

The Relational Database is the most natural and general way to store and retrieve data. We can — more or less — prove this mathematically. This post explains how.

# Preamble

Folks familiar with these matters will hopefully agree that I’m not waving my hands about too much. But yes, this is inevitably a tad simplified.

## Aristotle and Logic

In the 4th Century BCE, the Greek philosopher [Aristotle](https://en.wikipedia.org/wiki/Aristotle) described a system of logic based on the [*syllogism*](https://en.wikipedia.org/wiki/Syllogism). ”Socrates is a man; all men are mortal; therefore Socrates is mortal“ is the sort of thing.

Remarkably, from that time to the 17th Century, — over two millenia — syllogisms were regarded as essentially [the complete and last word about logic](https://en.wikipedia.org/wiki/Syllogism#Jean_Buridan).

But then, starting in the 19th Century, mathematics began to change profoundly. Before that time, mathematics was always taken as being *about* some real thing. Newton developed the Infinitesimal Calculus so he could use it to invent modern Physics, for example.

The 19th Century saw mathematicians turn to studying mathematical ideas for their own sake, with the likes of [non-euclidean geometries](https://en.wikipedia.org/wiki/Non-Euclidean_geometry) and [transfinite numbers](https://en.wikipedia.org/wiki/Transfinite_numbers). Some philosophers and mathematicians worried that these new strange and sometimes counterintuitive ideas that weren’t really *about* anything might hide deep contradictions. What was needed was some way to formalise what was and wasn’t correct mathematics, in some entirely unambiguous way. Mathematics should be built up from an unambiguously true and simple base, and then we would absolutely know that it was correct. This came to be known as [Hilbert’s Program](https://plato.stanford.edu/entries/hilbert-program/).

In a similar vein, folks noticed that performing calculations and even checking a proof seemed to be rote processes — you just checked whether each step followed from the last. The century that was building trains and other machines like mad naturally lead to thinking about machines that could perform calculations “by steam”. You’ll have heard of [Babbage’s Analytical Engine](https://en.wikipedia.org/wiki/Analytical_engine#Influence), no doubt.

At the same time, as mathematicians were talking increasingly about [“the crisis in mathematics”](https://simple.wikipedia.org/wiki/Foundational_crisis_of_mathematics), mathematicians started to turn their tools on Mathematics itself. Progress was made ([Boolean Algebra](https://simple.wikipedia.org/wiki/Boolean_algebra) for example) in small ways.

But then came [*Begriffsschrift*](https://en.wikipedia.org/wiki/Begriffsschrift).

## The natural language of Mathematics

German for “Thought Script”, Begriffsschrift (1876) was the 19th century‘s most significant intellectual achievement. In it, Gottlob Frege described a new and much more expressive kind of logic, very different to Aristotle‘s. Frege‘s logic was rich enough to express any sort of mathematical reasoning. It could form the basis both of the reinvention of mathematics and the automation of computation.

In the nearly 150 years since, Frege‘s logic has been extensively studied and elaborated on, but there has never been any serious suggestion to fundamentally replace it. Frege‘s logic is, in short, the natural language of mathematics.

# Weren‘t we talking about databases?

There is an intimate relationship between programming and Logic. Developments in programming are often driven by developments in Logic — Rust‘s *borrow checker* is basically an implementation of *Linear Logic*, for example. LISP, Haskell, Prolog and OCaml were all developed based on ideas from logic and mathematics.

The Relational Model — the theoretical foundation for the Relational Database — is also based on logic.

## First Order Logic and the Relational Model

The basis of the relational database is what we now call First Order Logic (FOL), a subset of Frege’s full logic.

In First Order Logic:
- We have *predicates* that express relations between objects
- We have *quantifiers* that allow us to express statements about collections of objects
- We have a formal way to derive new facts from existing ones

These properties map directly to relational databases:
- Tables represent *relations* (predicates)
- Rows represent individual facts
- Queries (especially in SQL) correspond to logical formulas
- Query execution corresponds to theorem proving

- The SQL statement `SELECT * FROM Employees WHERE department = ‘Engineering’` is essentially a logical formula that says “give me all objects x such that the predicate Employee(x) is true AND the property department(x) = ‘Engineering’ is true.” What the database does to answer this query is to write a program to prove the statement for every row of results it presents as the result.

## The Halting Problem and Logical Incompleteness

Now we come to the crucial point: First Order Logic is the most powerful logic that is suitable for a general purpose database.

The reason lies in two of the most profound results in mathematical logic:
1. Gödel‘s Incompleteness Theorems
2. The Halting Problem

In 1931, Kurt Gödel proved that any formal system more expressive than First Order Logic contains statements that are true but unprovable using that system.

A few years later, Alan Turing proved the Halting Problem — that in general, we can’t tell whether a computer program will ever finish. This established fundamental limits on what computation can achieve.

These two results are deeply connected. They both demonstrate that there are fundamental limits to formal systems and computation. Any language more expressive than First Order Logic would necessarily run into these limitations.

FOL is the most flexible and expressive *possible* way of representing reasoning and querying where we can guarantee that we can find any consequence of any fact in finite time.

## Why This Matters for Data

When we store and retrieve data, we need a system that:
1. Is expressive enough to represent complex relationships
2. Guarantees that queries terminate in finite time
3. Has a clear semantics that allows reasoning about correctness

First Order Logic hits this sweet spot perfectly:
- It‘s expressive enough to represent any facts we care about
- Queries in FOL are *decidable* (they always terminate)
- FOL is *complete* (any fact that is true can be proved)
- It has a well-understood formal semantics

In the nearly 200 years since Begriffsschrift, logicians have studied logic in great detail. None that are more expressive than First Order Logic offer this “always computable” guarantee. Gödel’s theorems make this a proven mathematical fact.

Before the relational database, programmers would often write their own programs to store and retreive data in general-purpose languages. This was incredibly inefficient and unreliable.

A relational database represents facts using logic, and treats query answering as proof. This system is the most general and expressive *possible* way to store and retrieve data where you can guarantee that you will get out all the answers in a finite amount of time.

## NoSQL and Beyond: The Limits of Alternative Paradigms

Since the 2000s, we‘ve seen the rise of various “NoSQL” database systems: document stores, key-value stores, graph databases, and others. These systems often claim to overcome limitations of the relational model, but they all face a fundamental tradeoff:

1. If they remain within the expressive power of FOL, they‘re essentially just different interfaces to less general representations than logic;
2. If they try to exceed the expressive power of FOL, they encounter the fundamental limitations established by Gödel and Turing.

Document databases like MongoDB, for example, optimize for certain access patterns but sacrifice the general flexibility of relational systems. The documents in your MongoDB database are great for what you’re using them for *now*, but as your needs change, you will inevitably wind up breaking them up into simpler, more table-like units until you’ve just reinvented the relational database with a worse storage model and query language.

The limitations of such systems aren’t technological — they‘re mathematical. No matter how much hardware or engineering we throw at the problem, we cannot escape the fundamental limitations established by the Halting Problem and Incompleteness Theorems.

# Conclusion

The connection between First Order Logic and the relational model isn‘t just an interesting academic observation — it’s a profound insight into the nature of data itself. First Order Logic represents the boundary of what is expressible while remaining finitely computable.

When we understand this connection, we see that the relational model isn’t just one paradigm among many — it’s the mathematically optimal way to represent general-purpose data storage and retrieval. Other approaches may optimize for specific use cases, but they either sacrifice generality or bump up against the fundamental limitations established by Gödel and Turing.

The relational model has endured for over 50 years not because of technological inertia but because it embodies a profound mathematical truth: relations are indeed the natural way to organize data.

## Coda: But SQL is terrible

It deserves to be said that SQL is still terrible. It is the only practical relational database query language we have, but it is deeply flawed. It’s horrible just as a programming language: verbose; hard to write; hard to read; counterintuitive; and filled with inconcistencies and foot guns. There are better languages (particularly Datalog), but only SQL is widely available and generally supported.

That we aren’t using something better is the single greatest tragedy in the computer industry. It is the single biggest contributor to software engineering being so inefficient. It is why folks are drawn to non-relational monstrosities like MongoDB. It’s why we have to write a whole separate layer of software (“middleware”) between our data store and our user interfaces, where we do as much of our computing as we can get away with in something that isn’t SQL.

I guess I’m just trying to be clear that I’m not saying here that *SQL* is the last word in database designs.