# Lecture 29 - Multiple Contexts

## Recap
Explored OpenGL contexts

Discussed concurrency and parallelism

Wrote robust tests, highly sensitive to data races and race conditions

## Agenda

OpenGL contexts, again

A sneak-peek at what multiple active contexts will give us

glad’s multi-context support

Major refactoring without excessive pain
Utilizing existing structures in our code
Judicious use of regular expressions


##Design Principles / Techniques In Action

Pointers to member functions

= delete to refine APIs

Delegating constructors: DRY

Single Responsibility Principle

Always think about copy/move semantics when designing a new type

Beware explicit when refactoring


