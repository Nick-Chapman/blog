
# Beyond x86

At the outset of my [Quarter Forth](https://github.com/Nick-Chapman/quarter-forth)
project, before it was even called _Quarter Forth_,
there was a single _x86_ implementation.
Learning to code x86 was one of the motivating reasons for the project.

In those early days, as I began to explore how
[bootstrapping](2.bootstrap.md) works in Forth,
I dreamed of porting Quarter Forth to other platforms.
In particular I wanted an implementation for the
[MOS Technology 6502](https://en.wikipedia.org/wiki/MOS_Technology_6502)
architecture, which would run on my
[BBC Micro computer](https://en.wikipedia.org/wiki/BBC_Micro) -- _The Beeb_ -- the machine from my childhood. I have that implementation now!

In a [recent project](https://github.com/Nick-Chapman/beeb)
I explored graphics programming on the Beeb.
In particular, I wanted to recreate something like the classic
[Meteors](https://bbcmicro.co.uk/game.php?id=8) game,
which was a rather nice port of the arcade classic _Asteroids_.
I made some good progress, but became bogged down which the sheer effort of coding in Asm.
My hope in porting Quarter Forth to the Beeb (as well as it being a fun thing to do anyway!), is to pick up my meters-clone project again.

I currently have four implementations of Quarter Forth:

- [x86](https://github.com/Nick-Chapman/quarter-forth) The original implementation.
- [Haskell](https://github.com/Nick-Chapman/tc-quarter) emulation. With type checker.
- [Golang](https://github.com/Nick-Chapman/go-quarter) emulation.
- [6502/BBC Micro](https://github.com/Nick-Chapman/beeb-quarter) implementation.

In a [previous article](3.quarter)
about the Quarter language, I wrote: _we don't choose to be cryptic just for the sake of it. Our purpose is to allow a full Forth system to be bootstrapped in an Asm-independent way._
I would claim that so far this has panned our quite well. The amount of kernel code required for each new port has been about as minimal as could be hoped.

All the existing implementations are 16 bit,
although this is not a fixed requirement for Quarter Forth.
The _x86_ and _6502/BBC_ are subroutine threaded.
The Haskell and Golang versions are emulations rather than native code implementations.
In the following I briefly discuss all the implementations, then focus on some special challenges on the BBC port.


## x86

This is the original implementations of Quarter Forth

- 16 bit; 64k address space; subroutine threaded
- Return stack
  - hardware stack; indexed using register `sp`
  - located at top of memory `&ffff`, growing downwards
  - operations: `call`, `ret`, `push`, `pop`

- Parameter stack
  - indexed using register `bp`
  - located at at `&fbff`, growing downwards (allowing 1k for return stack)

Memory Map. The
[kernel](https://github.com/Nick-Chapman/quarter-forth/tree/main/x86/kernel.asm)
is loaded at `&0500` (just above the BIOS).
It's size is 2905 bytes.
So the initial location of the heap pointer `here` is `&1059`.
After the
[full system](https://github.com/Nick-Chapman/quarter-forth/tree/main/full.list)
is loaded, including some examples and a snake application,
`here` reaches `&3ab0`.


## Haskell

This purpose of the Haskell
implementation was to provide a base to explore offline type-checking of Forth.
This was a side track project, which I hope to [write](4.typing.md) about in the future.
It is an emulation rather than a native code implementation.
And memory is modelled as a `Map` from `Addr` to `Slot`, where `Slot` is defined:

```
data Slot
  = SlotRet
  | SlotCall Addr
  | SlotLit Value
  | SlotChar Char
  | SlotEntry Entry
```

This allows tracking the intent of each memory location, which is necessary for the type checking approach taken.


## Go

The Golang implementation of Quarter Forth was a vehicle to learn `Golang`!

Like the Haskell version, this is an emulation rather than a native implementation.
Memory is modelled using `map[addr]slot` rather than a flat array -- this will have a performance cost -- the justification was my exploration of the _interfaces_ language feature of golang.
I may well switch to a flat array, in particular if I use the golang version to generate offline memory images, which will be necessary to improve the startup times when running the 6502/BBC Micro version. More details below.


## 6502 / BBC

This implementation of Quarter Forth targets the 6502 running on the BBC Micro computer.

- 16 bit; 64k address space; subroutine threaded
- Return stack
    - hardware stack; indexed using (hidden) register `S`
    - located in page-1 of memory (no choice) , growing downwards
    - operations: `jsr`, `rts`, `pha`, `pla`

- Parameter stack
    - indexed using (user) register `X`
    - located in 0-page memory, from `&90`, growing downwards

Memory Map:

- variables and parameter stack in zero-page
- return stack in page-1
- kernel loaded at `&1200`
- initial `here` at `&1b0b`
- full system `here` at `&3cfe`

### Status of the Beeb Port

The full Quarter Forth system is now up and running on the Beeb. Including buffered line entry, memory debugging: `dump` and `xxd`, and disassembly: `see`, `see-all`.
Running both in the [b-em](https://github.com/stardot/b-em) emulator,
and on real hardware.
The implementation is a work-on-progress, so the precise details given above regarding size and memory layout are in flux, but already we can see there is a problem. In fact there are two problems.

- Quarter Forth takes up too much space.
- Quarter Forth is too slow to startup.


#### Too slow

Obviously it's going to be slow to run on a 40 year old machine.
But the problem is more specific than that...

A native Quarter Forth system is composed of a host specific kernel (_x86_ or _6502_) and a codebase, made up of Quarter code and Forth code.
The kernel contains about 60 primitives, directly coded in the host Asm.
The remainder of the system (the majority) is compiled at startup,
before the operator gets to type even a single character at the system prompt.

Each system will enable access to the Forth source code in a host specific way. For both the x86 and 6502 implementations, the source code is embedded in the kernel binary image.
When the system starts, the source code is read using the `key` primitive, character by character until it runs out, whereupon `key` switches to reading from the operator's keyboard. In a very real sense it is as if the source code was directly typed into the system, automatically, on startup.

This is all well and good for the x86 implementation.
When using the Qemu emulation it takes just a second to boot the full system.
When running on bare metal it is even quicker: Just the blink of an eye.

But the BBC Micro is not so quick.
Running its 6502 microprocessor at 2 MhZ,
it takes about two and a half minutes to boot a Quarter Forth system.
For development, we can run the _b-em_ emulator at about 10x speed, so reducing startup to just 15 seconds or so. But it is not ideal.

But there is an easy solution to this problem. We can compile the system ahead of time, and embed the image of the compiled code directly in the kernel binary, rather than embedding the source code. This should completely minimize the startup time on the BBC. The only time remaining should being what it takes to load the image from disk. About 2 or 3 seconds.


#### Too big

Unlike the x86 implementation we do not have the full 64k address space at our disposal.
Firstly, the BBC Micro has only 32k RAM (from 0 to `&8000`). The top 32k of the address space is ROM containing the machine operating system (`MOS`) and the builtin BASIC interpreter.
Secondly, the BBC kernel is loaded a lot higher up in memory (`&1200`) than it was for the x86 version (`&500`). On the BBC, below `&1200` is working memory used by the MOS and BASIC. It is possible that some of this can be reclaimed, but for now `&1200` seems as low as we can go.

So do we have the ful range from `&1200` to `&8000` (27.5k) at our disposal?
No. This must be shared with screen memory.
Exactly how much memory is dedicated to the screen depends on which screen mode we are running in.

Fortunately the BBC has one very frugal screen mode: the so-called
[Mode-7](https://beebwiki.mdfs.net/MODE_7).
This is a text mode, running at 40 columns by 25 rows;
each character taking just one byte.
This is the mode which the BBC boots-up in.
When running in Mode-7, the screen memory starts at `&7c00`
so our 27.5k just got cut to 26.5k.

However, to do graphics programming we need to switch
to one of BBC's graphic [modes](https://beebwiki.mdfs.net/MODE).
Probably Mode-1 would be suitable for Meteors.
But with a resolution of 320x256 pixels, and four colours,
this takes up a full 20k of RAM for the screen memory.
Oh boy. We don't have much space left.
Just 7.5k for our user program. And that includes the forth system!

To be precise, Mode-1 screen memory starts at `&3000`
But from above we see that `here` for the full system already reaches `&3cfe`
And indeed if we switch to Mode-1 in the BBC implementation of Quarter Forth,
the system crashes as screen overlaps with out executable code

We need to find a way to be more frugal with memory. Some ideas:

- Use memory below `&1200`.
- Reduce the size of the BBC Asm kernel.
- Reduce the size of the forth system, discarding features we don't need.
- Reclaim memory from the forth system which is only required at compile time.

The final idea above follows the realization that much of the memory used by a forth system is solely to implement the dictionary structure.
The dictionary is required when we type words at the operator prompt, and when we compile new 
definitions using the colon compiler. But when an application begins to run, we wont need either of these features.

Dictionary headers take a lot of space. For example:

```
: square  dup * ;
```

Using _subroutine-threading_, the executable code takes just 7 bytes. This is the same on both 6502 and x86.
These 7 bytes are take by two three-byte `jsr` (_jump-subroutine_) instructions (`call` on x86), and one single byte `rts` (_return-subroutine_) instruction (`ret` on x86).
We can reduce this 7 bytes to 5 bytes by using a_direct-threading_ or _indirect-threading_ execution model,
instead of subroutine-threading. But this requires we pay an extra 2 bytes per definition (so no gain here!) and there are time trade offs.

But anyway, we are focusing here on the dictionary header costs. What are these for the `square` example? In general this is implementation dependent, but in my current implementation (both BBC and x86) we take 7 bytes for the `square` name string (which includes a null terminator), and 5 bytes for the dictionary header: a 2-byte pointer to the name string, a 2-byte pointer to the previous dictionary entry, and one final byte for the immediate and other flags.

In total the header consumes 12 bytes, for just 7 bytes of executable code. Not great.
Traditional Forths avoid the string terminating null byte by making use of a _counted string_ representation. Also, the _name-pointer_ and _flags-byte_ are often combined into a single _name-offset/flags_ byte. Thus making the overall header or our `square` example be just 9 bytes.
However, we would like to reclaim all the space used for the header.

The idea is to allocate the dictionary headers in a different part of the memory space, from `&3000` upwards -- where Mode-1 screen memory will reside -- leaving only the executable code in low memory. Now we just have to worry about keeping the executable code below `&3000`.
We are happy to give up the dictionary structure when the application starts. Before the application starts, we will remain in Mode-7, and as long as our dictionary headers keep below `&7c00`, we will be good.

The new scheme requires a small change to the Quarter Forth kernel primitives for dictionary management. Currently a dictionary entry is conflated with the execution token for the related compiled code, and the implementation relies on the dictionary header directly proceeding the compiled code. This will have to change.
Whether the new approach will reclaim enough space from the forth system to allow us to write a graphics application is an open question. We will see!


### Next time...

... I hope to write more about the BBC implementation.
Will it pan out as a good approach to developing graphics applications on the Beeb?
