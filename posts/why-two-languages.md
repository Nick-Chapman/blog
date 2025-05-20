
# Why Two Languages?

During the last couple of months I have been working on a new project, called
[Barefun](https://github.com/Nick-Chapman/barefun). The idea is to write a small operating system for bare metal x86, using a functional programming language.

Operating systems are traditionally written in a language such as C where all memory is managed explicitly. I want to see if instead we can use a garbage collected language from the ground up. The Garbage Collector will be integral to the OS. My goals are to learn about traditional osdev, whilst exploring some more unusual approaches to compilation and execution. There is no requirement to be compatible with any existing protocol or calling convention. There is no C here. And there is no existing operating system. The point of this project is to _write_ the operating system. And to do so in a nice high level functional language with a standard Hindley-Milner type system. This project is for fun, so if I want to reimplement every piece of the stack, then I can.

Towards the start of this project, I stumbled into an unusual decision:

- The _operating system_ is implemented in _barefun_, an ocaml-style language.
- The _compiler_ for _barefun_ is written in haskell.

Although _barefun_ is not ocaml, it strives to be a pure subset. No syntactic innovation. No semantic variation. One happy upshot is we can pretend barefun programs are ocaml, and take advantage of the mature ocaml development environment. There is no need to write a type checker for barefun.

Language design, including type system design, is a can of worms I wish to avoid in this project. The scope is already quite large. Advice on the osdev website actively [councils against](https://wiki.osdev.org/Alta_Lang)
trying to design a new programming language and write your own compiler while attempting to write an OS.
So at least restricting myself to _just_ writing a compiler for a subset of an existing language is probably a wise choice.

_But why have I decided to write the compiler for barefun in haskell and not in barefun/ocaml?_

This article is a justification for why this this is not quite as silly a decision as it might seem.

- I love Ocaml and Haskell.
- Haskell is a great language for writing a compiler.
- Ocaml could be a great language for writing low level code.
- I have more experience with compilation techniques for strict functional languages than for lazy.
- Implementing a compiler in a language different from that being compiled reduces complexity.
- You don't always need to be self hosting.
- You don't always have to dogfood.

### I love Ocaml and Haskell.

I have commercial experience in both languages. I have used both for my own projects in the past, although recently I am more likely to use Haskell. This project provides satisfaction for both passions. I also enjoy low level assembly coding, but doing anything substantial is so error prone. If only we had a high level language with a decent type system and automatic memory management which would compile to the assembly code we want....


### Haskell is a great choice for writing a compiler.

A compiler is best structured as a pipeline of pure transformations. I don't particularly care about laziness,
[but laziness kept haskell pure](https://stackoverflow.com/questions/31477074/how-lazy-evaluation-forced-haskell-to-be-pure)
and I do care about purity. Purity makes refactoring just work. I want to develop my compiler in a pure language. I don't want state. Or perhaps I should say: I am happy for all state to be mediated by monadic types, such as IO or my own compilation monads. Haskell also has lots of syntactic conveniences which make coding a joy. It's not the quickest to compile; but it's ok.

### Ocaml could be a great language for writing low level code.

[Other people think so.](https://mirage.io/)
I want to embrace the idea of using garbage collection everywhere, while still allowing mutable data-structures for deep programmer control. A balance of convenience and performance. Use of a strict functional language allows the programmer a simple mental model of execution. We get runtime memory safely. And we get a programming language with all the standard functional loveliness: specifically a decent type system including polymorphism, variant types, and inference.

_Why don't I also desire purity for the language in which I write my operating system?_ To be honest, I am not exactly sure. It just feels it would be an added burden. Maybe this is wrong; we will see. It would undoubtedly be harder for _me_ to write a compiler for a pure/lazy language. So perhaps I am post-rationalizing.

### I have more experience with compilation techniques for strict functional languages than for lazy.

My approach to compilation is focused on the CPS/ANF style promoted by
[Appel](https://www.goodreads.com/book/show/2079575.Compiling_with_Continuations)
and [others](https://news.ycombinator.com/item?id=40192579).
This approach targets strict functional languages.

One idea I really like is to use the heap for call frames as well as for closures and data objects. We are going to have a GC heap anyway, so lets use it for everything! Having stack allocated call-frames is commonly promoted for efficiency reasons. (Although [not everyone agrees](https://www.cs.princeton.edu/~appel/papers/45.pdf)). But by avoiding the stack entirely, even if it comes with some performance hit, we open the door to interesting approaches for the execution model. And not having a stack makes some things MUCH simpler:
Currency, and getting [_safe-for-space-complexity_](https://flint.cs.yale.edu/flint/publications/escc.html)
correct. I plan to write more about this in a future article.

I am sure it would also be pretty cool to use a pure/non-strict/lazy functional language to write an operating system from the ground-up. But sometimes you've just gotta do what you know!

### Implementing a compiler in a language different from that being compiled reduces complexity

Bare with me on this one. Object language choices are independent from implementation language choices. I can use Haskell for implementation, and make use of any feature or syntactic convenience I want without worrying that my compiler will have to support those features. I can keep my object language (ocaml subset) as small as possible. Syntactically and Semantically. Only increasing the subset in response to real needs driven by trying to code the operating system in that language.

### You don't always need to be self hosting

Maybe one day it would be fun to explore self hosting, but currently my focus is for off-line development of the OS. My current target architecture is x86 real mode (16 bit) which is tiny. But I still get a development environment with modern luxuries, such as: emacs, haskell, ocaml and dune.

### You don't always have to dogfood

In principle, it's a good idea. You could argue I am dogfooding the barefun language -- I am writing the compiler for barefun in tandem with writing an application in barefun. But perhaps the best language for writing the barefun compiler itself is not the same as the best language for writing the operating system.

## Conclusion

There are multiple goals for the Barefun project:

- To learn more about operating system concepts, including filesystems and concurrency.
- To learn more about the x86 architecture.
- To showcase some unusual ideas for compilation and execution.
- To figure out how best to engineer a compiler without being overwhelmed by complexity.
- To have fun.

I hope that embracing different languages for different levels will further these goals.
In the future I plan to write more about various aspects of this Barefun project. Watch this space.
