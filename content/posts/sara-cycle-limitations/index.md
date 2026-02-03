+++
date = '2026-02-02T15:00:00Z'
draft = true
title = 'SARA: Cycle Limitations'

+++

One of the joys of consoles such as the Atari 2600, is the ability to extend the console's capabilities via the cartridge slot. Indeed, the Atari 2600 without additional hardware inserted into this slot is useless, for there is no way of loading CPU instructions without using it. 

Originally, additional hardware took the form of cartridges containing only ROM chips, but RAM chips were added soon enough. The _SARA_ chip was Atari's implementation of cartridge RAM for Atari published games. It adds an additional 128 bytes of RAM to the console.

However, there is one very important limitation of the SARA chip that prevents it from being used for executable code. While this limitation is well known in the Atari 2600 development community, the intention of this blog post is to explore why the SARA chip cannot be used in this way.

<style>
.my-table {
  border-collapse: collapse; /* ensures borders don’t double */
  width: 100%;               /* optional, makes table full-width */
}

.my-table th,
.my-table td {
  border: 1px solid #ccc;    /* light gray internal lines */
  padding: 0.5em;            /* spacing inside cells */
}

.my-table th {
  background-color: #f5f5f5; /* optional header shading */
}
</style>

## The Problem

## Addressing the Cartridge Bus

Like all cartridge hardware, the SARA chip is accessed through the cartridge bus. The cartridge bus consists of only twelve lines which is enough for 4096 addresses.

In the case of a cartridge containing both a ROM chip and the SARA chip, some of those addresses will be for accessing RAM and some for accessing ROM. The key detail however, is the lack of read/write line, which is important for accessing RAM. (The CPU in the 2600 does have a read/write line but it is not connected to the cartridge bus.)

For RAM to be addressed effectively therefor, there needs to be a way of _encoding_ the read/write signal in the address.

The method Atari settled on was to use two sets of addresses. The first set of 128 addresses is used for write operations and the second set of 128 addresses is used for read operations. This means that the total amount of memory is less than it might otherwise be, but losing 128 bytes of total memory is a good trade off for having some memory that you can write to.

The specific addresses chosen by Atari for SARA are shown in the table below. [^addressing]

<table class="my-table">
  <tr>
	<th>Signal</th>
    <th>Origin</th>
    <th>Memtop</th>
  </tr>
  <tr>
	<th>Read</th>
    <td><code>$080</code></td>
    <td><code>$0ff</code></td>
  </tr>
  <tr>
	<th>Write</th>
    <td><code>$000</code></td>
    <td><code>$07f</code></td>
  </tr>
</table>

If we look at those address numbers in binary form it becomes clear how the read/write signal is _encoded_ in the address. With a suitable bitmask we can extract the read/write signal from the address.

<table class="my-table">
  <tr>
	<th>Signal</th>
    <th>Origin</th>
    <th>Memtop</th>
  </tr>
  <tr>
	<th>Read</th>
    <td><code>0000 1000 0000</code></td>
    <td><code>0000 1111 1111</code></td>
  </tr>
  <tr>
	<th>Write</th>
    <td><code>0000 0000 0000</code></td>
    <td><code>0000 0111 1111</code></td>
  </tr>
</table>

Examining those binary forms allows us to deduce a suitable bitmask value of `$0f80`. The _Bitmask deduction_ section below contains an explanation for how this bitmask is derived. It's not necessary to understand the details but it's provided for those who are interested.

The key information to remember is that _there are two sets of addresses. One set for reading and a second set for writing._

{{< supplement >}}
#### Bitmask deduction

How is that bitmask value deduced? The key to the deduction is noticing how some bits in the table, are the same in all four values in the table; and conversely, that some bits are different in at least one value.

For the bits that are the same we simply set the equivalient bit in the bitmask. This gives us a partial bitmask of:

`1111 xxxx xxxx`

For the bits that are not the same in all four values, we need to think about why they are not the same.

If the bits change between rows (ie. between the read and write) then the corresponding bit in the mask is set to one.
 
If the bits change between the origin and memtop columns, then we leave those bits set to `0` in the mask.

This gives us a final bitmask, in binary, of:

`1111 1000 0000` 

Which in hexadecimal is the afore mentioned `$0f80`.

#### Extracting the read/write signal

To actually detect whether the SARA access is a read or write, we need to compare the result of the bitmask against a value.

Looking at the differences in the values in the table above, we can see that the first bit in the second nibble is set in the _read_ row and unset in the _write_ row.

This means that applying our bitmask to _any_ address will result in a value of `$0080` for an address containing the read signal. 

An address with the write signal meanwhile, will result in a value of `$0000` when the bitmask is applied.

```
switch address & $0f80
case $0000:
	write_SARA(address)
case $0080:
	read_SARA(address)
default:
	read_rom(address)
```

#### Extracting RAM address

Finally, the RAM address itself can be extracted by masking the address with `$007f`.

What is the meaning of this mask? Even though we use different addresses for reading and writing to SARA RAM, the underlying address for any given pair of read and write addresses, is the same.

{{< /supplement >}}

## 6507 Instructions for Reading and Writing

Reading and writing to SARA RAM is done in exactly the same way as accessing cartridge ROM. For example, loading a value from SARA RAM into the CPU might be done with the `LDA` instruction.

`LDA $1080`

In this form of the `LDA` instruction the CPU is using the _absolute_ addressing mode. In the case of the `LDA` instruction, this addressing mode requires four cycles.

Similarly, the indexed addressing mode requires four cycles.

`LDA $1080,X`



[^addressing]: When programming the 2600 we also need to indicate that the address is targeting the cartridge bus. This is done by setting the thirteenth most-significant bit. However, in the main text we are examining the address as viewed from the persepective of the cartridge bus device, which as we've stated, is only twelve bits wide.

[^supplement]: This supplementary section is really only of interest from the point-of-view of emulation. It's not directly indicative of what happens in the actual hardware. That is to say, the hardware doesn't apply masks in this literal way.
