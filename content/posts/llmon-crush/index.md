+++
date = '2026-09-25T12:00:00Z'
draft = false
title = 'LLMon Crush'
+++

I've recently conducted experiments with a coding agent working on the
_Gopher2600_ code. I did not use one of those agents that require a subscription
(I've never used one of those). I used instead an agent that uses an LLM running
on my own hardware. 

This definitely **should not** be taken as a signal that I am embracing agentic
coding. I am only exploring the possibilities and trying to understand how
these tools might benefit me.

<!--more-->

Moreover, given the how quickly agentic coding has found a footing in software
engineering, it would be strange if I never mentioned it at all. This article
should be seen as an admission that I have considered how these tools might help
me and the _Gopher2600_ project.

I conclude that a coding agent running entirely on local hardware has real
potential for helping solve certain problems.

## Background Reading

Before continuing, I should emphasise that my opinions on coding agents and LLMs
are complex and evolving. One thing I am sure about however, is that we should
aim to own and control the software we use. That is to say, I think coding
agents and LLMs should be run on hardware we own. Renting time from companies
with dubious motivations is not an acceptable solution to me.

Recently read books that have informed and clarified my thinking about this are:

* _Empire of AI_, by Karen Hao
* _The Nerd Reich_, by Gil Duran
* _What The Dormouse Said_, by John Markoff

I recommend all books to people interested in the subject.

The last book in the list is relatively old but I found it especially relevant.
_What The Dormouse Said_ is a story of the personal computing industry and how
early research into personal computing conflicted directly with the academic
thinking of the time.

Whereas some people wanted computers to be owned by individuals, other
researchers did not see the need and saw the future in timeshare computing. The
fact that many in the latter group were artificial intelligence researchers I
find to be ironic.

## Hardware and Software

The pertinent hardware for this experiment is _12GB RTX 5070_. This isn't really
enough memory for this type of task but it's large enough to hold a small model
entirely in VRAM.

For agent work, a large context would quickly become overwhelming so keeping the
context small is crucial if we want to keep the LLM running quickly.[^gaming]

The LLM is running on a locally compiled _llama.cpp_ with _CUDA_ enabled. The agent
is _Crush_ from 

_Crush_ isn't a well know agent but it caught my attention because it was far
easier to compile and setup, for me. I've noticed that coding agents tend to be
written in Python. Nothing wrong with Python except that I find it painful to
set up the dependencies for a Python application. Modern tooling like _uv_
helps, but I still find myself avoiding other people's Python programs. [^python]

### The LLM Model

The LLM model for this test is _Qwen3.8_, but not the original release from
_Qwen_, which would far too large for the hardware. Instead, I've used a nine
billion parameter distillation quantised to 8 bit, released by [_empero-ai_
](https://huggingface.co/empero-ai/Qwen3.8-9B-Distill-GGUF).

## The Thumb-2 BFI Instruction

The problem we are working on is an instruction in the _Thumb-2_ instruction
set. _Thumb-2_ was initially added to _Gopher2600_ to support _ACE_ cartridges,

The 32bit subset of instructions in particular, has been a source of the
occasional bug since they were first added to the emulator.

The most recent bug was in the _BFI_ instruction identified by Wolfgang Stubig
using the _Claude_ agent. This is what that agent deduced when analysing the
behaviour of Stubig's _TinyELF Basic_.

    Gopher2600's Thumb-2 BFI instruction gives wrong results. In TinyELF Basic,
    BASIC line numbers above 255 came out wrong (300 became 44, 255 became 254),
    so GOSUB 900 failed. Real hardware is not affected.

    The cause is two wrong bit masks in thumb2_32bit.go. A two-line fix makes
    the line numbers come out right in Gopher. I have not submitted a pull
    request yet.

This is what the _ARM Architecture Reference Manual Thumb-2 Supplement_ says
about BFI:

    Bit Field Insert copies any number of low order bits from a register into
    the same number of adjacent bits at any position in the destination register.

The pseudo-code given in the reference is:

    if ConditionPassed() then
        EncodingSpecificOperations();
        if msbit >= lsbit then
            R[d]<msbit:lsbit> = R[n]<(msbit-lsbit):0>;
            // Other bits of R[d] are unchanged
        else
            UNPREDICTABLE;

This pseudo-code is typical of the the reference manual. My mistake in the
implementation was to misinterpret the intent of the instruction. 

Incidentally, I don't believe a testing strategy would have helped me here. Any
tests that I wrote myself would only be testing whether my implementation was
correct. On the other hand, a standard ARM test suite, official or unofficial,
would be of help, but I am not aware of any that are freely available.[^tdd]

### Prompting Crush/Qwen

So can my modest domestic hardware identify what the issue with the BFI
instruction is?

I'm not sure of the sequence of events that led up to Claude's statement. Maybe
it analysed the problem beginning only with the affected binary, but for this
experiment I'll start with the knowledge that the BFI instruction is where the
problem lies.

The prompt:

    I believe the BFI instruction in the Thumb-2 instruction set contains a bug.
    Find the implementation for this instruction in the source code and analyse it.

After less than a minute, Crush/Qwen correctly identifies the issue.

    I found the bug. Let me show you the analysis:

    ## The Bug in BFI

      maskInsert := uint32(((1 << width) - 1) << 1)

    The `<< 1` is incorrect. It should simply be:

      maskInsert := uint32((1 << width) - 1)

This is a correct analysis. My mistake was to interpret the directive,

    R[n]<(msbit-lsbit):0>

as a instruction to ignore the least significant bit and for bits we are
extracting to start from position one. A nonsense interpretation.

The response then gives more detail.

    ### Why it's wrong

    According to the ARM Thumb-2 spec (AArch64 ARM ARM v8.0, section C3.4.6.14), BFI is defined as:

      Rd ← (Rd & ~((1<<width) - 1) << lsbit) | ((Rn & ((1<<width) - 1)) << lsbit)

    The `maskInsert` is supposed to extract the top `width` bits of the source register `Rn`. The correct mask is `(1 << width) - 1`, which produces a mask of `width` ones at the most significant bit position.

    The erroneous `<< 1` shifts this mask left by one, which:

    • Changes the number of bits extracted (off by one)
    • Offsets the bit positions incorrectly

The specific ARM reference mentioned in this part of the responses is not the
reference that I've noted in the comments and documentation. 

In this case the difference in reference doesn't matter because the _BFI_
instruction will be the same across all relevant variants of the ARM. But it
might make a difference in other instances. It seems that the LLM is drawing
its conclusions from the training data rather than the context provided.

Continuing with the response, the agent gives an example of correct and
incorrect behaviour. It also, surprisingly, creates and applies a patch. It's
surprising because I didn't ask for the code to be patched. The initial prompt
only asked to identify the bug and to analyse it.

After applying the patch, the agent builds and runs the project tests to make
sure nothing has broken. If we ignore from the problem that the agent acted
_ultra vires_ and created the patch, I think building and running the tests is
good.

To emphasise, it would be better however if the building and running only
happened if we asked the agent to create the patch. Building and running has no
place in a simple analysis.

The agent concludes with a summary, which is effectively a repeat of the
initial response. The whole process took about two minutes. 

## Conclusions

On this specific task, I feel Crush/Qwen did quite well. Given very little
guidance in the prompt, the agent found the correct file in the source and the
correct function. It also successfully compared my implementation with the
pseudo-code in the official reference.

I think this shows that problems that involve comparing the logic written in one
language with an interpretation in another language, a locally run agent/LLM is
an entirely plausible solution. I will consider using this system in the future.

All that said, there is evidence that suggests that agentic coding should be
handled with kid gloves. On first running the agent prompted me to "initialise"
the project. The process was relatively quickly and produced the now ubiquitous
_AGENTS.md_.

On inspection of the agents file, it revealed some bad misunderstandings and
interpretations of the project. For example, the agent suggests that running
with the _--rewind_ option enables rewinding. There is no such option. The
ability to rewind is always enabled.

## Further Experiments

After achieving successful results with the _BFI_ instruction, I decided to give
the agent a more difficult task. I asked it to look at all the implementations of
the 32bit Thumb-2 instructions. It was less successful at this.

It identified a problem in the _ADDW_ and _SUBW_ instructions, claiming that the
implementation did not set the status flags according the results of the
add/subtract operations.

What the agent didn't notice is that the specific implementations it identified
were both dealing with the _T4_ encoding of the instruction. The _T4_ encoding
explicitly does not set the status flags (section _4.6.3_ of the _Thumb-2 Supplement_).

What was worse however, was the refusal of the agent to accept that it was
dealing with the _T4_ encoding, even after several exchanges.

There were other issues but this was the most glaring. Equally, there were some
observations in the broader analysis that are worth following up on.

## Final Thoughts

I would feel comfortable using a local agent, whether it be Crush/Qwen or
something else, to help with debugging logic problems. Giving ourselves an
alternative view of a problem is something that we often need as problem
solvers. When used in this way, I see coding agents as no different to a
traditional debugger.

It's important though, even when using debuggers or coding agents, that we
continue to actively engage with the problem and to make sure we understand the
solutions.

I don't see myself ever wanting to use them for the production of code. I enjoy
the act of writing code and indeed the act of typing is where I do my thinking. I
don't believe I would be as effective at programming in any other way.

[^gaming]: If I thought I was going to be running LLMs I would have paid extra for a card
with more memory, but I bought it for gaming and 12GB is plenty for that.

[^python]: There's an irony here because my original experimental version of a 2600 emulator was written in Python. It was called _HeadlessVCS_ at that time
to reflect its intended usage. I'll write about _HeadlessVCS_ in a future
article.

[^tdd]: Compare to the _6502_ where there are excellent test suites that can be
used to test the accuracy of an emulation. In the case of _Gopher2600's_ 6502
emulation, I use [Klaus Dormann's set of functional tests](https://github.com/Klaus2m5/6502_65C02_functional_tests).
