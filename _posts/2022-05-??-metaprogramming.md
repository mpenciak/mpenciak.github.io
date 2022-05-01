---
layout: post
title: "The Monads of Lean 4"
subtitle: "A meta description of meta programming in Lean 4"
---

The current state of documentation on metaprogramming in Lean 4 is rapidly expanding. 


paper: <https://arxiv.org/abs/2001.10490>
Talk: <https://www.youtube.com/watch?v=hxQ1vvhYN_U&t=3380s>

The first monad `CoreM`

## Goal

The goal of this project is to provide a reference for classically trained mathematicians to write tactics in Lean 4. 

The audience is focused on mathematicians who have moderate experience with functional programming languages (Lean 4, or Haskell are good choices). Though maybe at some point I can include a blog post about 

## TODO :

* Monads: What they are, and examples 
* Monad transformers (`ReaderT`, `StateT`, etc.)
* Monads making up Lean 4. This includes `CoreM`, `MetaM`, ...
* Talk about the paper (need for hygiene, and the solution)