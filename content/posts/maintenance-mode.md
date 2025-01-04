+++
date = '2024-07-19T00:00:00Z'
draft = false
title = 'Maintenance Mode'
+++

_This article originally appeared in the "Current Status" page of the Gopher2600 Wiki_

The current status of Gopher2600 is maintenance. I'm happy to fix specific bugs people may have but I won't be adding anything substantial. This may change in years to come but I have other projects that I want to focus on. Too many years have been wasted on this and I need to look at something else.

<!--more-->

### Self Reflection

Gopher2600 started out as a way of exploring the Atari2600. I had written a small 4k game for console and enjoyed doing so. What I didn't enjoy was cycle-counting. So Gopher2600 was originally meant to be a cycle-counter. It slowly grew from that into a complete emulator. That wasn't my intention when I started but each step led me to the inevitable result.

During early development I tried my best to keep the project well engineered. I was learning Go at the same time (this is my first Go project) so some early development choices are now incorrect, and maybe even incorrect at the time. As such there are areas of the code that undoubtedly require refactoring. Moreover, I began to add features requested by users. This has meant that the emulator has gained a genuine utility but to do so I cut corners during development. Therefore, the way different parts of the project communicate with one another is not always uniform in the way you might expect. If I ever return to the project then a refactoring would be a good start.

From a Go perspective, I think the language is fine for this sort of development. I'm unsure whether the language has had an impact on the performance, which is slow when compared to Stella for example. Profiling suggests that my method of TIA emulation is the bottleneck. However, my method also means the emulation can be paused and inspected during a single CPU cycle. This was very useful during development of the emulator itself but whether it's useful to the general 2600 developer is debatable.

I'm particularly pleased that I've kept the bill of materials to a minimum. The very large requirements like SDL and OpenGL are only used by the GUI version of the emulator. Although I've not made any effort to package the components separately, it is possible to use the emulator core in other contexts. The https://github.com/JetSetIlly/Gopher2600-Utils/ repository contains a couple of (slightly out-of-date) examples including a sketch that uses the Ebiten system.

On a personal level this project was a departure from my normal way of working. I have never really released anything before this and certainly never accepted development input from anyone else. On the whole, interacting with other people has been a positive experience and meant that the project has grown far beyond the original scope. Although imperfect I am happy with the current state. It's not good enough for the v1.0 tag but it's good for me :-)
