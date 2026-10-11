+++
date = '2026-10-11T09:00:00Z'
draft = false
title = 'Efficiently Recording Game State'
+++

A recent feature proposal for Stella has made me think once again about Gopher2600 playback files and the underlying ideas of compression and time. This blog entry very briefly explains how Gopher2600 currently records gameplay for future playback. It also briefly touches on the closely related topic of the rewind system.

<!--more-->

### Raw Data

The [Stella feature proposal](https://github.com/stella-emu/stella/issues/1185) adds a new feature to Stella that allows the emulator to record a stream of raw emulator data to a file (or a pipe). The idea is that this data can then be used by other tools for further processing:

The first suggestion in the proposal is for the data to be used to create "frame-perfect" video recordings. The difficulty of making "frame-perfect" videos is a good observation by the proposer and Gopher2600 offers a `-video` option for this reason. Video recording, although related to the proposal, is a quite separate topic and we won't mention it again.

Other suggested uses for a raw data stream include, "watching RAM by symbol every frame" and "checking scanline counts". These are also great ideas but the amount of data that must be created (many megabytes for just a few seconds) is enormous. A simple tool that checks scanline counts, for example, requires only a small fraction of that data.

### Compressed Data

Naturally, when presented with large quantities of data we must start to think about data compression. The raw data created by the feature in the Stella proposal will compress very well (with zip for example). However, there is a more fundamental quality to emulation that can be exploited.

My observation is that the 2600 emulator (or indeed any emulator) can be thought of as a _decompressor_. In this view, the ROM file is _decompressed_ by the emulator to reach the game state we need.

Thinking of an emulator in this way works because the emulator (and the machine being emulated) is _deterministic_. That is, given a starting point, the emulator will always arrive at the same state over time.

### User Input

The view that the emulator is entirely deterministic is a little simplistic, however. A game would be uninteresting if it produced exactly the same results each time it is played. A game only becomes interesting only when it responds to user input.[^complications]

What we need then is a way of recording and playing back user input. These `playback` files can then form part of the input to the emulator (or _decompressor_ in our analogy) along with the ROM file.

### The Playback File

The ability to record and playback user input was added very early in the development of Gopher2600. It is one of the central pillars of the emulator's testing regime. 

The playback file is written in plain ASCII. An example is shown below, followed by a brief description of the format:

    gopher2600playback
    1.2
    /home/steve/gopher2600/roms/Pitfall.bin
    8d525480445d48cc48460dc666ebad78c8fb7b73
    AUTO
    Left, Right, true, 23, 1, -65, 98b6ccedb1876d3e156ed489ad3fcb93b18573aa
    Left, Fire, true, 43, 3, -53, 96e78618dda22dae327831f1e8faccb51aa6ed3e
    Left, Fire, false, 51, 0, 73, 71a5ee9a5912e13c4e2ae862345a7d51a88db0b6
    Left, Fire, true, 81, 3, -26, 15b9c07e9445ea997718fe9dc4064559bcb9c08a
    Left, Fire, false, 93, 2, 103, 3d72d8cbc0f4c564b76a91ee067f5db8fa659735
    Left, Right, false, 107, 0, -44, ca26a8e97591aa1075955a5ce8dc1639826da1c4
    Panel, PanelPowerOff, , 145, 0, 94, 68da0a8774afa0f12800e4cdb814ad355f05ec06

1. The _first_ line identifies the file type and the _second_ line the version number of the playback file. 
2. The _third_ line is the path to the ROM file that was recorded and the _fourth_ line is the MD5 hash of that file.
3. The _fifth_ line specifies the television type at the time of recording.

#### Input Events

While the first five lines form the _header_ of the playback file, the remaining lines record the actual user input _events_. Each line is split into fields, separated by a comma.

The first field specifies which _port_ the record refers to. This can be _left_ or _right_ to indicate the player port or _panel_ to indicate the switches and buttons on the console itself.

The second field specifies the _event_ and the third field is the _event data_. The possible values for these fields depend on what the peripheral attached to that port. In most cases, the peripheral in the _left_ and _right_ ports will be a joystick and the _event_ will be one of the cardinal directions (or fire) and _event data_ will be true or false.

In the example above, we can deduce that the player moved the joystick right and pressed fire twice before powering the console off. The final record in the playback file shows the power off event.

#### Coordinates

The next three numeric fields specify the coordinates of the console at the moment when user input occurred. In the example, the player first moves the joystick exactly on _frame 23_, _scanline 1_, _clock position -65_. During playback the user input is inserted into the emulation at exactly that moment. [^time]

The final field is the MD5 hash of the screen at the time of the input event. It is used to make sure that the emulation is proceeding as expected.

The _hash_ field is very important for the emulator's regression tests. In the case of _playback_ regression tests, if any of these MD5 hashes do not match during playback, the test has failed and the emulation is considered temporarily broken.

### Rewinding

We can think of the each record in the user input stream as a _key state_. The concept is similar to that of a _key frame_ in MPEG video encoding. States in between the key states can be inferred and do not need to be recorded. 

This concept of _key states_ also appears in Gopher2600's rewind system. Stepping backwards in time requires knowledge of a previous state. So if we want to step back a single scanline does that mean we need to record the machine state at every scanline? That would require an enormous amount of memory and more so if we want the stepping granularity to be finer than the scanline (which Gopher2600 does).

Gopher2600 solves this problem by only recording state at key moments and to then recreate the state at a specific television coordinate by rerunning the emulation from the nearest key state. For example, to move back one scanline, the nearest previous state is restored and the emulation is stepped forward to the correct point.

By default, the rewind system records state at the beginning of each frame, but this is for speed as much as anything else. Key states could potentially by taken only ever two or even ten frames. At the most extreme, the rewind system could work (albeit more slowly) if it only recorded state at the moment of each user input.  

## Conclusion

The proposal for Stella to record _raw data_ is a good one but it produces output files that are very large. A better solution is to use the emulator more interactively, meaning that any output file from a recording can be very small. 

The `HEADLESS` mode is the interactive method Gopher2600 offers instead. In this mode, a third-party tool (a coding agent, for example) can ask the emulator to generate only the information it needs and only when it needs it.

Moreover, the discussion about playback files and key states, shows that recording of a session does not need to consume a lot of data at all. If the goal is to record a session for future analysis, recording user input is all that is required.

We can combine the playback of a recording with continuation of the emulation in HEADLESS mode. We can even step back using the rewind system as required. 

In short: a good rewind system and an interactive interface are more flexible than a monolithic file of raw data.

### Further Work on Gopher2600

Finally, while thinking about the contents of this blog, I'm reminded of the need for improvement in Gopher2600's playback file and also the rewind system. The rewind system would benefit from a smarter use of key states and the playback file format needs more information in the header. 

[^complications]: There are other indeterminate inputs to the emulated machine that are worth mentioning, very briefly. Firstly, there will be randomness in the form of heat or radio interference. Also, network communication is indeterminate by nature, although this only affects PlusCart emulation currently. Mitigation of these complications is beyond the scope of the article.

[^time]: Coordinates are the way in which Gopher2600 measures and specifies time. Coordinates refer literally to the position of the television beam at any given moment. The concept works because of the close association of the program running on the 6507 and the generation of the television image.
