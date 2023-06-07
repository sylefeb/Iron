# Iron: selectively turn RISC-V binaries into hardware

## TL;DR

Start from a compiled RISC-V binary, tell **Iron** which function symbols to turn into hardware. **Iron** outputs a SOC (in [Silice](https://github.com/sylefeb/silice)) including a RV32I CPU and hardware co-processors implementing the selected functions. The CPU triggers the co-processors when the selected functions are reached.

- [Read the abstract](publications/2023-summit-abstract.pdf) (2 pages)
- [Take a look at the poster](publications/2023-summit-poster.pdf)

(**Iron** was presented as a poster at the RISC-V European summit in 2023).

> **Where's the source code?** It's coming soon! I'm hard at work implementing an improved version of reReg, and cleaning things up. There are also changes in Silice to be pushed upstreamed, [stay tuned](https://twitter.com/sylefeb)!

## But why?

Well, first because it seemed too interesting and fun to pass on! A more rational reason is that **Iron** could be used to accelerate binaries *without having access to the original source code*, with the additional ability *to select what to accelerate depending on the available resources* (basically how big the target FPGA is).

## How fast does that get?

> Keep it mind **Iron** is a proof of concept: a first implementation to explore the feasibility of the idea. The prototype has many limitations.

Right now, **Iron** is focused on squashing cycles out, in order to form long combinational chains, reducing the cycle count. Think of it as forming large, highly specialized instructions out of the original binary code. The current version reaches an average of up to 7 instructions into a cycle on favorable cases (see table in [extended abstract](publications/2023-summit-abstract.pdf)). I think this should be more!

> I pushed things a bit by targeting RV32I with demos that use a lot of multiplications, so there are many calls to e.g. `__mulsi3`. These calls introduce additional cycles. I *could* (should?) target RV32IM instead, but see notes on [future work](#future-work) first. This remark gets even worse for all the demos using floating point arithmetics.

The objective of creating large combinational chains is to retrieve maximum flexibility for re-scheduling and pipelining (there is a preliminary version of that for special cases). Because, of course, large combinational chains reduce the cycle count (good) but also reduce the maximum frequency of the design (bad)!

## Future work

- It would be great to have **Iron** generate new opcodes using RISC-V extensions. Right now the CPU tracks special addresses corresponding to hardware function, which is not very elegant.
- When targeting C, some specialized functions like `__mulsi3` or floating point functions like `__adddf3` could be intercepted and replaced by an optimized hardware implementation (even going through a FP pipeline with a FIFO!).

Both of these points are why I target the simplest ISA (RV32I) as I feel a future version of **Iron** should be able to figure out RV32IM on its own, replacing all calls to `__mulsi3` by an new instruction implementing the multiplication.

- Beyond BRAM: currently **Iron** assumes everything happens in BRAM. A more elaborate memory model is required that could allow to wait while a cache gets filled. I also wonder how one could detect that a function is using a limited segment of memory. Could that be automatically mapped to BRAM?

## Challenges

Here's a list of fun ongoing challenges:

- `JALR`. When burning code into hardware, we need to know the possible destination of any jump. However, `JALR` happily jumps to any address stored in a register (plus immediate offset). So, supporting arbitrary `JALR` would mean having one state per-instruction ... making the whole thing useless as nothing can be collapsed into hardware. Yes, that is a problem. Fortunately, `gcc` at least does not use `JALR` beyond the typical function return ... *and* jump tables. These are currently *somehow* detected by **Iron** in a way that *seems* to work ok. I guess you can tell how confident I am in that feature ;)

- One pass **Iron** performs is *destack*. This intends to remove memory accesses that are only there to push/pop from the stack, replacing them with registers. While this works ok in many cases, there are cases where the stack is manipulated in ways that currently make *destack* fails. Worse, failure cases are not automatically detected. Whether this can be improved upon while still performing destack is an open question.

- **Iron** has been mainly tested on binaries compiled from `gcc`. Does this approach easily extend to other compilers? Are other compilers following the C calling convention? (Why would they?). This needs testing!

## That all sounds rather simple ...

Right? It seemed like an easy project. But well, there are *complications*. One thing that was difficult is the *reReg* pass, that attempts to undo the compiler register optimization (!!) to reduce multiplexer pressure. This is complicated by the fact that whatever logic we output should not contain any combinational cycle (so a register is not the same *before* and *after* being written in a given cycle).

Please see references in the [extended abstract](publications/2023-summit-abstract.pdf), [LegUp](https://web.archive.org/web/20230329233302/http://legup.eecg.utoronto.ca//) in particular, to get an idea of how deep this rabbit hole truly is :).
