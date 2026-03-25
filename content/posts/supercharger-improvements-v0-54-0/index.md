+++
date = '2026-03-25T12:00:00Z'
draft = false
title = 'Supercharger Improvements in v0.54.0'

+++

Gopher2600 has supported the Atari2600 _Supercharger_ for several years.
However, in recent weeks some longstanding flaws and some newly identified ones,
have been addressed.

The improvements relate to the way in which both tapes and binary _AR_ files are
loaded. The latter in particular being now being accessible to people without
access to the real _Supercharger_ BIOS.

<!--more-->

## The TAPE load register

The _TAPE_ load register is a reserved address that is used by the BIOS to
receive data from the tape input of the _Supercharger_ cartridge. The address of
this register is _$xff9_ in any of the cartridges mirrors.

In the real hardware, reading this address simply returns the data that is being
supplied by the tape at that moment. In our emulation however, we also use the
access of this register to control the automatic playing of the virtual audio
tape (a WAV or MP3 recording of a real _Supercharger_ tape).

That is, rather than require the user to start and stop the audio tape, the
emulator automatically begins to stream the audio data once the _TAPE_ register
has been accessed (frequently within a period of time).

Similarly, once access of the _TAPE_ register has ceased, the streaming of the
audio data is stopped.

We also use the _TAPE_ register to activate the _bootstrapping_ of data from a
binary _AR_ file. Prior to _v0.54.0_ the use of the _TAPE_ register to facilitate
bootstrapping has been problematic.

#### Phantom Access of _TAPE_

Using the _TAPE_ register for bootstrapping fails  when non-BIOS code accesses
the register. Accessing the register can be intentional or unintentional. 

An example of unintentional access can be seen in the game _Suicide Mission_.

In Bank 3 of _Suicide Mission_ at address _$1ff8_ there is an _RTS_ instruction.
This is a one byte instruction but because of how the _6507_ works, there will
be a _phantom_ read of the next address. The next address is of course
the _TAPE_ register.

This isn't a problem for the real hardware, but for our emulation it causes
erroneous bootstrapping of _AR_ files. In the case of _Suicide Mission_ and
likely other games loaded from an _AR_ file, the game would immediately crash
with a _CPU KIL_ error.

As of _v0.54.0_ the emulation handles phantom accesses by ignoring _TAPE_
accesses that follow a specific pattern. The pattern is very simple: handling of
auto-loads and bootstraps only occurs when the access was not preceded by an
access of _$1ff8_ or _$1ff9_.

Loading from a virtual audio tape worked but it did mean that silently in the
background, the tape was always "playing" in the background. While it didn't
create a problem, that behaviour has been corrected too.

#### Intentional non-BIOS Access of _TAPE_

My original fix for the phantom access problem was to do nothing (ie. no
auto-load or bootstrapping) unless the BIOS was active. That's a good solution
except it prevents intentional access by non-BIOS code.

And as it happens, I came across [an example of such a ROM](https://forums.atariage.com/topic/389048-morse-monitor-decoder-pal-50hz-starpath-supercharger) while I was working on
the problem. This ROM helped me to better understand the problem and to arrive at
the superior solution described above.

A video of this ROM, a Morse code decoder, is shown below.

{{< youtube C_ihVYEGDpw >}}

Currently, Gopher2600 requires the audio to be concatenated into one file. Future
versions may support the ability to specify more than one file, or even reading
from the computer's microphone port.

The WAV file used in the video above can be download [here](morsecode_decoder_example.wav)

## Fake Supercharger BIOS

As of _v0.54.0_ it is possible to load _AR_ binary files without a real
Supercharger BIOS.

Previously, a file containing the real BIOS was required. It was to be provided
by the user and stored in Gopher2600's configuration directory [(full details in
the project wiki)](https://github.com/JetSetIlly/Gopher2600-Docs/wiki/Supercharger)

A real BIOS is still required for loading from a virtual tape but it should
never have been a requirement for binary files. Loading _AR_ binary files now
uses a built-in fake BIOS.

## Future Changes

Although Gopher2600's emulation of the Supercharger device is very advanced and
arguably the best available from any current emulator, there's still room for
improvement.

The next enhancement will likely be better MP3 decoding. The code for decoding
MP3s is one of the few areas of Gopher2600 where I uses a third-party module. As
it happens, the module I chose has flaws and cannot cope with all MP3 encodings
even when the quality is seemingly sufficient.

It's likely that I will still use a third-party solution but I've not yet
settled on one just yet.

Another improvement, mentioned in passing already, is the possibility of loading
audio from the computer's microphone port. This would be a nice feature but I'm
reluctant to add code that gathers data from the user's environment because the
intention of the code could easily be misinterpreted.

A better option might be to allow loading of data from a named pipe (if
available via the operating system), leaving the user to plumb the pipe as the
see fit. This requires more thinking.



